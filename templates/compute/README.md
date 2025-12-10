# ECS Compute Templates - OMS Spider

Templates CloudFormation para criação dos clusters ECS EC2, Task Definitions e Services do OMS Spider.

## 📦 Templates Disponíveis

### ECS Clusters

#### 1. ecs-cluster-oms.json
Cluster dedicado ao processamento de ordens (OMS - Order Management System)

**Responsabilidades:**
- Consumir mensagens das filas SQS
- Processar ordens recebidas via WebSocket
- Enviar ordens para exchanges via NAT Gateway
- Armazenar dados no DynamoDB
- Cache com ElastiCache Redis

**IAM Permissions:**
- ✅ SQS: Consume, Publish, Delete (múltiplas filas)
- ✅ DynamoDB: Full CRUD operations + Query/Scan
- ✅ Secrets Manager: Read secrets
- ✅ CloudWatch: Write logs
- ✅ ElastiCache: Describe clusters

#### 2. ecs-cluster-frontend.json
Cluster dedicado ao Frontend Dashboard e WebSocket Server

**Responsabilidades:**
- Servir dashboard administrativo (React/Next.js)
- WebSocket server para receber eventos das exchanges
- Publicar eventos no SQS
- Autenticação via Cognito
- Consultas ao DynamoDB

**IAM Permissions:**
- ✅ SQS: Publish messages
- ✅ DynamoDB: Read/Write operations
- ✅ Cognito: User authentication and management
- ✅ Secrets Manager: Read secrets
- ✅ CloudWatch: Write logs
- ✅ ElastiCache: Describe clusters

### Task Definitions

#### 3. task-definition-oms.json
Task Definition e Service para o OMS

**Características:**
- ✅ Network mode: bridge (EC2)
- ✅ Container único: oms-service
- ✅ CPU/Memory configuráveis
- ✅ Health check HTTP customizável
- ✅ Secrets vazios (adicione conforme necessário)
- ✅ Environment variables dinâmicas
- ✅ CloudWatch Logs integration
- ✅ Deployment circuit breaker
- ✅ Placement strategies (spread por AZ e instance)

#### 4. task-definition-frontend.json
Task Definition e Service para Frontend/WebSocket

**Características:**
- ✅ Network mode: bridge (EC2)
- ✅ Multi-container: frontend + websocket
- ✅ Container linking (frontend → websocket)
- ✅ Dependency ordering (websocket inicia primeiro)
- ✅ CPU/Memory configuráveis por container
- ✅ Health checks HTTP em ambos containers
- ✅ Secrets vazios (adicione conforme necessário)
- ✅ Environment variables dinâmicas
- ✅ CloudWatch Logs integration (streams separados)
- ✅ Deployment circuit breaker

## 🚀 Deploy

### Pré-requisitos

Antes de criar os clusters ECS, você precisa ter:

1. ✅ VPC criada (templates/networking/vpc-*.json)
2. ✅ ECR repositories criados
3. ✅ Imagens Docker pushadas no ECR
4. ✅ Key Pair criado (opcional, para SSH)

### Deploy - OMS Cluster (Dev)

```bash
aws cloudformation create-stack \
  --stack-name oms-cluster-dev \
  --template-body file://ecs-cluster-oms.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=VPCStackName,ParameterValue=oms-vpc-dev \
    ParameterKey=InstanceType,ParameterValue=t3.medium \
    ParameterKey=MinSize,ParameterValue=1 \
    ParameterKey=MaxSize,ParameterValue=3 \
    ParameterKey=DesiredCapacity,ParameterValue=1 \
    ParameterKey=SQSQueueNames,ParameterValue="orders-queue-dev,events-queue-dev" \
    ParameterKey=DynamoDBTableNames,ParameterValue="orders-dev,positions-dev,trades-dev" \
    ParameterKey=KeyPairName,ParameterValue=my-keypair \
  --capabilities CAPABILITY_NAMED_IAM
```

### Deploy - OMS Cluster (Production)

