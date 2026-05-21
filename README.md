# aws-sftp-server

AWS Transfer Family SFTP server with on-demand lifecycle management. The server is not permanently running — a Lambda function creates and destroys the underlying CloudFormation stack on demand, minimising costs while keeping SFTP access available when needed.

## Architecture

```text
Internet → NLB (Elastic IP) → VPC Endpoint → AWS Transfer Family (SFTP)
                                                       ↓
                                               S3 Storage Bucket
                                                       ↓
                                          handle_s3_events Lambda → Slack
```

### Components

| Component | Description |
|-----------|-------------|
| **AWS CDK (Python)** | IaC — provisions the permanent infrastructure |
| **AWS Transfer Family** | SFTP server (VPC_ENDPOINT type) |
| **NLB** | Internet-facing Network Load Balancer with fixed Elastic IP on port 22 |
| **VPC Endpoint** | Interface endpoint bridging NLB → Transfer service |
| **S3 (storage)** | Upload destination; per-user home directories |
| **S3 (templates)** | Stores CloudFormation templates used at runtime |
| **Secrets Manager** | Stores SSH public keys, one key per username |
| **start_stop Lambda** | Creates or deletes the SFTP server CloudFormation stack on demand |
| **handle_s3_events Lambda** | Sends a Slack notification on every S3 `PutObject` event |

### On-demand lifecycle

The SFTP server itself lives in a separate CloudFormation stack (`<project_name>-sftp-server`). The `start_stop` Lambda manages it:

- `action: "start"` → `cloudformation:CreateStack` using the template in the template bucket
- `action: "stop"` → `cloudformation:DeleteStack`
- `action: "test"` → sends a test Slack notification

This means zero Transfer Family running costs when the server is idle.

---

## Prerequisites

- Python 3.12+
- AWS CDK v2: `npm install -g aws-cdk`
- An existing VPC with a public subnet
- An Elastic IP (pre-allocated)
- A Slack Incoming Webhook URL

---

## Setup

### 1. Clone and install dependencies

```bash
python3 -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate.bat
pip install -r requirements.txt
```

### 2. Create the config file

```bash
cp configs/config.sample.yml configs/config.yml
```

Edit `configs/config.yml`:

```yaml
region: eu-west-1
account_id: 123456789012

project_name: my-project

vpc:
  id: vpc-xxxxxxxx
  eip_allocation_id: eipalloc-xxxxxxxx
  eip_address: 1.2.3.4
  subnet_cidr: 10.0.1.0/24

slack_webhook_url: https://hooks.slack.com/services/...
```

> The config file is loaded at CDK synth time and also injected as Lambda environment variables.

### 3. Create SSH key secrets in AWS Secrets Manager

Create a single **key-value** secret named `<project_name>-project-secrets`. Add one entry per SFTP user:

| Key     | Value               |
|---------|---------------------|
| `Alice` | `ssh-rsa AAAA...`   |
| `Bob`   | `ssh-rsa AAAA...`   |

### 4. Create user folders in the storage bucket

After the first `cdk deploy`, the `<project_name>-sftp-storage-bucket` is created. Add a folder per user/environment combination:

```text
sftp-storage-bucket/
├── AliceStaging/
├── AliceProduction/
├── BobStaging/
└── BobProduction/
```

The folder name becomes the SFTP username and the user's home directory is scoped to it.

### 5. Add users to the CloudFormation template

Edit `templates/cloudformation/sftp-server.yaml`:

**Parameters section** — add one entry per user:

```yaml
Parameters:
  Alice:
    Type: String
    Description: SSH key lookup key for Alice
  Bob:
    Type: String
    Description: SSH key lookup key for Bob
```

**Resources section** — add one nested stack per user/environment:

```yaml
AliceStaging:
  Type: AWS::CloudFormation::Stack
  Properties:
    TemplateURL: !Sub 'https://${TemplateBucket}.s3.eu-west-1.amazonaws.com/sftp-user.yaml'
    Parameters:
      SFTPServerId: !GetAtt SFTPServer.ServerId
      UserName: AliceStaging
      SshPublicKey: !Sub '{{resolve:secretsmanager:${SecretName}:SecretString:${Alice}}}'
      ProjectName: !Ref ProjectName
      S3Bucket: !Ref S3Bucket
      UserRoleArn: !Ref UserRoleArn

AliceProduction:
  Type: AWS::CloudFormation::Stack
  Properties:
    TemplateURL: !Sub 'https://${TemplateBucket}.s3.eu-west-1.amazonaws.com/sftp-user.yaml'
    Parameters:
      SFTPServerId: !GetAtt SFTPServer.ServerId
      UserName: AliceProduction
      SshPublicKey: !Sub '{{resolve:secretsmanager:${SecretName}:SecretString:${Alice}}}'
      ProjectName: !Ref ProjectName
      S3Bucket: !Ref S3Bucket
      UserRoleArn: !Ref UserRoleArn
```

### 6. Build the start/stop Lambda package

The `handle_s3_events` Lambda has no external dependencies and is deployed directly from source.  
The `start_stop_sftp_server` Lambda requires a zip package:

```bash
cd templates/lambda/start_stop_sftp_server

mkdir package
pip install -r requirements.txt -t package/
cp sftp_start_stop_aws_lambda.py package/
cd package/
zip -r ../function.zip .
cd ../../..
```

### 7. Upload CloudFormation templates to S3

After the first deploy (so the template bucket exists), upload the templates:

```bash
aws s3 cp templates/cloudformation/sftp-server.yaml \
  s3://<project_name>-sftp-cloudformation-template-bucket/sftp-server.yaml

aws s3 cp templates/cloudformation/sftp-user.yaml \
  s3://<project_name>-sftp-cloudformation-template-bucket/sftp-user.yaml
```

### 8. Deploy

```bash
cdk bootstrap   # first time only
cdk deploy
```

---

## Usage

### Start / Stop the SFTP server

Invoke the `<project_name>-sftp-start-stop-lambda` with a payload:

```bash
# Start
aws lambda invoke \
  --function-name <project_name>-sftp-start-stop-lambda \
  --payload '{"action": "start"}' \
  response.json

# Stop
aws lambda invoke \
  --function-name <project_name>-sftp-start-stop-lambda \
  --payload '{"action": "stop"}' \
  response.json
```

Slack notifications are sent automatically on start, stop, and errors.

### Connect via SFTP

```bash
sftp -i ~/.ssh/your_private_key AliceStaging@<elastic_ip>
```

---

## CDK commands

```bash
cdk ls        # list stacks
cdk synth     # emit CloudFormation template
cdk diff      # diff deployed vs local
cdk deploy    # deploy to AWS
cdk destroy   # destroy permanent infrastructure
```

---

## Project structure

```text
.
├── app.py                                  # CDK entry point
├── cdk.json                                # CDK config
├── configs/
│   └── config.sample.yml                  # Config template
├── templates/
│   ├── cloudformation/
│   │   ├── sftp-server.yaml               # SFTP server + NLB + users
│   │   └── sftp-user.yaml                 # Per-user nested stack
│   ├── infrastructure/
│   │   └── infrastructure_stack.py        # CDK stack (S3, IAM, Lambda)
│   └── lambda/
│       ├── start_stop_sftp_server/
│       │   └── sftp_start_stop_aws_lambda.py
│       └── handle_s3_events/
│           └── handle_s3_events.py
├── utilities/
│   └── utility.py                         # Config loader, helpers
├── requirements.txt
└── requirements-dev.txt
```
