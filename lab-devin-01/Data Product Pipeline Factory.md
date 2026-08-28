# Data Product Pipeline Factory

## Purpose

Generate standardized Data Product pipelines for a Data Mesh architecture using AWS services and Terraform.

The skill receives a declarative Data Product definition and generates the required infrastructure and processing artifacts.

The skill MUST NOT deploy infrastructure.

---

# 1. Core Principles

The skill MUST follow these principles:

1. Data Product first.
2. Infrastructure as Code using Terraform.
3. AWS Glue is the standard data processing and catalog service.
4. Glue Data Catalog MUST be used for SOR, SOT and SPEC datasets.
5. S3 is the standard storage layer.
6. Kafka sources MUST use Lambda Sink to S3 TEMP.
7. SOR is the standardized raw/normalized persistence layer.
8. SOT is derived from one or more SOR datasets.
9. SPEC is optional and represents business/regulatory/functional processing.
10. Step Functions is an orchestration layer, not a replacement for Glue.
11. Templates MUST be used instead of generating infrastructure freely.
12. Security, naming and tagging policies MUST be respected.
13. Terraform MUST be validated before returning the result.
14. Terraform Apply MUST NEVER be executed by this skill.

---

# 2. Supported Sources

The skill supports:

```yaml
source:
  type:
    - sql_table
    - kafka_topic
```

## SQL Source

The developer may provide:

```sql
CREATE TABLE investment_transaction (
    transaction_id STRING,
    account_id STRING,
    amount DECIMAL(18,2),
    transaction_date TIMESTAMP
);
```

The skill MUST extract:

- table name
- columns
- data types
- nullable information when available
- constraints when available

---

## Kafka Source

The developer may provide:

```yaml
source:
  type: kafka_topic

  kafka:
    topic: investment.transactions
    message_format: avro
    consumer_group: investment-sor
```

The skill MUST generate a Lambda Sink configuration.

---

# 3. Data Layers

The supported layers are:

```text
TEMP
SOR
SOT
SPEC
```

The standard architecture is:

```text
SOURCE
   |
   v
TEMP
   |
   v
SOR
   |
   v
SOT
   |
   v
SPEC
```

Not every Data Product requires all layers.

---

# 4. TEMP

TEMP is a temporary landing area.

Standard:

```yaml
temp:
  enabled: true

  storage:
    type: s3

  format: parquet
```

TEMP SHOULD have:

- lifecycle policy
- encryption
- restricted IAM access
- standardized prefix
- retention policy

TEMP is not considered a governed consumption dataset.

---

# 5. SOR

SOR means System of Record.

SOR represents the standardized persisted representation of the source data.

Required components:

```text
S3
Glue Job
Glue Database
Glue Table
IAM
```

Example:

```yaml
sor:
  enabled: true

  storage:
    type: s3
    format: parquet

  catalog:
    enabled: true

  ingestion:
    engine: glue
```

The skill MUST generate:

```text
S3 SOR
Glue Job
Glue Database
Glue Table
IAM Role
```

---

# 6. SOT

SOT means System of Truth.

SOT MUST be derived from SOR or other governed datasets.

Example:

```yaml
sot:
  enabled: true

  source_tables:
    - database: investment_sor
      table: transaction

  transformation:
    type: sql

    sql: |
      SELECT
          account_id,
          SUM(amount) AS total_amount,
          transaction_date
      FROM transaction
      GROUP BY
          account_id,
          transaction_date
```

Standard architecture:

```text
SOR
 |
 v
Glue Job
 |
 +-- Join
 +-- Filter
 +-- Aggregate
 +-- Business Transformation
 +-- Data Quality
 |
 v
SOT
 |
 v
Glue Catalog
```

---

# 7. SPEC

SPEC is optional.

It MUST be used when the Data Product requires complex business, regulatory, fiduciary or domain-specific processing.

Example:

```yaml
spec:
  enabled: true

  type: fiduciary

  source:
    database: investment_sot
    table: consolidated_transaction

  rules:

    - name: validate_amount
      type: validation
      expression: "amount > 0"

    - name: validate_account
      type: validation
      expression: "account_id IS NOT NULL"
```

When SPEC is enabled, the default orchestration is:

```text
Step Functions
       |
       +--> Glue Validation
       |
       +--> Glue Transformation
       |
       +--> Glue Business Rules
       |
       +--> Glue Data Quality
       |
       +--> Glue Publish
       |
       v
      SPEC
       |
       v
Glue Catalog
```