```bash
aws cloudformation create-stack \
  --stack-name oms-cluster-prod \
  --template-body file://ecs-cluster-oms.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=prod \
    ParameterKey=VPCStackName,ParameterValue=oms-vpc-prod \
    ParameterKey=InstanceType,ParameterValue=c5.xlarge \
    ParameterKey=MinSize,ParameterValue=2 \
    ParameterKey=MaxSize,ParameterValue=6 \
    ParameterKey=DesiredCapacity,ParameterValue=3 \
    ParameterKey=SQSQueueNames,ParameterValue="orders-queue-prod,events-queue-prod,dlq-prod" \
    ParameterKey=DynamoDBTableNames,ParameterValue="orders-prod,positions-prod,trades-prod" \
    ParameterKey=KeyPairName,ParameterValue=prod-keypair \
  --capabilities CAPABILITY_NAMED_IAM
```

### Deploy - Frontend Cluster (Dev)

```bash
aws cloudformation create-stack \
  --stack-name frontend-cluster-dev \
  --template-body file://ecs-cluster-frontend.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=VPCStackName,ParameterValue=oms-vpc-dev \
    ParameterKey=InstanceType,ParameterValue=t3.medium \
    ParameterKey=MinSize,ParameterValue=1 \
    ParameterKey=MaxSize,ParameterValue=2 \
    ParameterKey=DesiredCapacity,ParameterValue=1 \
    ParameterKey=SQSQueueNames,ParameterValue="websocket-events-dev,orders-queue-dev" \
    ParameterKey=DynamoDBTableNames,ParameterValue="orders-dev,users-dev" \
    ParameterKey=CognitoUserPoolId,ParameterValue=us-east-1_XXXXXXXXX \
    ParameterKey=KeyPairName,ParameterValue=my-keypair \
  --capabilities CAPABILITY_NAMED_IAM
```

### Deploy - Frontend Cluster (Production)

```bash
aws cloudformation create-stack \
  --stack-name frontend-cluster-prod \
  --template-body file://ecs-cluster-frontend.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=prod \
    ParameterKey=VPCStackName,ParameterValue=oms-vpc-prod \
    ParameterKey=InstanceType,ParameterValue=c5.large \
    ParameterKey=MinSize,ParameterValue=2 \
    ParameterKey=MaxSize,ParameterValue=4 \
    ParameterKey=DesiredCapacity,ParameterValue=2 \
    ParameterKey=SQSQueueNames,ParameterValue="websocket-events-prod,orders-queue-prod" \
    ParameterKey=DynamoDBTableNames,ParameterValue="orders-prod,users-prod" \
    ParameterKey=CognitoUserPoolId,ParameterValue=us-east-1_XXXXXXXXX \
    ParameterKey=KeyPairName,ParameterValue=prod-keypair \
  --capabilities CAPABILITY_NAMED_IAM
```

### Deploy - OMS Task Definition (Dev)

```bash
aws cloudformation create-stack \
  --stack-name oms-task-dev \
  --template-body file://task-definition-oms.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=ECSClusterStackName,ParameterValue=oms-cluster-dev \
    ParameterKey=ECRRepositoryUri,ParameterValue=123456789.dkr.ecr.us-east-1.amazonaws.com/oms-service \
    ParameterKey=ImageTag,ParameterValue=latest \
    ParameterKey=ContainerCpu,ParameterValue=512 \
    ParameterKey=ContainerMemory,ParameterValue=1024 \
    ParameterKey=ContainerMemoryReservation,ParameterValue=512 \
    ParameterKey=ContainerPort,ParameterValue=3000 \
    ParameterKey=DesiredCount,ParameterValue=1 \
    ParameterKey=HealthCheckPath,ParameterValue=/health \
  --capabilities CAPABILITY_NAMED_IAM
```

### Deploy - OMS Task Definition (Production)

