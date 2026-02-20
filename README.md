# 

# 

# **DEMO: Arquitetura de Solução**

# 

## 

## **Emissão de Boleto no Mainframe a partir de Requisição Externa**

## *Fevereiro 2026*

## 

## *Version: 0.1*

**Histórico do Documento**

**Revisão do Documento**

| Revisão | Data | Criado | Descrição resumida das mudanças |
| :---: | :---: | :---: | ----- |
| 0.1 | 19/02/2026 | Fabrizio Rodrigues | Initial Version |

**Conteúdo**

1\.	Descrição da Solução	[4](#descrição-da-solução)

1.1	Objetivos e regras de negócio	[4](#objetivos-e-regras-de-negócio)

1.2	Escopo geral do projeto	[4](#escopo-geral-do-projeto)

2\.	Cenário de negócio	[5](#cenário-de-negócio)

2.1	Visão estado atual – AS-IS	[5](#visão-estado-atual-–-as-is)

2.2	Oportunidades e Soluções – TO-BE	[8](#oportunidades-e-soluções-–-to-be)

2.2.1	Cenário 1: Híbrido (Refatoração Java \+ MQ \+ Mainframe	[8](#cenário-1:-híbrido-\(refatoração-java-+-mq-+-mainframe\))

2.2.2	Cenário 2: Migração Cloud	[11](#cenário-2:-migração-cloud)

3\.	Anexos	[15](#anexos)

1. # **Descrição da Solução** {#descrição-da-solução}

   1. ## **Objetivos e regras de negócio** {#objetivos-e-regras-de-negócio}

Migrar para nuvem o funcionamento de um sistema em mainframe para gerar boletos a partir de requisições externas.

2. ## **Escopo geral do projeto** {#escopo-geral-do-projeto}

O escopo dos seguintes Sistemas que serão integrados:

* Canais Digitais (e-commerce)  
* Sistema dentro de arquitetura Monolitico  
* AWS Cloud  
* Sistema Mainframe  
* Banco de Dados

1. # **Cenário de negócio** {#cenário-de-negócio}

   3. ## **Visão estado atual – AS-IS** {#visão-estado-atual-–-as-is}

A partir de uma requisição externa, O cliente (ex: e-commerce) envia uma solicitação de emissão de boleto através de uma chamada SOAP **(fire-in-forget)**, os dados como cpf/CNPJ do pagador, valor vencimento, instruções e o CNPJ do beneficiário não entram diretamente no COBOL, mas sim por um web server (webshpere) onde através de um conector JNI é executada uma transação CICS. A partir do recebimento, o programa COBOL lê os dados, aplica as regras de negocio, calcula o “nosso numero” e persiste os dados do boleto em um banco de dados DB2. Há uma regra secundaria que nao será estará no cenário de migração, porém faz parte da jornada da emissao do Boleto: Se o boleto for registrado, o dado é enviado para CIP para que ela homologue o boleto para ser demonstrado para os demais canais bancários do cpf/CNPJ do pagador

Segue o Fluxo atual de negócio.

![Diagrama BPMN](./imagens/diagrama%20de%20negocio.png)

Abaixo, detalhe das etapas desse processo:

**1\. Entrada da Requisição (JSP)**

Um cliente externo (ex: site de e-commerce) envia uma requisição **HTTP/SOAP** utilizando Javascript (Fetch API) inserido no HTML para enviar o envelope XML.

**2\. Web server (WAS)**

Como o mainframe geralmente não processa essas chamadas diretamente em sua forma nativa, utiliza-se o middleware usando JNDI (Java Naming and Directory Interface) para traduzir o Payload (XML) em um formato que o mainframe entenda (commarea).

**3\. Processamento Transacional (CICS)**

A requisição chega ao **CICS (Customer Information Control System)**, o monitor de transações do mainframe para identificar a transação correta a ser executada com base nos dados recebidos.

Um programa, geralmente escrito em **COBOL** (copybook), valida os dados do cliente, realiza o cálculo do boleto e aplica as regras de cobrança no banco de dados (como o **DB2**).

**4\. Registro do Boleto**

Os dados da emissão do bolerto são acumulados para processamento em lote (**Batch**), onde os arquivos de remessa são gerados para registrar as cobranças. O Boleto deve estar registrado junto à CIP.

**5\. Resposta da Requisição**

Após gerar a linha digitável e o código de barras, o mainframe devolve os dados para a camada de transação, o middleware converte os dados de volta para XML e o cliente externo recebe a confirmação ou o PDF do boleto para exibição de forma assincrona.

Segue o Diagrama dos componentes atuais.

![Diagrama C4](./imagens/diagrama%20C4%20-%20cenario%20atual.png)

4. ## **Oportunidades e Soluções – TO-BE** {#oportunidades-e-soluções-–-to-be}

   1. ### **Cenário 1: Híbrido (Refatoração Java \+ MQ \+ Mainframe)** {#cenário-1:-híbrido-(refatoração-java-+-mq-+-mainframe)}

Modernizar a experiência do cliente e a lógica de negócio na nuvem **AWS**, mantendo a persistência de dados no **Mainframe** (Edge) via mensageria assíncrona.

Os componentes para compor esta arquitetura são definidos em Camada de Entrada e Segurança, Camada de Conteinerização, Camada de Integração e a Camada de Processamento atual, conforme os recursos abaixo:

* **Amazon API Gateway:** Porta de entrada para o e-commerce. Ele recebe a requisição REST/JSON, faz o throttling (controle de vazão) e gerencia o versionamento da API.

* **AWS Lambda ou Cognito:** Antes de chegar ao microserviço, a requisição é validada para garantir que o usuário está autenticado e a transformação de XML para JSON é realizada dando transparência para o sistema externo.

* **AWS EKS microserviço de Autenticação (Pod A):** Responsável apenas por validar o token e o perfil do cliente.

* **AWS EKS microserviço de Geração de Boleto (Pod B \- Java/Spring):**

  •	Contém a lógica refatorada do antigo COBOL (cálculos de datas, juros e linha digitável)

  **•	IBM MQ Client:** Biblioteca Java integrada ao Spring Boot que encapsula a mensagem JSON/XML e a coloca na fila de envio.

* **AWS Direct Connect ou VPN Site to Site: **Túnel dedicado e seguro que interliga a VPC da AWS ao Data Center onde o Mainframe está hospedado.

* **IBM MQ (Queue Manager): **Gerenciador de filas que garante que, se o mainframe ficar offline por segundos, a mensagem não se perca; ela fica retida na fila até ser consumida.

* **IBM MQ for z/OS:** Recebe a mensagem vinda da nuvem**.**

* **Trigger Monitor:** Componente do CICS que identifica a chegada de uma nova mensagem na fila e dispara automaticamente o programa COBOL**.**

* **Proxy COBOL (Consumidor):** Recebe os dados e executa o EXEC SQL INSERT no DB2 


![Diagrama Infra](./imagens/diagrama%20de%20arquitetura%20cenario%201%20-%20migracao%20modelo%20hibrido.png)

***Processo de Refatoração com os padrões recomendados (clean architecture \+ DDD):***

A **extração de Regras** deverá ser realizada com o uso da AWS Bedrock onde a partir do código fonte do COBOL o prompt deve ser solicitado para extrair o "Business Logic" em linguagem natural (Pseudo-código). O procedimento evita que erros do legado passem para o Java.

Com o pseudo-código, o Bedrock gera as classes Java, os DTOs, os Dominios, Entidades, Usecase e a forma de integrar os eventos com o API Gateway e a integração com o IBM MQ.

A IA identifica como o Copybook do Mainframe deve ser montado no Java para que o **Proxy COBOL** a partir da fila MQ receba os dados corretamente

O prompt na IA deve seguir de forma clara e estruturada esperando o seguinte resultado:

A refatoração do COBOL para Java (usando o Claude 3.5 no Bedrock) deve isolar a regra de negócio da infraestrutura:

* Domain: Onde reside o "coração" do sistema (Entidades de Boleto, Value Objects, Regras de Cálculo de Juros). Sem dependências externas.

* Application (Interfaces/DTOs): Contém os casos de uso (ex: GerarBoletoUseCase) e os objetos de transferência de dados (DTOs) para comunicação com o API Gateway.

* Infrastructure: Implementação de detalhes técnicos como o cliente do IBM MQ, repositórios e adaptadores para o Amazon SQS.

***Estratégia de Resiliência e Event-Driven*** 

O uso do AWS SQS pelo microserviço antes do MQ será para isolar falhas no **Direct Connect**:

1. O Microserviço de Boleto processa a regra e faz o put no **Amazon SQS**.

2. Um componente (pode ser um worker no EKS ou um BFF especialista) consome o SQS e tenta enviar para o **IBM MQ** no Mainframe.

3. Se o link físico cair e causar falha no Direct Connect, as mensagens ficam retidas e seguras no Amazon SQS.

4. Caso uma mensagem falhe repetidamente (ex: dados corrompidos que o COBOL não aceita), ela é movida para uma **DLQ (dead letter queue)**. Isso evita que mensagens com erro tranquem a fila principal.

*Retentativa***:** Assim que o Direct Connect volta, o fluxo de consumo do SQS processa o acúmulo automaticamente, garantindo a eventual consistência no **DB2.**

***Estratégia de Observabilidade***

Para que o time de sustentação acompanhe de forma eficaz as métricas e logs, o uso de log estruturado pela aplicação enviando para o elasticSearch para que seja visualizado no Kibana indexando chaves como transaction-id e um correlation-id para facilitar o tracing do API Gateway até o Proxy COBOL permitindo o rastreio do boleto em todo ambiente hibrido.O uso do AWS Cloudwatch ajuda a rastrear o inicio do fluxo.

 

***Vantagem estratégica:***

Este cenário permite que o "switch off" não custe caro o processamento lógico no Mainframe (MIPS), movendo-o para o custo reduzido do Kubernetes na AWS, mantendo a segurança de ter os dados financeiros no ambiente legado durante a transição.

Para o processo de refatoração sugiro o recurso de IA da **AWS Bedrock** com o modelo ***Claude 3.5 Sonnet*** que na interpretação de lógicas legadas (como programas COBOL), ele consegue entender as regras de negócio e sugerir a melhor estrutura em Java/Spring Boot melhor do que o LLM ***GPT-4***.

Para nível de comparação, o uso do **IBM watsonx Code Assistant for Z** sendo uma ferramenta específica para tradução de COBOL para Java foi avaliada porque é treinada pela IBM especificamente para traduzir código COBOL para Java.

Como a sugestão do planejamento é o uso dos recursos cloud na AWS, utilizar o **AWS Bedrock** faz total sentido porque esta no ecossistema e o codigo por ser interno, nao serao usado para dados publicos. 

O diferencial de utilizar **o BFF compartilhando a fila SQS** é porque ele atua como um **"Circut Breaker"** inteligente. Se ele detecta que o MQ está fora, ele para de consumir o SQS, deixando as mensagens acumularem lá com segurança até que o caminho para o Mainframe seja restabelecido.

2. ### **Cenário 2: Migração Cloud** {#cenário-2:-migração-cloud}

Neste cenário, o switch off é total do Mainframe pois a aplicação atinge o estado de maturidade máxima em nuvem, utilizando os serviços gerenciados para eliminar a carga operacional e os possíveis custos fixos de hardware legado.

Neste cenário, o fluxo de integração é simplificado, removendo a necessidade de mensageria assíncrona para persistência, pois o banco de dados está na mesma rede que os microserviços.

Os componentes para compor esta arquitetura são complementares ao cenário 1, incluindo Camada de  borda, camada de Devops, Camada de Rede:

* **AWS Cognito** gerencia o pool de usuários e fornece tokens JWT.

* **AWS Aurora PostgreSQL** substitui o DB2. Configurado em **Multi-AZ**, garante que se uma zona falhar, o banco de dados faça failover automático em segundos.

* **AWS S3:** Os arquivos gerados (PDF) são gravados com políticas de ciclo de vida para reduzir custos de armazenamento a longo prazo.

* **CloudFront & AWS WAF**: Protegem contra ataques L3/4 (DDoS) e L7 (SQL Injection/Cross-site Scripting).

* **AWS Certificate Manager:** Gerencia os certificados SSL/TLS para criptografia em trânsito de ponta a ponta.

* **AWS Systems Manager (SSM)** Parameter Store: Centraliza configurações, permitindo os configurar parâmetros (como URLs de serviços externos) sem precisar recompilar o código ou alterar arquivos YAML dentro do Kubernetes.

* **Multi-AZ**: A VPC é distribuída em Zonas de Disponibilidade (AZs).

* Subnets **Públicas**: Uso do Internet Gateway e os NAT Gateways.

* Subnets **Privadas** (Aplicação): Onde residem os worker nodes do EKS.

* Subnets Privadas (Dados): Subnets isoladas apenas para o banco de dados Aurora, sem saída para a internet.

* **SG**\-LoadBalancer: Permite entrada apenas nas portas 443 vindas do API Gateway.

* **SG**\-EKS-Nodes: Permite entrada apenas do SG-LoadBalancer.

* **SG**\-Database: Permite entrada apenas do SG-EKS-Nodes na porta XXX do PostgreSQL.

* **AWS CodeCommit:** Repositório privado Git onde o código refatorado (COBOL \-\> Java) é armazenado

* **AWS CodeBuild:** Acionado pelo Pipeline para:

* Compilar o Java.

* Executar testes unitários e de integração.

* Gerar a imagem Docker.

* Fazer o Security Scan da imagem.

* **AWS ECR**: O CodeBuild faz o push da imagem versionada para o registro de containers.

* **AWS CodeDeploy / Helm**: O pipeline atualiza o manifesto do Kubernetes no EKS para realizar o rolling uypdate (substituindo os Pods sem interromper o serviço).

* **IAM Roles for Service Accounts**: O Pod no EKS não usará chaves fixas (Access Keys) para acessar o S3 ou o DB. Ele usará uma Role temporária do IAM, seguindo o princípio do menor privilégio.

* **VPC Endpoints**: Para que o EKS fale com o ECR ou S3 sem precisar sair pela internet (via NAT Gateway). O tráfego nunca sai da rede global da AWS.

* **AWS Cloudformation:** serviço nativo para Infraestrutura como Código (IaC)
* **AWS SES:** O SES envia o e-mail com o link do boleto (gerado via S3 Pre-signed URL) para o cliente

***Padrões recomendados (clean architecture \+ DDD):***

A aplicação Java (Spring Boot) pode usar o SDK para buscar os parâmetros no startup. Ou, de forma mais nativa ao Kubernetes, usar o **Secrets Store CSI Driver**, que monta os parâmetros do SSM como arquivos dentro do volume do Pod ou como variáveis de ambiente nativas

***Estratégia de Resiliência***

Neste cenário não existe a dependência do link físico (Direct Connect) para o isolamento de falhas nos microsserviços distribuídos. Para garantir que o cliente não fique "travado" com o tempo de resposta alto de forma sincrona, sugiro a utilização de padrões de Mensageria Assíncrona e Fallback e o envio de um "Protocolo de Processamento". Isso libera o e-commerce para continuar a jornada do cliente.

O Worker denominado Producer (Boleto Pod) recebe a requisição, valida os dados e em vez de tentar gravar no Aurora ele posta a mensagem na fila do Amazon SQS.

Outra mudança, ocorre no Worker de Processamento denominado Consumer, ou seja, consome a fila SQS, gera o boleto e grava no Aurora.

Se o Amazon Aurora sofrer um failover, a mensagem volta para a fila (Retentativa). Se falhar mais 1 vez, vai para a DLQ. O processo apenas fica retido para correção através da area de sustentação.

***Vantagem estratégica:***

Este cenário apresenta a completa jornada de modernização. A estratégia de IaC é para garantir que por exemplo o banco de dados e o cluster de containers suba em minutos. 
Uso do pilar de Confiabilidade (AWS Well-Architected), pois o uso de SQS/DLQ impede a perda de dados em picos de carga ou falhas de banco.

Segue o diagrama C4 e o diagrama de arquitetura completo abaixo:

![Diagrama Infra](./imagens/diagrama%20de%20infra%20cenario%202%20-%20migracao%20full.png?)

![Diagrama C4](./imagens/diagrama%20C4%20-%20nivel%20c2%20-%20Proposta.png?)

2. # **Anexos** {#anexos}

*N/A*


