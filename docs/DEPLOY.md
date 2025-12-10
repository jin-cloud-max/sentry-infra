# Guia de Deploy - Sentry OMS

## Ordem de Criação das Stacks

As stacks devem ser criadas na seguinte ordem devido às dependências entre elas:

```
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 0 - Foundation                         │
│  (Sem dependências - podem ser criadas em paralelo)             │
├─────────────────────────────────────────────────────────────────┤
│  1. vpc              │ VPC, Subnets, NAT Gateway, IGW           │
│  2. cognito          │ User Pool, Groups, App Client            │
│  3. dynamodb-config  │ Tabela de configurações                  │
│  4. dynamodb-trading │ Tabela de trading                        │
│  5. sqs              │ Filas SQS (FIFO e Standard)              │
│  6. secrets          │ Secrets Manager                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 1 - Data                               │
│  (Depende da VPC)                                               │
├─────────────────────────────────────────────────────────────────┤
│  7. redis            │ ElastiCache Redis                        │
│  8. alb              │ Application Load Balancer                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 2 - Compute                            │
│  (Depende da VPC, SQS, DynamoDB)                                │
├─────────────────────────────────────────────────────────────────┤
│  9.  cluster-gateway │ ECS Cluster para Frontend/WebSocket      │
│  10. cluster-oms     │ ECS Cluster para OMS                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 3 - Services                           │
│  (Depende dos Clusters, ALB, Cognito)                           │
├─────────────────────────────────────────────────────────────────┤
│  11. task-frontend   │ Frontend Admin Service                   │
│  12. task-oms        │ OMS Service                              │
│  13. task-websocket  │ WebSocket Service                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 4 - CI/CD (Opcional)                   │
├─────────────────────────────────────────────────────────────────┤
│  14. pipeline-*      │ CodePipeline para cada serviço           │
└─────────────────────────────────────────────────────────────────┘
```

## Mapeamento de Templates

| Stack Name Pattern          | Template                                    |
|-----------------------------|---------------------------------------------|
| `sentry-vpc-{env}`          | `templates/networking/vpc-nonprod.json`     |
| `sentry-cognito-{env}`      | `templates/cognito/cognito-user-pool.json`  |
| `sentry-dynamodb-config-{env}` | `templates/dynamodb/oms-config-table.json` |
| `sentry-dynamodb-trading-{env}` | `templates/dynamodb/oms-trading-table.json` |
| `sentry-sqs-{env}`          | `templates/messaging/sqs-queues.json`       |
| `sentry-secrets-{env}`      | `templates/secrets/secrets-manager.json`    |
| `sentry-redis-{env}`        | `templates/data/elasticache-redis.json`     |
| `sentry-alb-{env}`          | `templates/networking/alb-frontend.json`    |
| `sentry-cluster-gateway-{env}` | `templates/compute/ecs-cluster-gateway.json` |
| `sentry-cluster-oms-{env}`  | `templates/compute/ecs-cluster-oms.json`    |
| `sentry-task-frontend-{env}` | `templates/compute/task-definition-frontend-admin.json` |
| `sentry-task-oms-{env}`     | `templates/compute/task-definition-oms.json` |
| `sentry-task-websocket-{env}` | `templates/compute/task-definition-websocket.json` |
| `sentry-pipeline-*-{env}`   | `templates/cicd/codepipeline-codestar.json` |

## Deploy Rápido

### Usando o Script de Deploy

```bash
# Deploy completo de um ambiente
./deploy.sh dev

# Deploy de uma stack específica
./deploy.sh dev vpc

# Listar stacks disponíveis
./deploy.sh --list
```

### Deploy Manual (Stack Individual)

```bash
# Variáveis
ENV=dev
REGION=us-east-1
PROJECT=sentry

# 1. VPC
aws cloudformation create-stack \
  --stack-name ${PROJECT}-vpc-${ENV} \
  --template-body file://templates/networking/vpc-nonprod.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=${ENV} \
    ParameterKey=AvailabilityZone1,ParameterValue=${REGION}a \
    ParameterKey=AvailabilityZone2,ParameterValue=${REGION}b \
  --region ${REGION}
```

