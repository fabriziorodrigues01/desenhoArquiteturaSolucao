# Arquitetura de Solução

## **Gestão de limites e autorização em canais digitais**

Março 2026

Version: 0.1

Histórico do Documento

Revisão do Documento

| **Revisão** | **Data** | **Criado** | **Descrição resumida das mudanças** |
| --- | --- | --- | --- |
| 0.1 | 01/03/2026 | Fabrizio Rodrigues | Initial Version |

Conteúdo

1\. Descrição da Solução 4

1.1 Objetivos e regras de negócio 4

1.2 Escopo geral do projeto 4

2\. Cenário de negócio 5

2.1 Visão estado atual - AS-IS 5

2.2 Oportunidades e Soluções - TO-BE 5

2.2.1 Arquitetura Lógica 5

2.2.2 Arquitetura de infraestrutura 10

3\. Anexos 14

# Descrição da Solução

## Objetivos e regras de negócio

Prover a **gestão de limites** e **autorização** de cartão de crédito, devendo ficar disponível de forma **consistente em todos os canais** (App, web e central de atendimento).

## Escopo geral do projeto

- O limite deve refletir **em tempo real** para garantir que transações subsequentes sejam autorizadas com base no valor atualizado.
- A autorização de transações deve considerar o limite mais atualizado possível, mesmo que a origem seja offline.
- O sistema atual de cartões é legado e possui uma **interface síncrona com latência média de 800ms** e disponibilidade limitada fora do horário comercial.
- Estima-se um tráfego de **1 milhão de requisições por dia**, podendo chegar a **10 milhões em datas promocionais** (_ex.: black Friday_).
- Os canais devem permanecer disponíveis e responsivos, mesmo em momentos de falha de conectividade com o sistema legado.

# Cenário de negócio

## Visão estado atual - AS-IS

Atualmente o cliente pode consultar e ajustar seu limite do cartão de crédito através dos canais (App, Web e Canal de atendimento), porém a atualização não ocorre em **tempo real.**

## Oportunidades e Soluções - TO-BE

## **Arquitetura Lógica**

### Realizado o (Gap Analise) dos sistemas externos:

Identificado a ausência destes elementos no escopo original e que são vitais para a conclusividade da experiência do Cliente:

• Bandeira (Visa/Mastercard): Autorizar o limite nas maquininhas para transações offline. Elas precisam receber a atualização do limite (API) para autorizar transações na ponta.

• Motor de Risco/Fraude: Alterar limite não é apenas uma troca de valor; exige uma validação de risco em tempo real.

• Sistema de Notificações (Push/SMS): Para confirmar a alteração do limite ao cliente (segurança).

• Bacen/Orgão Regulatório: Logs de auditoria para conformidade com o Banco Central em relação a fiscalização das operações de cartão de crédito.

![Diagrama BPMN](./imagens/fluxo%20de%20negocio.png)

O Cliente ao efetuar o ajuste de seu limite através dos canais digitais, o serviço responsável pelas regras de negócio (caixa Gestão de Limite) efetua as validações e atualiza o **Redis** e o **DynamoDB** antes de responder ao cliente, garantindo que a próxima transação (mesmo que ocorra 1ms depois) já leia o limite novo do Redis e acima de tudo realize a **Idempotência,** efetuando a verificação da chave, tratando o risco de duplicidades de ajustes de limite para mesma transação.

O **Motor de Risco** inserido como a primeira validação crítica em relação ao _Score_ e a _Política de Crédito_. No desenho, ele é acessado via API antes de qualquer escrita em banco.

O **Sistema de Notificações** tem um papel essencial para o "fechamento" da jornada do cliente pois ele recebe o push assim que o Redis/Dynamo são atualizados, reforçando a percepção de tempo real.

O **Sistema de Bandeira** precisa ser notificada do novo limite para que, se o cliente passar o cartão em uma maquininha offline, a Bandeira saiba que ele tem saldo. A atualização deve ser feita de forma online e offline.

O uso da mensageria (caixa SQS) tem um papel importante para o isolamento de falha, pois se o "Sistema Legado" falhar e o cliente já ter recebido a notificação de OK, o processo não será interrompido. A idéia é que o cliente receba a resposta muito antes do Legado ou da Bandeira serem atualizados garantidos por um back-end resiliente.

