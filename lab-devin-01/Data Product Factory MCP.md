# Data Product Factory MCP

## Purpose

Expose Data Product generation capabilities to an AI coding agent.

The MCP server provides controlled tools for interpreting Data Product definitions, generating infrastructure and validating generated artifacts.

The MCP server MUST NOT deploy infrastructure.

---

# Tools

## 1. validate_data_product

### Description

Validate a Data Product definition before generating infrastructure.

### Input

```json
{
  "definition": "<markdown or yaml>"
}
```

### Output

```json
{
  "valid": true,
  "errors": [],
  "warnings": [],
  "architecture": {
    "source": "sql_table",
    "layers": ["TEMP", "SOR", "SOT"],
    "orchestration": "glue"
  }
}
```

---

# 2. generate_data_product

### Description

Generate the complete Data Product infrastructure and processing artifacts.

### Input

```json
{
  "definition": "<markdown or yaml>",
  "output_path": "<repository path>"
}
```

### Output

```json
{
  "status": "success",

  "architecture": {
    "source": "kafka_topic",
    "layers": [
      "TEMP",
      "SOR",
      "SOT",
      "SPEC"
    ],
    "orchestration": "step_functions"
  },

  "artifacts": {
    "terraform": [
      "terraform/s3.tf",
      "terraform/glue.tf",
      "terraform/iam.tf",
      "terraform/lambda.tf",
      "terraform/step_functions.tf"
    ],
    "glue": [
      "glue/sor_job.py",
      "glue/sot_job.py",
      "glue/spec_job.py"
    ],
    "lambda": [
      "lambda/kafka_sink.py"
    ],
    "step_functions": [
      "step-functions/spec_workflow.json"
    ]
  }
}
```

---

# 3. validate_terraform

### Description

Validate generated Terraform without deploying infrastructure.

### Input

```json
{
  "path": "<terraform path>",
  "run_plan": false
}
```

### Allowed commands

```text
terraform fmt
terraform validate
```

Optional:

```text
terraform plan
```

### Forbidden commands

```text
terraform apply
terraform destroy
terraform import
```

### Output

```json
{
  "valid": true,

  "fmt": "passed",

  "validate": "passed",

  "plan": "not_executed",

  "deployment": {
    "executed": false
  }
}
```

---

# 4. explain_architecture

### Description

Explain why the selected AWS architecture was generated.

### Input

```json
{
  "definition": "<markdown or yaml>"
}
```

### Output

```json
{
  "source": "kafka_topic",

  "decision": [
    "Kafka requires Lambda Sink",
    "TEMP is used as landing layer",
    "Glue processes TEMP into SOR",
    "SOT is derived from SOR",
    "SPEC requires Step Functions orchestration"
  ]
}
```

---

# MCP Security Policy

The MCP server MUST enforce:

```text
ALLOW
 ├── read templates
 ├── read policies
 ├── parse definitions
 ├── generate files
 ├── terraform fmt
 ├── terraform validate
 └── terraform plan

DENY
 ├── terraform apply
 ├── terraform destroy
 ├── terraform import
 ├── AWS resource creation
 └── AWS resource deletion
```

The MCP server MUST NOT expose generic shell execution to the AI agent.

Only explicitly allowed commands may be executed.