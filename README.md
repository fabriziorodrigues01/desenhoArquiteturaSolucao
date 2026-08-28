# devin lab terraform data products
- Como isso fica no Devin
DEVIN
  │
  │ "Crie um Data Product para..."
  ▼
DATA PRODUCT SKILL
  │
  │ interpreta intenção
  ▼
MCP
  │
  ├── validate_data_product
  │
  ├── explain_architecture
  │
  └── generate_data_product
  │
  ▼
TEMPLATE REGISTRY
  │
  ├── S3
  ├── Glue
  ├── Glue Catalog
  ├── Lambda
  └── Step Functions
  │
  ▼
GENERATED REPOSITORY
  │
  ├── Terraform
  ├── Glue
  ├── Lambda
  └── Step Functions
  │
  ▼
validate_terraform
  │
  ▼
GIT / PR

------

E há uma decisão muito importante

nao inclua AWS Credentials no MCP.

A primeira versão pode ser completamente offline em relação à AWS:

Markdown → Skill → MCP → Templates → Terraform → Validate → PR

Isso deixa o componente muito mais seguro e transforma o MCP em um gerador governado de infraestrutura, não em um agente com acesso direto à sua conta AWS.

Depois podemos evoluir para uma segunda camada, se necessário, onde o CI/CD da empresa seja o responsável por fazer o apply.

Próximo passo

Agora temos o contrato da Skill + MCP. O próximo passo natural é construir o Template Registry Terraform V1, começando pelos 4 templates fundamentais:

**SQL → TEMP → SOR
Kafka → Lambda Sink → TEMP → SOR
SOR → SOT
SOT → SPEC com Step Functions + Glue**

A partir desses quatro templates, conseguimos montar uma primeira versão realmente funcional da Data Product Factory.

----------
System Prompt do Devin não descreva cada pipeline individualmente. Ele deve funcionar como um orquestrador, obrigando o Devin a usar a Skill/MCP e seguir o padrão Data Mesh.

System Prompt — Data Product Engineer

----------
Esse prompt cria uma separação importante: o Devin decide o que precisa ser feito; a Skill conhece o padrão; o MCP controla as operações; os templates controlam como a infraestrutura é construída.

E isso deixa espaço para a próxima evolução: o desenvolvedor poder simplesmente escrever “gere uma pipe para esta tabela”, e o Devin conduzir todo o processo sem precisar conhecer Terraform, Glue ou Step Functions em detalhes.

------------
## prompts de exemplo
### Exemplo 1 — SQL → SOR

Crie um Data Product para a tabela abaixo.

```sql
CREATE TABLE investment_transaction (
    transaction_id STRING,
    account_id STRING,
    amount DECIMAL(18,2),
    transaction_date TIMESTAMP,
    transaction_type STRING
);
```

Fonte: SQL

Quero criar somente a camada SOR.

Use o padrão Data Mesh definido no Data Product Factory, gere os recursos AWS usando os templates Terraform existentes e utilize Glue + Glue Catalog.

Não execute Terraform apply.

------------
### Exemplo 2 — Kafka → SOR

Crie uma pipeline Data Mesh para o tópico Kafka:

Topic: investment.transactions
Format: Avro
Consumer Group: investment-sor

Quero:

Kafka → Lambda Sink → S3 TEMP → Glue → SOR → Glue Catalog.

Use os templates Terraform existentes.

Não execute deployment na AWS.


------------

### Exemplo 3 — SOR → SOT

Para o Data Product `investment-transactions`, crie uma camada SOT a partir da tabela SOR `investment_transaction`.

A transformação deve ser:

```sql
SELECT
    account_id,
    transaction_date,
    SUM(amount) AS total_amount
FROM investment_transaction
GROUP BY account_id, transaction_date;
```

Use Glue para o processamento, S3 para armazenamento e Glue Catalog para catalogação.

Utilize os templates existentes e não execute Terraform apply.


----------------

### Exemplo 4 — SOT → SPEC

Crie a camada SPEC para o Data Product `investment-transactions`, usando a SOT existente.

A SPEC deve validar:

* `amount > 0`
* `account_id IS NOT NULL`
* `transaction_date IS NOT NULL`

Use Step Functions para orquestrar as etapas e Glue para executar as validações e transformações.

Publique o resultado em S3 e registre a tabela no Glue Catalog.

Gere somente os artefatos Terraform e código necessários. Não faça deployment.

---------------

prompt ainda mais simples, por exemplo:

"Crie uma pipe SOR para esta tabela: [CREATE TABLE...]"

A Skill/MCP faria automaticamente a interpretação, escolheria TEMP + SOR + Glue + Glue Catalog + Terraform, validaria e entregaria o projeto pronto para PR.