### **_Fluxo de dados (Transação completa com padrão de resiliência)_**

O fluxo abaixo mantém a experiência fluida em canais digitais, mesmo com um legado síncrono e com latência de 800ms.

- **1 Request Omnichannel:** O cliente solicita o ajuste via App/Web. A requisição carrega uma chave Idempotente única gerada pelo dispositivo.

- **2 Camada Edge:** O tráfego passa pelo **API Gateway**.

- **3 Check de Idempotência (Redis):** O Gestor de limite (**EKS**) consulta o **Amazon ElastiCache (Redis)**. Se a chave já existir, o sistema retorna o status da transação anterior em menos tempo, evitando processamento duplicado quando houver picos de acessos por exemplo na black Friday.

- **4 Motor de Risco (SaaS):** O serviço realiza uma chamada síncrona ao **Motor de Risco (SaaS Externo)**. Se negado, o fluxo encerra aqui com um erro de negócio.

- **6 Persistência (DynamoDB):** O limite é gravado no **Amazon DynamoDB** com o status PENDING_SYNC (para aguardar os bloqueios serem liberados se outra transação modificar o mesmo item). O banco escala instantaneamente para suportar a carga da Black Friday em um trafego imprevisível no modo (**On-Demand**).

- **7 Atualização do Estado Projetado (Redis):** Simultaneamente, o **Redis** é atualizado com o novo valor de limite.

- **8 Confirmação Otimista (Push 1):** O sistema dispara uma notificação via **Amazon SNS/Push** informando que o pedido foi recebido e o limite já está disponível para uso.

- **10 Resposta ao Cliente:** O App recebe o "OK" em **< XXms**, cumprindo o requisito de responsividade.

- **11**O **DynamoDB Streams** captura a alteração e aciona uma **Lambda (Integration Worker)** de forma assíncrona.

- **12 Buffer de Resiliência (SQS FIFO):** A Lambda posta o evento em uma fila **Amazon SQS FIFO**. Isso garante a ordem cronológica (essencial para ajustes de limite) e atua como um amortecedor contra a latência de 800ms do legado.

- **13 Integração com Legado (ACL):** O worker de integração tenta atualizar o **Legado** via **Direct Connect**.

- **15 Cenário de Sucesso:** O Legado confirma em 800ms. O Worker atualiza o DynamoDB para status COMPLETED

- **16** Uma trigger aciona **Lambda** que dispara o **Push 2** ("Limite confirmado no sistema central").

- **17 e 18 Cenário de Falha Temporária (Resiliência):** Se o Legado estiver offline (fora do horário comercial), a mensagem permanece no SQS. O sistema tentará novamente de forma automática sem intervenção humana.

- **19 e 20 Cenário de Erro Fatal (DLQ):** Se após X tentativas o erro persistir, a mensagem é movida para a **Dead Letter Queue (DLQ)**. Um alerta é gerado para o time de operações, mas o cliente **não teve sua experiência interrompida** no canal digital.

![Diagrama Dados](./imagens/fluxo%20de%20dados%202.png)

### **_Justificativa Técnica_**

A solução proposta baseia-se no desacoplamento total entre a Experiência do Cliente (tempo real) e o Registro no Legado.

**Performance e Escalabilidade (Pilar de Eficiência)**: Utilização do Amazon EKS com HPA (Horizontal Pod Autoscaler) e DynamoDB On-Demand para suportar a elasticidade necessária (de 1M para 10M de requisições). O gargalo do sistema legado (800ms) é isolado por uma camada de mensageria (SQS FIFO), garantindo que a latência percebida pelo usuário seja ditada apenas pela infraestrutura AWS (~30ms a 45ms).

**Consistência e Confiabilidade**: O uso do Redis (ElastiCache) como State Store resolve o problema de idempotência e concorrência em alta escala. Ao atualizar o Redis e o DynamoDB antes de responder ao cliente, é garantido que qualquer transação subsequente (autorização) consulte o saldo mais atualizado, cumprindo o requisito de "tempo real".

**Resiliência (Pilar de Confiabilidade)**: Caso o legado esteja offline ou instável, as operações de ajuste de limite não são perdidas; elas permanecem persistidas na fila com lógica de retentativa automática e DLQ, garantindo 99,99% de disponibilidade para os canais digitais.