Step Functions MUST orchestrate processing.

Glue MUST perform data processing.

---

# 8. Architecture Decision Rules

The skill MUST apply these rules.

## Rule 1

If:

```yaml
source.type: sql_table
```

generate:

```text
SQL
 |
 v
TEMP
 |
 v
Glue
 |
 v
SOR
```

---

## Rule 2

If:

```yaml
source.type: kafka_topic
```

generate:

```text
Kafka
 |
 v
Lambda Sink
 |
 v
TEMP
 |
 v
Glue
 |
 v
SOR
```

---

## Rule 3

If:

```yaml
sot.enabled: true
```

generate:

```text
SOR
 |
 v
Glue
 |
 v
SOT
 |
 v
Glue Catalog
```

---

## Rule 4

If:

```yaml
spec.enabled: true
```

generate:

```text
SOT
 |
 v
Step Functions
 |
 +--> Glue
 +--> Glue
 +--> Glue
 |
 v
SPEC
 |
 v
Glue Catalog
```

---

# 9. Template Selection

The skill MUST NOT freely invent Terraform resources.

Templates MUST be selected from:

```text
templates/
├── common/
├── s3/
├── glue/
├── lambda/
└── step-functions/
```

Template selection:

```text
SQL
 -> s3
 -> glue

Kafka
 -> lambda-kafka-sink
 -> s3
 -> glue

SOT
 -> glue-transformation

SPEC
 -> step-functions
 -> glue-transformation
```

---

# 10. Required AWS Resources

Depending on the requested layers, the skill may generate:

### Storage

```text
S3 TEMP
S3 SOR
S3 SOT
S3 SPEC
```

### Glue

```text
Glue Database
Glue Table
Glue Job
Glue Catalog
```

### Compute

```text
Lambda
```

### Orchestration

```text
Step Functions
```

### Security

```text
IAM Roles
IAM Policies
KMS configuration when required
```

---

# 11. Generated Project Structure

The generated Data Product MUST follow:

```text
<data-product>/
│
├── terraform/
│   ├── main.tf
│   ├── provider.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── s3.tf
│   ├── glue.tf
│   ├── glue_catalog.tf
│   ├── iam.tf
│   ├── lambda.tf
│   └── step_functions.tf
│
├── glue/
│   ├── sor_job.py
│   ├── sot_job.py
│   └── spec_job.py
│
├── lambda/
│   └── kafka_sink.py
│
├── step-functions/
│   └── spec_workflow.json
│
├── data-product.yaml
└── README.md
```

Only files required by the selected architecture should be generated.

---

# 12. Terraform Validation

After generation, the skill MAY execute:

```bash
terraform fmt
terraform validate
terraform plan
```

`terraform plan` is optional and MUST NOT create or modify AWS resources.

The skill MUST NOT execute:

```bash
terraform apply
terraform destroy
terraform import
```

These commands are prohibited.

---

# 13. Output Contract

The skill MUST return:

```yaml
result:
  status: success|failed

  data_product:
    name: <name>
    domain: <domain>

  source:
    type: sql_table|kafka_topic

  layers:
    temp: true|false
    sor: true|false
    sot: true|false
    spec: true|false

  architecture:
    orchestration: glue|step_functions

  resources:
    - S3
    - Glue
    - Glue Catalog
    - IAM
    - Lambda
    - Step Functions

  artifacts:
    terraform: []
    glue: []
    lambda: []
    step_functions: []

  validation:
    terraform_fmt: passed|failed
    terraform_validate: passed|failed
    terraform_plan: passed|failed|not_executed

  deployment:
    executed: false
```

The final field MUST always be:

```yaml
deployment:
  executed: false
```

---

# 14. Failure Handling

The skill MUST stop generation when:

- required source information is missing
- schema cannot be interpreted
- unsupported data type is provided
- required SOR information is missing
- SOT references an unknown source table
- SPEC contains invalid rules
- required template does not exist
- Terraform validation fails

The skill MUST explain the failure and identify the missing or invalid information.

---

# 15. Golden Rule

The skill is a:

> Data Product Infrastructure Generator.

It is NOT a deployment engine.

Its responsibility ends at:

```text
Interpret
   ↓
Validate
   ↓
Decide Architecture
   ↓
Select Templates
   ↓
Generate
   ↓
Terraform Validate
   ↓
Return Artifacts
```

Never deploy.