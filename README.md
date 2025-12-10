# Sentry OMS - CloudFormation

Infraestrutura AWS para o Order Management System.

## Ordem de Deploy

As stacks devem ser criadas na seguinte ordem:

```
LAYER 0 - Foundation (sem dependencias)
├── 1. vpc
├── 2. cognito
├── 3. dynamodb-config
├── 4. dynamodb-trading
├── 5. sqs
└── 6. secrets

LAYER 1 - Data (depende: vpc)
├── 7. redis
└── 8. alb

LAYER 2 - Compute (depende: vpc, sqs, dynamodb)
├── 9.  cluster-gateway
└── 10. cluster-oms

LAYER 3 - Services (depende: clusters, alb, cognito)
├── 11. task-frontend
├── 12. task-oms
└── 13. task-websocket

LAYER 4 - CI/CD (opcional)
└── 14. pipeline-*
```

## Templates

| Stack | Template | Descricao |
|-------|----------|-----------|
| vpc | `networking/vpc-nonprod.json` | VPC, Subnets, NAT Gateway |
| cognito | `cognito/cognito-user-pool.json` | User Pool com MFA |
| dynamodb-config | `dynamodb/oms-config-table.json` | Tabela de configuracoes |
| dynamodb-trading | `dynamodb/oms-trading-table.json` | Tabela de trading |
| sqs | `messaging/sqs-queues.json` | Filas FIFO e Standard |
| secrets | `secrets/secrets-manager.json` | Secrets Manager |
| redis | `data/elasticache-redis.json` | ElastiCache Redis |
| alb | `networking/alb-frontend.json` | Application Load Balancer |
| cluster-gateway | `compute/ecs-cluster-gateway.json` | ECS para Frontend/WebSocket |
| cluster-oms | `compute/ecs-cluster-oms.json` | ECS para OMS |
| task-frontend | `compute/task-definition-frontend-admin.json` | Frontend Service |
| task-oms | `compute/task-definition-oms.json` | OMS Service |
| task-websocket | `compute/task-definition-websocket.json` | WebSocket Service |
| pipeline-frontend | `cicd/codepipeline-frontend.json` | CI/CD Frontend |
| pipeline-oms | `cicd/codepipeline-oms.json` | CI/CD OMS |
| pipeline-websocket | `cicd/codepipeline-websocket.json` | CI/CD WebSocket |

## Nomenclatura

```
sentry-{stack}-{env}

Exemplos:
- sentry-vpc-dev
- sentry-cognito-staging
- sentry-cluster-gateway-prod
```

## Deploy

### Criar Stack

```bash
aws cloudformation create-stack \
  --stack-name sentry-vpc-dev \
  --template-body file://templates/networking/vpc-nonprod.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=AvailabilityZone1,ParameterValue=us-east-1a \
    ParameterKey=AvailabilityZone2,ParameterValue=us-east-1b \
  --region us-east-1
```

### Atualizar Stack

```bash
aws cloudformation update-stack \
  --stack-name sentry-vpc-dev \
  --template-body file://templates/networking/vpc-nonprod.json \
  --parameters ParameterKey=Environment,ParameterValue=dev \
  --region us-east-1
```

### Ver Status

```bash
# Status
aws cloudformation describe-stacks --stack-name sentry-vpc-dev --query 'Stacks[0].StackStatus'

# Eventos
aws cloudformation describe-stack-events --stack-name sentry-vpc-dev --query 'StackEvents[0:5]'

# Outputs
aws cloudformation describe-stacks --stack-name sentry-vpc-dev --query 'Stacks[0].Outputs'
```

## Change Sets

Para revisar mudancas antes de aplicar:

```bash
# 1. Criar change set
aws cloudformation create-change-set \
  --stack-name sentry-vpc-dev \
  --change-set-name my-changes \
  --template-body file://templates/networking/vpc-nonprod.json

# 2. Ver mudancas
aws cloudformation describe-change-set \
  --stack-name sentry-vpc-dev \
  --change-set-name my-changes

# 3. Aplicar
aws cloudformation execute-change-set \
  --stack-name sentry-vpc-dev \
  --change-set-name my-changes

# Ou cancelar
aws cloudformation delete-change-set \
  --stack-name sentry-vpc-dev \
  --change-set-name my-changes
```

Ver mais em [docs/CHANGESET.md](docs/CHANGESET.md).

## Comandos por Layer

### Layer 0 - Foundation