## Arquivos de Parâmetros

Para facilitar o deploy, use arquivos de parâmetros JSON:

```bash
# Deploy usando arquivo de parâmetros
aws cloudformation create-stack \
  --stack-name sentry-vpc-dev \
  --template-body file://templates/networking/vpc-nonprod.json \
  --parameters file://params/dev/vpc.json \
  --region us-east-1
```

### Estrutura de Parâmetros

```
params/
├── dev/
│   ├── vpc.json
│   ├── cognito.json
│   ├── dynamodb-config.json
│   ├── dynamodb-trading.json
│   ├── sqs.json
│   ├── secrets.json
│   ├── redis.json
│   ├── alb.json
│   ├── cluster-gateway.json
│   ├── cluster-oms.json
│   ├── task-frontend.json
│   ├── task-oms.json
│   └── task-websocket.json
├── staging/
│   └── ... (mesmos arquivos)
└── prod/
    └── ... (mesmos arquivos)
```

## Verificar Status

```bash
# Status de uma stack
aws cloudformation describe-stacks --stack-name sentry-vpc-dev --query 'Stacks[0].StackStatus'

# Eventos de uma stack (para debug)
aws cloudformation describe-stack-events --stack-name sentry-vpc-dev --query 'StackEvents[0:5]'

# Listar todas as stacks do projeto
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --query "StackSummaries[?contains(StackName, 'sentry')]"
```

## Atualizar Stack Existente

Para atualizar uma stack, use `update-stack` ao invés de `create-stack`:

```bash
aws cloudformation update-stack \
  --stack-name sentry-vpc-dev \
  --template-body file://templates/networking/vpc-nonprod.json \
  --parameters file://params/dev/vpc.json
```

Ou use Change Sets para revisar as mudanças antes de aplicar (veja [CHANGESET.md](CHANGESET.md)).

## Deletar Ambiente

Para deletar um ambiente completo, delete as stacks na **ordem inversa**:

```bash
# Layer 4 - CI/CD
aws cloudformation delete-stack --stack-name sentry-pipeline-frontend-dev

# Layer 3 - Services
aws cloudformation delete-stack --stack-name sentry-task-websocket-dev
aws cloudformation delete-stack --stack-name sentry-task-oms-dev
aws cloudformation delete-stack --stack-name sentry-task-frontend-dev

# Layer 2 - Compute
aws cloudformation delete-stack --stack-name sentry-cluster-oms-dev
aws cloudformation delete-stack --stack-name sentry-cluster-gateway-dev

# Layer 1 - Data
aws cloudformation delete-stack --stack-name sentry-alb-dev
aws cloudformation delete-stack --stack-name sentry-redis-dev

# Layer 0 - Foundation (cuidado: dados serão perdidos!)
aws cloudformation delete-stack --stack-name sentry-secrets-dev
aws cloudformation delete-stack --stack-name sentry-sqs-dev
aws cloudformation delete-stack --stack-name sentry-dynamodb-trading-dev
aws cloudformation delete-stack --stack-name sentry-dynamodb-config-dev
aws cloudformation delete-stack --stack-name sentry-cognito-dev
aws cloudformation delete-stack --stack-name sentry-vpc-dev
```

## Troubleshooting

### Stack em estado ROLLBACK_COMPLETE

```bash
# Deletar e recriar
aws cloudformation delete-stack --stack-name sentry-vpc-dev
aws cloudformation wait stack-delete-complete --stack-name sentry-vpc-dev
# Então recriar...
```

### Erro de Dependência Circular

Se receber erro de dependência, verifique a ordem de criação no diagrama acima.

### Recursos Órfãos

Alguns recursos têm `DeletionPolicy: Retain` (Secrets, DynamoDB). Para deletar completamente:

```bash
# Listar recursos retidos
aws cloudformation describe-stack-resources --stack-name sentry-secrets-dev

# Deletar manualmente se necessário
aws secretsmanager delete-secret --secret-id sentry/dev --force-delete-without-recovery
```