```bash
aws cloudformation create-stack \
  --stack-name oms-task-prod \
  --template-body file://task-definition-oms.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=prod \
    ParameterKey=ECSClusterStackName,ParameterValue=oms-cluster-prod \
    ParameterKey=ECRRepositoryUri,ParameterValue=123456789.dkr.ecr.us-east-1.amazonaws.com/oms-service \
    ParameterKey=ImageTag,ParameterValue=v1.0.0 \
    ParameterKey=ContainerCpu,ParameterValue=1024 \
    ParameterKey=ContainerMemory,ParameterValue=2048 \
    ParameterKey=ContainerMemoryReservation,ParameterValue=1024 \
    ParameterKey=ContainerPort,ParameterValue=3000 \
    ParameterKey=DesiredCount,ParameterValue=3 \
    ParameterKey=HealthCheckPath,ParameterValue=/health \
  --capabilities CAPABILITY_NAMED_IAM
```

### Deploy - Frontend Task Definition (Dev)

```bash
aws cloudformation create-stack \
  --stack-name frontend-task-dev \
  --template-body file://task-definition-frontend.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=ECSClusterStackName,ParameterValue=frontend-cluster-dev \
    ParameterKey=ECRRepositoryUriFrontend,ParameterValue=123456789.dkr.ecr.us-east-1.amazonaws.com/frontend \
    ParameterKey=ECRRepositoryUriWebSocket,ParameterValue=123456789.dkr.ecr.us-east-1.amazonaws.com/websocket \
    ParameterKey=ImageTagFrontend,ParameterValue=latest \
    ParameterKey=ImageTagWebSocket,ParameterValue=latest \
    ParameterKey=ContainerCpu,ParameterValue=512 \
    ParameterKey=ContainerMemory,ParameterValue=1024 \
    ParameterKey=FrontendMemoryReservation,ParameterValue=384 \
    ParameterKey=WebSocketMemoryReservation,ParameterValue=384 \
    ParameterKey=FrontendPort,ParameterValue=3000 \
    ParameterKey=WebSocketPort,ParameterValue=8080 \
    ParameterKey=DesiredCount,ParameterValue=1 \
  --capabilities CAPABILITY_NAMED_IAM
```

### Deploy - Frontend Task Definition (Production)

```bash
aws cloudformation create-stack \
  --stack-name frontend-task-prod \
  --template-body file://task-definition-frontend.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=prod \
    ParameterKey=ECSClusterStackName,ParameterValue=frontend-cluster-prod \
    ParameterKey=ECRRepositoryUriFrontend,ParameterValue=123456789.dkr.ecr.us-east-1.amazonaws.com/frontend \
    ParameterKey=ECRRepositoryUriWebSocket,ParameterValue=123456789.dkr.ecr.us-east-1.amazonaws.com/websocket \
    ParameterKey=ImageTagFrontend,ParameterValue=v1.0.0 \
    ParameterKey=ImageTagWebSocket,ParameterValue=v1.0.0 \
    ParameterKey=ContainerCpu,ParameterValue=1024 \
    ParameterKey=ContainerMemory,ParameterValue=2048 \
    ParameterKey=FrontendMemoryReservation,ParameterValue=768 \
    ParameterKey=WebSocketMemoryReservation,ParameterValue=768 \
    ParameterKey=FrontendPort,ParameterValue=3000 \
    ParameterKey=WebSocketPort,ParameterValue=8080 \
    ParameterKey=DesiredCount,ParameterValue=2 \
  --capabilities CAPABILITY_NAMED_IAM
```

## 📊 Outputs Importantes

Após o deploy, os seguintes outputs estarão disponíveis:

### ECS Clusters (ecs-cluster-*.json)

**Cluster Information:**
- `ClusterId` - ID do cluster ECS
- `ClusterName` - Nome do cluster ECS

**Networking:**
- `SecurityGroupId` - Security Group do cluster

**IAM Roles:**
- `TaskExecutionRoleArn` - Role para execução de tasks (pull images, secrets)
- `TaskRoleArn` - Role para aplicação (SQS, DynamoDB, etc)

**Auto Scaling:**
- `AutoScalingGroupName` - Nome do ASG
- `CapacityProviderName` - Nome do Capacity Provider