```bash
# VPC
aws cloudformation create-stack \
  --stack-name sentry-vpc-dev \
  --template-body file://templates/networking/vpc-nonprod.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=AvailabilityZone1,ParameterValue=us-east-1a \
    ParameterKey=AvailabilityZone2,ParameterValue=us-east-1b

# Cognito
aws cloudformation create-stack \
  --stack-name sentry-cognito-dev \
  --template-body file://templates/cognito/cognito-user-pool.json \
  --parameters ParameterKey=Environment,ParameterValue=dev

# DynamoDB Config
aws cloudformation create-stack \
  --stack-name sentry-dynamodb-config-dev \
  --template-body file://templates/dynamodb/oms-config-table.json \
  --parameters ParameterKey=Environment,ParameterValue=dev

# DynamoDB Trading
aws cloudformation create-stack \
  --stack-name sentry-dynamodb-trading-dev \
  --template-body file://templates/dynamodb/oms-trading-table.json \
  --parameters ParameterKey=Environment,ParameterValue=dev

# SQS
aws cloudformation create-stack \
  --stack-name sentry-sqs-dev \
  --template-body file://templates/messaging/sqs-queues.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=Queue1Name,ParameterValue=master-order \
    ParameterKey=Queue2Name,ParameterValue=bitget-orders \
    ParameterKey=Queue3Name,ParameterValue=bitget-filled-orders \
    ParameterKey=Queue4Name,ParameterValue=bingx-orders \
    ParameterKey=Queue5Name,ParameterValue=bingx-filled-orders \
    ParameterKey=Queue6Name,ParameterValue=okx-orders \
    ParameterKey=Queue7Name,ParameterValue=okx-filled-orders \
    ParameterKey=Queue8Name,ParameterValue=order-execution-polling \
    ParameterKey=Queue9Name,ParameterValue=clients-events

# Secrets
aws cloudformation create-stack \
  --stack-name sentry-secrets-dev \
  --template-body file://templates/secrets/secrets-manager.json \
  --parameters ParameterKey=Environment,ParameterValue=dev
```

### Layer 1 - Data

```bash
# Redis (aguardar VPC CREATE_COMPLETE)
aws cloudformation create-stack \
  --stack-name sentry-redis-dev \
  --template-body file://templates/data/elasticache-redis.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=VPCStackName,ParameterValue=sentry-vpc-dev

# ALB
aws cloudformation create-stack \
  --stack-name sentry-alb-dev \
  --template-body file://templates/networking/alb-frontend.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=VPCStackName,ParameterValue=sentry-vpc-dev
```

### Layer 2 - Compute

```bash
# Cluster Gateway
aws cloudformation create-stack \
  --stack-name sentry-cluster-gateway-dev \
  --template-body file://templates/compute/ecs-cluster-gateway.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=VPCStackName,ParameterValue=sentry-vpc-dev \
    ParameterKey=SQSQueueNames,ParameterValue="master-order-queue-dev.fifo,clients-events-queue-dev" \
    ParameterKey=DynamoDBTableNames,ParameterValue="oms_config_dev,oms_trading_data_dev" \
    ParameterKey=ALBSecurityGroupId,ParameterValue=sg-xxx \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM

# Cluster OMS
aws cloudformation create-stack \
  --stack-name sentry-cluster-oms-dev \
  --template-body file://templates/compute/ecs-cluster-oms.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=VPCStackName,ParameterValue=sentry-vpc-dev \
    ParameterKey=SQSQueueNames,ParameterValue="master-order-queue-dev.fifo,bitget-orders-queue-dev.fifo" \
    ParameterKey=DynamoDBTableNames,ParameterValue="oms_config_dev,oms_trading_data_dev" \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM
```

### Layer 3 - Services

```bash
# Task Frontend
aws cloudformation create-stack \
  --stack-name sentry-task-frontend-dev \
  --template-body file://templates/compute/task-definition-frontend-admin.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=ECSClusterStackName,ParameterValue=sentry-cluster-gateway-dev \
    ParameterKey=ALBStackName,ParameterValue=sentry-alb-dev \
    ParameterKey=ECRRepositoryUri,ParameterValue=123456789.dkr.ecr.us-east-1.amazonaws.com/frontend-admin \
    ParameterKey=SecretsArn,ParameterValue=arn:aws:secretsmanager:us-east-1:123456789:secret:sentry/dev

# Task OMS
aws cloudformation create-stack \
  --stack-name sentry-task-oms-dev \
  --template-body file://templates/compute/task-definition-oms.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=ECSClusterStackName,ParameterValue=sentry-cluster-oms-dev \
    ParameterKey=ECRRepositoryUri,ParameterValue=123456789.dkr.ecr.us-east-1.amazonaws.com/oms-service \
    ParameterKey=AppSecretArn,ParameterValue=arn:aws:secretsmanager:us-east-1:123456789:secret:sentry/dev
```

### Layer 4 - CI/CD