### **_Decisão de Design Adotado_**

Consistência Eventual (Legado) versus Consistência Forte (Canais)

     Decisão: Será permitido que o sistema legado esteja temporariamente desatualizado em relação à AWS por alguns segundos.

     Justificativa: Priorizar a Disponibilidade e a Performance. Manter uma transação síncrona com o legado de 800ms degradaria a experiência do cliente e aumentaria o risco de timeout nos canais digitais durante picos de tráfego.

Complexidade de Implementação versus Resiliência

     Decisão: O uso de mensageria (SQS), Streams e Cache aumenta a complexidade do ecossistema e exige maior esforço de monitoramento (Distributed Tracing).

     Justificativa: É um custo necessário para garantir que o sistema digital não sofra "efeito cascata" em caso de indisponibilidade do legado.

Custo Operacional versus Escalabilidade (Serverless/Managed)

     Decisão: O uso de serviços gerenciados como DynamoDB On-Demand e ElastiCache pode ter um custo unitário superior a instâncias fixas.

     Justificativa: A capacidade de escalar de 1M para 10M de requisições sem intervenção manual (Ops) justificam o investimento, especialmente para eventos críticos como a Black Friday, onde o custo da queda do sistema supera o custo da infraestrutura.

## **Arquitetura de infraestrutura**

### **_Pilha de Serviços_**

A infraestrutura será baseada em uma topologia Híbrida e Multi-AZ na AWS.

Os componentes para compor esta arquitetura são definidos em Camada de Entrada e Segurança, Camada de Conteinerização, Camada de Integração e a Camada de Processamento, conforme os recursos abaixo:

- **Amazon API Gateway:** Porta de entrada para os canais. Ele recebe a requisição REST/JSON, faz o throttling (controle de vazão) e gerencia o versionamento da API.
- **AWS Lambda:** Terá o papel de workers de integração leves e orientados a eventos.
- **AWS EKS microserviço:** Atua como orquestrador central para os microserviços de negócio, garantindo alta disponibilidade e portabilidade.
- **AWS Direct Connect ou VPN Site to Site:** Túnel dedicado e seguro que interliga a VPC da AWS ao Data Center onde o Legado está hospedado.
- **Amazon DynamoDB:** A camada de dados será composta por bancos de dados gerenciados (NoSQL e In-memory) para suportar a baixa latência exigida. Sugestao de uso com o modelo Global Tables para DR e Point-in-Time Recovery para segurança de dados onde os backups podem ser replicados automaticamente entre Regiões para maior resiliência e com recuperação pontual.

        Observação para o caso em que a região principal falhe:

        RPO (Recovery Point Objective): Próximo de zero. Como a escrita no DynamoDB acontece antes de qualquer tentativa de sincronização, o dado nunca é perdido, mesmo que o Legado falhe.

        RTO (Recovery Time Objective): Segundos. Com Global Tables, a infraestrutura chaveia para outra região AWS se houver um desastre regional.

- Cache & Idempotência: **Amazon ElastiCache for Redis** (Latência < 1ms para travas de idempotência e saldo projetado).  

- Amazon SQS FIFO (Garantia de ordem e buffer de resiliência): será para isolar falhas no **Direct Connect.**
- **AWS CodeCommit:** Repositório privado Git onde o código é armazenado
- **AWS CodeBuild:** Acionado pelo Pipeline para:
    • Compilar a Aplicação.
    • Executar testes unitários e de integração.
    • Gerar a imagem Docker.
    • Fazer o Security Scan da imagem.

- **AWS ECR**: O CodeBuild faz o push da imagem versionada para o registro de containers.
- **AWS CodeDeploy / Helm**: O pipeline atualiza o manifesto do Kubernetes no EKS para realizar o rolling update (substituindo os Pods sem interromper o serviço).
- **IAM Roles for Service Accounts**: O Pod no EKS não usará chaves fixas (Access Keys) para acessar o DB. Ele usará uma Role temporária do IAM, seguindo o princípio do menor privilégio.
- **AWS Cloudformation:** serviço nativo para Infraestrutura como Código (IaC)

![Arquitetura Cloud](./imagens/diagrama%20arquitetura%20-%20draft.png)

### **_Estratégias para Consistência e Atualização em Tempo Real_**