**Logs:**
- `LogGroupName` - CloudWatch Log Group

### Task Definitions (task-definition-*.json)

**Task Definition:**
- `TaskDefinitionArn` - ARN da Task Definition
- `TaskDefinitionFamily` - Family name da Task Definition

**Service:**
- `ServiceName` - Nome do ECS Service
- `ServiceArn` - ARN do ECS Service

**Adicional (Frontend):**
- `FrontendPort` - Porta do container Frontend
- `WebSocketPort` - Porta do container WebSocket

### Obter Outputs

```bash
# Ver todos os outputs
aws cloudformation describe-stacks \
  --stack-name oms-cluster-prod \
  --query 'Stacks[0].Outputs' \
  --output table

# Obter ARN da Task Role
aws cloudformation describe-stacks \
  --stack-name oms-cluster-prod \
  --query 'Stacks[0].Outputs[?OutputKey==`TaskRoleArn`].OutputValue' \
  --output text
```

## ⚙️ Configuração Dinâmica

### Instance Types Suportados

#### Compute Optimized (Recomendado para OMS)
- `c5.large`, `c5.xlarge`, `c5.2xlarge`, `c5.4xlarge`
- `c6i.large`, `c6i.xlarge`, `c6i.2xlarge`, `c6i.4xlarge`

#### General Purpose (Balanceado)
- `t3.micro`, `t3.small`, `t3.medium`, `t3.large`, `t3.xlarge`, `t3.2xlarge`
- `m5.large`, `m5.xlarge`, `m5.2xlarge`, `m5.4xlarge`

#### Memory Optimized (Para cache intensivo)
- `r5.large`, `r5.xlarge`, `r5.2xlarge`

### Variáveis de Ambiente e Secrets

As tasks podem acessar secrets dinamicamente do Secrets Manager:

```bash
# Padrão de naming: ${ProjectName}/${Environment}/*

# Exemplos de secrets a criar:
aws secretsmanager create-secret \
  --name oms-spider/prod/bitget-api-key \
  --secret-string '{"apiKey":"xxx","apiSecret":"yyy"}'

aws secretsmanager create-secret \
  --name oms-spider/prod/okx-api-key \
  --secret-string '{"apiKey":"xxx","apiSecret":"yyy","passphrase":"zzz"}'

aws secretsmanager create-secret \
  --name oms-spider/prod/bingx-api-key \
  --secret-string '{"apiKey":"xxx","apiSecret":"yyy"}'

aws secretsmanager create-secret \
  --name oms-spider/prod/redis-endpoint \
  --secret-string '{"host":"xxx.cache.amazonaws.com","port":"6379"}'
```

### Adicionando Secrets às Task Definitions

Os templates vêm com o array `Secrets` vazio. Para adicionar secrets, edite o template e adicione na seção `ContainerDefinitions`:

```json
{
  "ContainerDefinitions": [
    {
      "Name": "oms-service",
      "Secrets": [
        {
          "Name": "BITGET_API_KEY",
          "ValueFrom": "oms-spider/prod/bitget-api-key:apiKey::"
        },
        {
          "Name": "BITGET_API_SECRET",
          "ValueFrom": "oms-spider/prod/bitget-api-key:apiSecret::"
        },
        {
          "Name": "REDIS_HOST",
          "ValueFrom": "oms-spider/prod/redis-endpoint:host::"
        }
      ]
    }
  ]
}
```

Depois atualize a stack:
```bash
aws cloudformation update-stack \
  --stack-name oms-task-prod \
  --template-body file://task-definition-oms.json \
  --use-previous-parameters \
  --capabilities CAPABILITY_NAMED_IAM
```

### Múltiplas Filas SQS

Você pode especificar múltiplas filas separadas por vírgula:

```bash
ParameterKey=SQSQueueNames,ParameterValue="orders-queue,events-queue,dlq-queue"
```

A Task Role terá permissões em todas as filas.

### Múltiplas Tabelas DynamoDB

Você pode especificar múltiplas tabelas separadas por vírgula:

```bash
ParameterKey=DynamoDBTableNames,ParameterValue="orders,positions,trades,users"
```

A Task Role terá permissões em todas as tabelas e seus GSIs.

## 🔒 Segurança

### Security Groups

#### OMS Cluster
- **Inbound:**
  - ECS Dynamic Port Range (32768-65535) - Apenas da VPC
- **Outbound:**
  - Permite todo tráfego (necessário para exchanges via NAT)

#### Frontend Cluster
- **Inbound:**
  - ECS Dynamic Port Range (32768-65535) - Apenas da VPC
  - Port 3000 (Frontend App) - Apenas da VPC
  - Port 8080 (WebSocket) - Apenas da VPC
- **Outbound:**
  - Permite todo tráfego

### IMDSv2 Enforcement

Ambos os templates forçam o uso do IMDSv2:
```json
"MetadataOptions": {
  "HttpTokens": "required",
  "HttpPutResponseHopLimit": 1
}
```

### IAM Least Privilege

- **Task Execution Role**: Apenas pull de imagens e leitura de secrets
- **Task Role**: Apenas recursos específicos (SQS, DynamoDB)
- **Instance Role**: Apenas permissões do ECS agent

## 📈 Auto Scaling

### Capacity Provider

Ambos os clusters usam Capacity Provider com managed scaling:

- **Target Capacity**: 80% (mantém 20% de buffer)
- **Min Step Size**: 1 instância por vez
- **Max Step Size**: 2 instâncias (OMS) ou 2 instâncias (Frontend)

### Recomendações por Ambiente

#### Development
- Min: 1, Max: 2, Desired: 1
- Instance Type: t3.medium
- Single-AZ permitido

#### Staging
- Min: 1, Max: 3, Desired: 2
- Instance Type: t3.large ou c5.large
- Multi-AZ recomendado

#### Production
- Min: 2, Max: 6, Desired: 3
- Instance Type: c5.xlarge (OMS) ou c5.large (Frontend)
- Multi-AZ obrigatório
- Managed Termination Protection: ENABLED

## 📝 CloudWatch Logs

### Retenção por Ambiente
- **Dev**: 7 dias
- **Staging**: 14 dias
- **Prod**: 30 dias

### Container Insights
- **Dev/Staging**: Disabled (economia de custos)
- **Production**: Enabled (métricas detalhadas)

### Acessar Logs

```bash
# Listar log streams
aws logs describe-log-streams \
  --log-group-name /ecs/oms-spider-oms-prod \
  --order-by LastEventTime \
  --descending

# Ver logs em tempo real
aws logs tail /ecs/oms-spider-oms-prod --follow
```

## 💰 Custos Estimados (Mensais)

### OMS Cluster - Development
- 1x t3.medium (730h): ~$30
- EBS gp3 (30GB): ~$3
- CloudWatch Logs (7 dias): ~$2
- **Total**: ~$35/mês

### OMS Cluster - Production
- 3x c5.xlarge (730h): ~$450
- EBS gp3 (100GB): ~$10
- CloudWatch Logs (30 dias): ~$8
- Container Insights: ~$5
- **Total**: ~$473/mês

### Frontend Cluster - Development
- 1x t3.medium (730h): ~$30
- EBS gp3 (30GB): ~$3
- CloudWatch Logs (7 dias): ~$2
- **Total**: ~$35/mês

### Frontend Cluster - Production
- 2x c5.large (730h): ~$150
- EBS gp3 (60GB): ~$6
- CloudWatch Logs (30 dias): ~$5
- Container Insights: ~$5
- **Total**: ~$166/mês

## 🔧 Troubleshooting

### Instâncias não aparecem no cluster

```bash
# Verificar se instâncias estão registradas
aws ecs list-container-instances --cluster oms-spider-oms-prod

# SSH na instância e verificar logs
sudo cat /var/log/ecs/ecs-agent.log
sudo cat /etc/ecs/ecs.config

# Verificar se ECS agent está rodando
sudo systemctl status ecs
```