```bash
# Pipeline Frontend
aws cloudformation create-stack \
  --stack-name sentry-pipeline-frontend-dev \
  --template-body file://templates/cicd/codepipeline-frontend.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=ECRRepositoryName,ParameterValue=sentry/admin-front-end-dev \
    ParameterKey=CodeStarConnectionArn,ParameterValue=arn:aws:codestar-connections:us-east-1:123456789:connection/xxx \
    ParameterKey=GitHubRepoFullName,ParameterValue=org/oms-spider \
    ParameterKey=GitHubBranch,ParameterValue=main \
    ParameterKey=CognitoRegion,ParameterValue=us-east-1 \
    ParameterKey=CognitoUserPoolId,ParameterValue=us-east-1_xxx \
    ParameterKey=CognitoClientId,ParameterValue=xxx \
    ParameterKey=ECSClusterName,ParameterValue=sentry-gateway-dev \
    ParameterKey=ECSServiceName,ParameterValue=sentry-frontend-admin-service-dev \
  --capabilities CAPABILITY_NAMED_IAM

# Pipeline OMS
aws cloudformation create-stack \
  --stack-name sentry-pipeline-oms-dev \
  --template-body file://templates/cicd/codepipeline-oms.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=ECRRepositoryName,ParameterValue=sentry/oms-dev \
    ParameterKey=CodeStarConnectionArn,ParameterValue=arn:aws:codestar-connections:us-east-1:123456789:connection/xxx \
    ParameterKey=GitHubRepoFullName,ParameterValue=org/oms-spider \
    ParameterKey=GitHubBranch,ParameterValue=main \
    ParameterKey=ECSClusterName,ParameterValue=sentry-oms-dev \
    ParameterKey=ECSServiceName,ParameterValue=sentry-oms-service-dev \
  --capabilities CAPABILITY_NAMED_IAM

# Pipeline WebSocket
aws cloudformation create-stack \
  --stack-name sentry-pipeline-websocket-dev \
  --template-body file://templates/cicd/codepipeline-websocket.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=ECRRepositoryName,ParameterValue=sentry/websocket-dev \
    ParameterKey=CodeStarConnectionArn,ParameterValue=arn:aws:codestar-connections:us-east-1:123456789:connection/xxx \
    ParameterKey=GitHubRepoFullName,ParameterValue=org/oms-spider \
    ParameterKey=GitHubBranch,ParameterValue=main \
    ParameterKey=ECSClusterName,ParameterValue=sentry-gateway-dev \
    ParameterKey=ECSServiceName,ParameterValue=sentry-websocket-service-dev \
  --capabilities CAPABILITY_NAMED_IAM
```

## Deletar

Deletar na ordem inversa (Layer 4 -> 0):

```bash
# Layer 4 - CI/CD
aws cloudformation delete-stack --stack-name sentry-pipeline-websocket-dev
aws cloudformation delete-stack --stack-name sentry-pipeline-oms-dev
aws cloudformation delete-stack --stack-name sentry-pipeline-frontend-dev

# Layer 3
aws cloudformation delete-stack --stack-name sentry-task-websocket-dev
aws cloudformation delete-stack --stack-name sentry-task-oms-dev
aws cloudformation delete-stack --stack-name sentry-task-frontend-dev

# Layer 2
aws cloudformation delete-stack --stack-name sentry-cluster-oms-dev
aws cloudformation delete-stack --stack-name sentry-cluster-gateway-dev

# Layer 1
aws cloudformation delete-stack --stack-name sentry-alb-dev
aws cloudformation delete-stack --stack-name sentry-redis-dev

# Layer 0
aws cloudformation delete-stack --stack-name sentry-secrets-dev
aws cloudformation delete-stack --stack-name sentry-sqs-dev
aws cloudformation delete-stack --stack-name sentry-dynamodb-trading-dev
aws cloudformation delete-stack --stack-name sentry-dynamodb-config-dev
aws cloudformation delete-stack --stack-name sentry-cognito-dev
aws cloudformation delete-stack --stack-name sentry-vpc-dev
```

## Estrutura

```
cloudformation/
├── README.md
├── .gitignore
├── .variables              # Exemplo de variaveis
├── templates/              # Templates CloudFormation
│   ├── networking/
│   ├── cognito/
│   ├── dynamodb/
│   ├── messaging/
│   ├── data/
│   ├── secrets/
│   ├── compute/
│   └── cicd/
└── docs/                   # Documentacao
    ├── DEPLOY.md           # Guia de deploy
    ├── CHANGESET.md        # Guia de change sets
    └── ...
```

## Ambientes

| Ambiente | VPC CIDR | Caracteristicas |
|----------|----------|-----------------|
| dev | 172.22.0.0/16 | Single-AZ, recursos minimos |
| staging | 172.22.0.0/16 | Multi-AZ opcional |
| prod | 10.x.x.x/16 | Multi-AZ, PITR, backups |