Armazenamento de estado e cache: O ajuste de limite é gravado primeiro no DynamoDB (Persistência) e imediatamente no Redis (Estado Projetado).

Consistência Forte no Canal: O Gestor de Limites consulta o Redis, garantindo que o cliente possa utilizar o novo limite em milissegundos após a confirmação nos canais.

Consistência Eventual no Legado: A sincronização com o Legado ocorre de forma assíncrona, garantindo que a lentidão do legado não bloqueie a experiência do usuário

### **_Estratégia de Deployment_**

Containers (EKS): Usado no serviço principal de Limites para termos controle total do sistema e ele crescer conforme o uso.

Serverless (Lambda): Processa os dados da fila SQS de forma automática e barata, já que só funciona quando tem tarefa pendente.

Estratégia de atualização: Usaremos as estratégias Canary ou Blue/Green. Isso permite desfazer as mudanças na hora se a conexão com os sistemas antigos der problema


### **_Segurança, Escalabilidade, Monitoramento e Resiliência_**

Segurança: AWS WAF atuará como um escudo para bloquear ataques e invasões vindos da internet.

Escalabilidade: HPA e Cluster Autoscaler: O sistema cria novas cópias de si mesmo e contrata mais "espaço" em servidores automaticamente quando o número de acessos sobe.
Banco de Dados On-Demand: O banco de dados cresce e diminui sozinho, e só pagamos pelo que for realmente usado.

Monitoramento: Amazon CloudWatch (funciona como um painel de controle que nos avisa (com alertas) se algo sair do normal), AWS X-Ray (acompanha o "caminho" completo de um pedido, desde o celular do cliente até os sistemas alvo, ajudando a achar onde está a lentidão).

Resiliência: Uso de Dead Letter Queues (DLQ) pois se um processo der erro, ele é colocado em uma "fila de espera" separada para não travar o resto do sistema.


### **_Comunicação com o Sistema Legado_**

Conectividade: AWS Direct Connect (DX) como link principal de baixa latência.

Integração: Implementação de uma Anti-Corruption Layer (ACL) dentro do Worker de integração, responsável por traduzir JSON/REST para os formatos proprietários do legado (ex: Fixed Length/Copybook/DFDL/XML) e aplicar padrões de Circuit Breaker.

    Explicação ACL - Durante o processo de migração, quando uma aplicação monolítica é migrada para microsserviços, pode haver mudanças na semântica do modelo de domínio dos recém-migrados serviço. Quando as características dentro do monólito são obrigadas a chamar esses microsserviços, o As chamadas devem ser roteadas para o serviço migrado sem exigir alterações na chamada serviços. O padrão ACL permite que o monólito chame os microserviços de forma transparente por atuando como um adaptador ou camada de fachada que traduz as chamadas para as mais novas Semântica.

### **_Escala em Cenários de Pico (Black Friday)_**


Para aguentar o salto de 1 milhão para 10 milhões de acessos:
Controle de Tráfego (Throttling): O sistema barra o excesso de acessos para evitar que o serviço fique lento ou saia do ar por sobrecarga.

Banco de Dados Flexível: O banco de dados aumenta sua capacidade sozinho conforme a demanda, sem precisarmos configurar nada manualmente.

Mensageria (SQS): Funciona como uma fila que segura o excesso de mensagens e os entrega aos poucos para os sistemas antigos.


![Arquitetura C4](./imagens/C4%20arquitetura%20infra.png)

# Anexos

_Documentos utilizados no estudo:_

[_https://aws.amazon.com/blogs/architecture/modernization-of-real-time-payment-orchestration-on-aws/_](https://aws.amazon.com/blogs/architecture/modernization-of-real-time-payment-orchestration-on-aws/)

[_https://aws.amazon.com/elasticache/redis/_](https://aws.amazon.com/elasticache/redis/)

[_https://aws.amazon.com/blogs/database/understanding-amazon-dynamodb-latency/_](https://aws.amazon.com/blogs/database/understanding-amazon-dynamodb-latency/)

[DynamoDB global tables versions - Amazon DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/V2globaltables_versions.html)

[Point-in-time backups for DynamoDB - Amazon DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Point-in-time-recovery.html)

[Anti-corruption layer pattern - AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/acl.html)