### Tasks não iniciam

```bash
# Verificar eventos do cluster
aws ecs describe-clusters --clusters oms-spider-oms-prod

# Verificar service events
aws ecs describe-services \
  --cluster oms-spider-oms-prod \
  --services your-service-name

# Verificar IAM permissions
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::ACCOUNT:role/oms-spider-oms-task-role-prod \
  --action-names sqs:ReceiveMessage \
  --resource-arns arn:aws:sqs:us-east-1:ACCOUNT:orders-queue-prod
```

### Sem conectividade com exchanges

```bash
# SSH na instância e teste de dentro do container
docker exec -it <container_id> sh
curl -s ifconfig.me  # Deve retornar um dos Elastic IPs da VPC

# Verificar NAT Gateway
aws ec2 describe-nat-gateways \
  --filter "Name=vpc-id,Values=<VPC_ID>" \
  --query 'NatGateways[*].[NatGatewayId,State,NatGatewayAddresses[0].PublicIp]'

# Verificar route tables
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=<VPC_ID>" \
  --query 'RouteTables[*].Routes'
```

### Secrets não são carregados

```bash
# Verificar se o secret existe
aws secretsmanager describe-secret \
  --secret-id oms-spider/prod/bitget-api-key

# Verificar permissões da Task Execution Role
aws iam get-role-policy \
  --role-name oms-spider-oms-task-execution-role-prod \
  --policy-name SecretsManagerAccess

# Ver logs da task
aws logs tail /ecs/oms-spider-oms-prod --follow
```

## 🔄 Updates e Rollbacks

### Update de Instance Type

```bash
aws cloudformation update-stack \
  --stack-name oms-cluster-prod \
  --use-previous-template \
  --parameters \
    ParameterKey=Environment,UsePreviousValue=true \
    ParameterKey=VPCStackName,UsePreviousValue=true \
    ParameterKey=InstanceType,ParameterValue=c5.2xlarge \
    ParameterKey=MinSize,UsePreviousValue=true \
    ParameterKey=MaxSize,UsePreviousValue=true \
    ParameterKey=DesiredCapacity,UsePreviousValue=true \
    ParameterKey=SQSQueueNames,UsePreviousValue=true \
    ParameterKey=DynamoDBTableNames,UsePreviousValue=true \
  --capabilities CAPABILITY_NAMED_IAM
```

### Update de Capacidade

```bash
aws cloudformation update-stack \
  --stack-name oms-cluster-prod \
  --use-previous-template \
  --parameters \
    ParameterKey=Environment,UsePreviousValue=true \
    ParameterKey=VPCStackName,UsePreviousValue=true \
    ParameterKey=InstanceType,UsePreviousValue=true \
    ParameterKey=MinSize,ParameterValue=3 \
    ParameterKey=MaxSize,ParameterValue=10 \
    ParameterKey=DesiredCapacity,ParameterValue=5 \
    ParameterKey=SQSQueueNames,UsePreviousValue=true \
    ParameterKey=DynamoDBTableNames,UsePreviousValue=true \
  --capabilities CAPABILITY_NAMED_IAM
```

### Rollback

```bash
# CloudFormation rollback automático em caso de falha
# Para rollback manual:
aws cloudformation cancel-update-stack --stack-name oms-cluster-prod
```

## 📚 Próximos Passos

Após criar os clusters ECS:

1. ✅ Verificar que instâncias estão registradas no cluster
2. ✅ Criar secrets no Secrets Manager
3. ➡️ Criar Task Definitions (próximo template)
4. ➡️ Criar ECS Services
5. ➡️ Criar Application Load Balancer (para Frontend)
6. ➡️ Configurar Auto Scaling policies

## 📖 Referências

- [Amazon ECS Best Practices](https://docs.aws.amazon.com/AmazonECS/latest/bestpracticesguide/intro.html)
- [ECS Capacity Providers](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cluster-capacity-providers.html)
- [Task IAM Roles](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html)
- [Secrets Manager with ECS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/specifying-sensitive-data-secrets.html)
