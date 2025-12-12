# Sentry - Guia de Deployment CloudFormation

## Visão Geral

Este documento descreve a ordem de deployment dos recursos AWS para o projeto Sentry.
Os templates devem ser deployados na ordem especificada devido às dependências entre eles.

---

## Deploy via Console AWS

### Passo a Passo no Console

1. Acesse **CloudFormation** no Console AWS
2. Clique em **Create stack** → **With new resources (standard)**
3. Selecione **Upload a template file**
4. Faça upload do arquivo `.json` correspondente
5. Preencha os **Parameters** conforme a tabela de cada recurso
6. Em **Capabilities**, marque **I acknowledge that AWS CloudFormation might create IAM resources with custom names** (se necessário)
7. Clique em **Create stack**
8. Aguarde o status mudar para **CREATE_COMPLETE**

### Nomenclatura das Stacks

Use o padrão: `sentry-<recurso>-<ambiente>`

Exemplos:
- `sentry-vpc-dev`
- `sentry-ecs-oms-dev`
- `sentry-redis-prod`

---

## Diagrama de Dependências

```
                                    ┌─────────────────┐
                                    │      VPC        │
                                    │  (vpc-nonprod   │
                                    │   ou vpc-prod)  │
                                    └────────┬────────┘
                                             │
                     ┌───────────────────────┼───────────────────────┐
                     │                       │                       │
                     ▼                       ▼                       ▼
          ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
          │  ECS Cluster OMS │    │ECS Cluster Front │    │ ElastiCache Redis│
          └────────┬─────────┘    └────────┬─────────┘    └──────────────────┘
                   │                       │
                   ▼                       ├──────────────────┐
  ┌────────────────────────────┐           ▼                  ▼
  │ Task Def OMS               │  ┌──────────────────┐ ┌──────────────────┐
  │ (usa Secrets Manager)      │  │Task Def Frontend │ │Task Def WebSocket│
  └────────────────────────────┘  └──────────────────┘ └──────────────────┘
                   ▲
                   │ (opcional)
  ┌────────────────┴───────────┐
  │     Secrets Manager        │
  │  (exchange-credentials)    │
  └────────────────────────────┘

  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
  │    SQS Queues    │    │    DynamoDB      │    │     Cognito      │
  │  (independentes) │    │  (independentes) │    │  (independente)  │
  └──────────────────┘    └──────────────────┘    └──────────────────┘
```

---

## Ordem de Deployment

### Fase 1: Fundação (Sem dependências)

| # | Stack Name | Template | Descrição |
|---|------------|----------|-----------|
| 1.1 | `sentry-vpc-{env}` | `networking/vpc-nonprod.json` ou `vpc-prod.json` | VPC, Subnets, NAT Gateway |
| 1.2 | `sentry-cognito-{env}` | `cognito/cognito-user-pool.json` | Autenticação Admin |
| 1.3 | `sentry-dynamodb-config-{env}` | `dynamodb/oms-config-table.json` | Tabela de configuração |
| 1.4 | `sentry-dynamodb-trading-{env}` | `dynamodb/oms-trading-table.json` | Tabela de trading |
| 1.5 | `sentry-sqs-{queue}-{env}` | `messaging/sqs-queue.json` | Filas SQS (9x) |
| 1.6 | `sentry-secret-{name}-{env}` | `secrets/secrets-manager.json` | Secrets Manager (credenciais) |

### Fase 2: Data Layer (Depende da VPC)

| # | Stack Name | Template | Depende de |
|---|------------|----------|------------|
| 2.1 | `sentry-redis-{env}` | `data/elasticache-redis.json` | VPC |

### Fase 3: Compute (Depende da VPC)

| # | Stack Name | Template | Depende de |
|---|------------|----------|------------|
| 3.1 | `sentry-ecs-oms-{env}` | `compute/ecs-cluster-oms.json` | VPC |
| 3.2 | `sentry-ecs-frontend-{env}` | `compute/ecs-cluster-frontend.json` | VPC |

### Fase 4: Services (Depende dos Clusters)

| # | Stack Name | Template | Depende de |
|---|------------|----------|------------|
| 4.1 | `sentry-task-oms-{env}` | `compute/task-definition-oms.json` | ECS Cluster OMS |
| 4.2 | `sentry-task-frontend-{env}` | `compute/task-definition-frontend-admin.json` | ECS Cluster Frontend |
| 4.3 | `sentry-task-websocket-{env}` | `compute/task-definition-websocket.json` | ECS Cluster Frontend |

### Fase 5: CI/CD (Opcional)

| # | Stack Name | Template | Depende de |
|---|------------|----------|------------|
| 5.1 | `sentry-pipeline-{env}` | `cicd/codepipeline-codestar.json` | ECR Repository |

---

## Parâmetros por Recurso (Console)

### Fase 1: Fundação

#### 1.1 VPC NonProd (`networking/vpc-nonprod.json`)

| Parâmetro | DEV | STAGING | Descrição |
|-----------|-----|---------|-----------|
| Environment | `dev` | `staging` | Ambiente |
| VpcCIDR | `172.22.0.0/16` | `172.22.0.0/16` | CIDR do VPC |
| EnableFlowLogs | `false` | `true` | Logs de tráfego |

#### 1.1 VPC Prod (`networking/vpc-prod.json`)

| Parâmetro | PROD | Descrição |
|-----------|------|-----------|
| VpcCIDR | `172.23.0.0/16` | CIDR do VPC |
| AvailabilityZone1 | `us-east-1a` | Primeira AZ |
| AvailabilityZone2 | `us-east-1b` | Segunda AZ |
| AvailabilityZone3 | `us-east-1c` | Terceira AZ |

#### 1.2 Cognito (`cognito/cognito-user-pool.json`)

| Parâmetro | Valor | Descrição |
|-----------|-------|-----------|
| Environment | `dev` / `staging` / `prod` | Ambiente |
| AdminEmail | `seu-email@dominio.com` | Email do admin inicial |

> ⚠️ **Capabilities**: Marque "I acknowledge that AWS CloudFormation might create IAM resources"

#### 1.3 DynamoDB Config (`dynamodb/oms-config-table.json`)

| Parâmetro | Valor | Descrição |
|-----------|-------|-----------|
| Environment | `dev` / `staging` / `prod` | Ambiente |
| TableName | `sentry-config` | Nome da tabela |

#### 1.4 DynamoDB Trading (`dynamodb/oms-trading-table.json`)

| Parâmetro | Valor | Descrição |
|-----------|-------|-----------|
| Environment | `dev` / `staging` / `prod` | Ambiente |
| TableName | `sentry-trading` | Nome da tabela |

#### 1.5 SQS Queue (`messaging/sqs-queue.json`)

Crie uma stack para cada fila. Exemplos:

**Fila Standard:**
| Parâmetro | Valor | Descrição |
|-----------|-------|-----------|
| Environment | `dev` | Ambiente |
| QueueName | `orders-bitget` | Nome da fila |
| QueueType | `standard` | Tipo da fila |
| EnableDLQ | `true` | Criar Dead Letter Queue |

**Fila FIFO:**
| Parâmetro | Valor | Descrição |
|-----------|-------|-----------|
| Environment | `dev` | Ambiente |
| QueueName | `executions` | Nome da fila |
| QueueType | `fifo` | Tipo FIFO (ordem garantida) |
| EnableDLQ | `true` | Criar Dead Letter Queue |

#### 1.6 Secrets Manager (`secrets/secrets-manager.json`)

Crie uma stack para cada tipo de secret. Os secrets suportados:

**Exchange Credentials (para OMS):**
| Parâmetro | Valor | Descrição |
|-----------|-------|-----------|
| Environment | `dev` | Ambiente |
| SecretName | `exchange-credentials` | Nome do secret |
| SecretDescription | `Credenciais da exchange para trading` | Descrição |

> Após criar, atualize o secret no console com os valores reais:
> ```json
> {
>   "api_key": "sua-api-key",
>   "api_secret": "seu-api-secret",
>   "passphrase": "sua-passphrase"
> }
> ```

**Database Credentials:**
| Parâmetro | Valor | Descrição |
|-----------|-------|-----------|
| Environment | `dev` | Ambiente |
| SecretName | `database` | Nome do secret |
| SecretDescription | `Credenciais do banco de dados` | Descrição |

> Estrutura esperada do secret:
> ```json
> {
>   "connection_string": "postgresql://user:pass@host:5432/db",
>   "host": "db.example.com",
>   "port": "5432",
>   "database": "sentry",
>   "username": "admin",
>   "password": "senha-segura"
> }
> ```

**Redis Credentials:**
| Parâmetro | Valor | Descrição |
|-----------|-------|-----------|
| Environment | `dev` | Ambiente |
| SecretName | `redis` | Nome do secret |
| SecretDescription | `Credenciais do Redis` | Descrição |

> Estrutura esperada do secret:
> ```json
> {
>   "url": "redis://user:pass@host:6379",
>   "host": "redis.example.com",
>   "port": "6379",
>   "password": "senha-segura"
> }
> ```

> **Importante**: O secret é criado com um placeholder. Você DEVE atualizar manualmente os valores no Console AWS → Secrets Manager → Retrieve secret value → Edit.

---

### Fase 2: Data Layer

#### 2.1 ElastiCache Redis (`data/elasticache-redis.json`)

| Parâmetro | DEV | PROD | Descrição |
|-----------|-----|------|-----------|
| Environment | `dev` | `prod` | Ambiente |
| VPCStackName | `sentry-vpc-dev` | `sentry-vpc-prod` | Stack da VPC |
| NodeType | `cache.t4g.micro` | `cache.r7g.large` | Tipo da instância |
| NumCacheNodes | `1` | `2` | Número de nós |

---

### Fase 3: Compute

#### 3.1 ECS Cluster OMS (`compute/ecs-cluster-oms.json`)

| Parâmetro | DEV | PROD | Descrição |
|-----------|-----|------|-----------|
| Environment | `dev` | `prod` | Ambiente |
| VPCStackName | `sentry-vpc-dev` | `sentry-vpc-prod` | Stack da VPC |
| InstanceType | `t4g.small` | `c7g.large` | Tipo EC2 (ARM64) |
| MinSize | `1` | `2` | Mínimo de instâncias |
| MaxSize | `2` | `8` | Máximo de instâncias |
| DesiredCapacity | `1` | `2` | Capacidade desejada |

> ⚠️ **Capabilities**: Marque "I acknowledge that AWS CloudFormation might create IAM resources"

#### 3.2 ECS Cluster Frontend (`compute/ecs-cluster-frontend.json`)

| Parâmetro | DEV | PROD | Descrição |
|-----------|-----|------|-----------|
| Environment | `dev` | `prod` | Ambiente |
| VPCStackName | `sentry-vpc-dev` | `sentry-vpc-prod` | Stack da VPC |
| InstanceType | `t4g.small` | `t4g.medium` | Tipo EC2 (ARM64) |
| MinSize | `1` | `2` | Mínimo de instâncias |
| MaxSize | `2` | `4` | Máximo de instâncias |
| DesiredCapacity | `1` | `2` | Capacidade desejada |

> ⚠️ **Capabilities**: Marque "I acknowledge that AWS CloudFormation might create IAM resources"

---

### Fase 4: Services

#### 4.1 Task Definition OMS (`compute/task-definition-oms.json`)

| Parâmetro | Valor | Descrição |
|-----------|-------|-----------|
| Environment | `dev` | Ambiente |
| ECSClusterStackName | `sentry-ecs-oms-dev` | Stack do cluster OMS |
| ECRRepositoryUri | `123456789.dkr.ecr.us-east-1.amazonaws.com/sentry-oms` | URI do ECR |
| ImageTag | `latest` | Tag da imagem |
| EnvFileS3Arn | `arn:aws:s3:::sentry-config/oms/dev.env` | Arquivo .env no S3 (opcional) |
| ExchangeCredentialsSecretArn | `arn:aws:secretsmanager:us-east-1:123456789:secret:sentry/dev/exchange-credentials-XXXXXX` | ARN do secret (opcional) |

> **Variáveis de ambiente via Secrets Manager**:
> Se `ExchangeCredentialsSecretArn` for fornecido, as seguintes variáveis serão injetadas automaticamente:
> - `EXCHANGE_API_KEY` - extraído de `api_key`
> - `EXCHANGE_API_SECRET` - extraído de `api_secret`
> - `EXCHANGE_PASSPHRASE` - extraído de `passphrase`
>
> O ARN completo pode ser obtido nos Outputs da stack do Secrets Manager (`sentry-secret-exchange-credentials-dev`).

#### 4.2 Task Definition Frontend (`compute/task-definition-frontend-admin.json`)

| Parâmetro | Valor | Descrição |
|-----------|-------|-----------|
| Environment | `dev` | Ambiente |
| ECSClusterStackName | `sentry-ecs-frontend-dev` | Stack do cluster Frontend |
| ECRRepositoryUri | `123456789.dkr.ecr.us-east-1.amazonaws.com/sentry-frontend` | URI do ECR |
| ImageTag | `latest` | Tag da imagem |
| EnvFileS3Arn | `arn:aws:s3:::sentry-config/frontend/dev.env` | Arquivo .env no S3 (opcional) |

#### 4.3 Task Definition WebSocket (`compute/task-definition-websocket.json`)

| Parâmetro | Valor | Descrição |
|-----------|-------|-----------|
| Environment | `dev` | Ambiente |
| ECSClusterStackName | `sentry-ecs-frontend-dev` | Stack do cluster Frontend |
| ECRRepositoryUri | `123456789.dkr.ecr.us-east-1.amazonaws.com/sentry-websocket` | URI do ECR |
| ImageTag | `latest` | Tag da imagem |
| EnvFileS3Arn | `arn:aws:s3:::sentry-config/websocket/dev.env` | Arquivo .env no S3 (opcional) |

---

### Fase 5: CI/CD

#### 5.1 CodePipeline com CodeStar (`cicd/codepipeline-codestar.json`)

**Pré-requisitos:**
1. ECR Repository já criado (pode criar manualmente no console ECR)
2. CodeStar Connection com GitHub já configurada

**Como criar CodeStar Connection:**
1. Acesse **Developer Tools** → **Settings** → **Connections**
2. Clique em **Create connection**
3. Selecione **GitHub**
4. Autorize acesso ao repositório
5. Copie o ARN da conexão criada

| Parâmetro | Valor | Descrição |
|-----------|-------|-----------|
| Environment | `dev` | Ambiente |
| ProjectName | `sentry` | Nome do projeto |
| ECRRepositoryName | `sentry-oms` | Nome do repositório ECR |
| CodeStarConnectionArn | `arn:aws:codestar-connections:us-east-1:123456789:connection/xxx` | ARN da conexão CodeStar |
| GitHubRepoFullName | `seu-usuario/sentry` | Repositório GitHub (owner/repo) |
| GitHubBranch | `main` | Branch a monitorar |
| BuildspecPath | `buildspec.yml` | Caminho do buildspec |

> ⚠️ **Capabilities**: Marque "I acknowledge that AWS CloudFormation might create IAM resources"

**Criar um pipeline para cada serviço:**

| Stack Name | ECRRepositoryName | GitHubBranch |
|------------|-------------------|--------------|
| `sentry-pipeline-oms-dev` | `sentry-oms` | `main` ou `develop` |
| `sentry-pipeline-frontend-dev` | `sentry-frontend` | `main` ou `develop` |
| `sentry-pipeline-websocket-dev` | `sentry-websocket` | `main` ou `develop` |

**Exemplo de `buildspec.yml` para ARM64:**
```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
  build:
    commands:
      - echo Building Docker image...
      - docker build -t $ECR_REPOSITORY_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION .
      - docker tag $ECR_REPOSITORY_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPOSITORY_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
      - docker tag $ECR_REPOSITORY_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPOSITORY_NAME:latest
  post_build:
    commands:
      - echo Pushing Docker image...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPOSITORY_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPOSITORY_NAME:latest
```

> **Nota**: O CodeBuild está configurado para usar `ARM_CONTAINER` com a imagem `amazonlinux2-aarch64-standard:3.0`, compilando nativamente para ARM64.

---

## Atualização de Stacks

Para atualizar uma stack existente:

```bash
aws cloudformation update-stack \
  --stack-name ${PROJECT}-<recurso>-${ENV} \
  --template-body file://templates/<path>/template.json \
  --parameters \
    ParameterKey=<Param>,ParameterValue=<Value> \
  --capabilities CAPABILITY_NAMED_IAM \
  --region ${AWS_REGION}
```

---

## Deletar Stacks

**IMPORTANTE**: Delete na ordem inversa do deploy!

```bash
# Fase 4 primeiro
aws cloudformation delete-stack --stack-name ${PROJECT}-task-websocket-${ENV}
aws cloudformation delete-stack --stack-name ${PROJECT}-task-frontend-${ENV}
aws cloudformation delete-stack --stack-name ${PROJECT}-task-oms-${ENV}

# Aguardar
aws cloudformation wait stack-delete-complete --stack-name ${PROJECT}-task-oms-${ENV}

# Fase 3
aws cloudformation delete-stack --stack-name ${PROJECT}-ecs-frontend-${ENV}
aws cloudformation delete-stack --stack-name ${PROJECT}-ecs-oms-${ENV}

# Fase 2
aws cloudformation delete-stack --stack-name ${PROJECT}-redis-${ENV}

# Fase 1 (por último)
aws cloudformation delete-stack --stack-name ${PROJECT}-sqs-*-${ENV}
aws cloudformation delete-stack --stack-name ${PROJECT}-dynamodb-*-${ENV}
aws cloudformation delete-stack --stack-name ${PROJECT}-cognito-${ENV}
aws cloudformation delete-stack --stack-name ${PROJECT}-vpc-${ENV}
```

---

## Verificar Status

### Status de uma stack
```bash
aws cloudformation describe-stacks \
  --stack-name ${PROJECT}-vpc-${ENV} \
  --query 'Stacks[0].StackStatus' \
  --output text
```

### Listar todas as stacks do projeto
```bash
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --query "StackSummaries[?starts_with(StackName, '${PROJECT}')].[StackName,StackStatus]" \
  --output table
```

### Ver eventos de erro
```bash
aws cloudformation describe-stack-events \
  --stack-name ${PROJECT}-<recurso>-${ENV} \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`].[LogicalResourceId,ResourceStatusReason]' \
  --output table
```

### Ver outputs de uma stack
```bash
aws cloudformation describe-stacks \
  --stack-name ${PROJECT}-vpc-${ENV} \
  --query 'Stacks[0].Outputs' \
  --output table
```

---

## Checklist por Ambiente

### DEV
- [ ] VPC NonProd
- [ ] Cognito
- [ ] DynamoDB Config
- [ ] DynamoDB Trading
- [ ] SQS Queues (9x)
- [ ] Secrets Manager (exchange-credentials)
- [ ] ElastiCache Redis
- [ ] ECS Cluster OMS
- [ ] ECS Cluster Frontend
- [ ] Task Definition OMS (com ExchangeCredentialsSecretArn)
- [ ] Task Definition Frontend
- [ ] Task Definition WebSocket

### STAGING
- [ ] (mesmos itens)

### PROD
- [ ] VPC Prod (multi-AZ)
- [ ] (demais itens com configurações de produção)

---

## Troubleshooting

### Erro: Stack em ROLLBACK_COMPLETE
```bash
# Delete a stack e tente novamente
aws cloudformation delete-stack --stack-name <stack-name>
aws cloudformation wait stack-delete-complete --stack-name <stack-name>
```

### Erro: Export já existe
Outra stack está usando o mesmo nome de export. Verifique conflitos de nomes.

### Erro: Dependency violation
Você está tentando deletar uma stack que outra depende. Delete na ordem inversa.

### Erro: Capabilities required
Adicione `--capabilities CAPABILITY_NAMED_IAM` ao comando.

---

## Estratégias de Auto Scaling

### Visão Geral

O sistema utiliza duas camadas de auto scaling que trabalham em conjunto:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ESTRATÉGIA DE AUTO SCALING                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. ECS SERVICE AUTO SCALING (Tasks)                                        │
│     ├── Métrica: ECSServiceAverageCPUUtilization                           │
│     ├── Trigger: CPU média das tasks > TargetCPUUtilization (70%)          │
│     ├── Ação: Adiciona/remove tasks                                        │
│     └── Cooldown: 60s (scale out) / 300s (scale in)                        │
│                                                                             │
│  2. CAPACITY PROVIDER MANAGED SCALING (Instâncias)                         │
│     ├── Métrica: Capacidade reservada do cluster                           │
│     ├── Trigger: Capacidade reservada > TargetCapacity (70%)               │
│     ├── Ação: Adiciona/remove instâncias EC2                               │
│     └── Proativo: Mantém 30% de buffer para novas tasks                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Fluxo de Scaling

```
                    ┌──────────────────────┐
                    │   Task CPU > 70%     │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ ECS Service adiciona │
                    │     nova task        │
                    └──────────┬───────────┘
                               │
              ┌────────────────┴────────────────┐
              │                                 │
              ▼                                 ▼
   ┌─────────────────────┐          ┌─────────────────────┐
   │  Cabe nas instâncias│          │ NÃO cabe nas        │
   │     existentes?     │          │ instâncias          │
   └──────────┬──────────┘          └──────────┬──────────┘
              │                                 │
              │ SIM                             │ NÃO
              ▼                                 ▼
   ┌─────────────────────┐          ┌─────────────────────┐
   │  Task é colocada    │          │ Capacity Provider   │
   │  em instância livre │          │ sobe nova instância │
   └─────────────────────┘          └──────────┬──────────┘
                                               │
                                               ▼
                                    ┌─────────────────────┐
                                    │ Task é colocada na  │
                                    │   nova instância    │
                                    └─────────────────────┘
```

### Comportamento por Ambiente

| Ambiente | Task Auto Scaling | Instance Auto Scaling | Motivo |
|----------|-------------------|----------------------|--------|
| **dev** | Desabilitado (default) | Desabilitado | Controle de custos |
| **staging** | Desabilitado (default) | Desabilitado | Controle de custos |
| **prod** | Habilitável via parâmetro | Habilitado automaticamente | Alta disponibilidade |

### Parâmetros de Auto Scaling

#### Task Definition (OMS e WebSocket)

| Parâmetro | Default | Descrição |
|-----------|---------|-----------|
| `EnableServiceAutoScaling` | `false` | Habilita scaling automático de tasks |
| `MinTaskCount` | `1` | Mínimo de tasks (mesmo com baixa CPU) |
| `MaxTaskCount` | `4` | Máximo de tasks (limite de scale up) |
| `TargetCPUUtilization` | `70` | % de CPU que dispara scaling |
| `ScaleOutCooldown` | `60` | Segundos entre scale outs (rápido) |
| `ScaleInCooldown` | `300` | Segundos entre scale ins (conservador) |

#### ECS Cluster (OMS e Gateway)

| Parâmetro | Default | Descrição |
|-----------|---------|-----------|
| `TargetCapacity` | `70` | % de capacidade reservada que dispara scaling |
| `MinimumScalingStepSize` | `1` | Mínimo de instâncias adicionadas por vez |
| `MaximumScalingStepSize` | `2` | Máximo de instâncias adicionadas por vez |

### Configuração Recomendada

#### DEV/STAGING (Custos controlados)

```bash
# Task Definition - Auto Scaling desabilitado
EnableServiceAutoScaling=false
DesiredCount=1

# Cluster - Managed Scaling desabilitado (automático para non-prod)
DesiredCapacity=1
MinSize=1
MaxSize=2
```

#### PROD (Alta disponibilidade)

```bash
# Task Definition - Auto Scaling habilitado
EnableServiceAutoScaling=true
MinTaskCount=2
MaxTaskCount=8
TargetCPUUtilization=70
ScaleOutCooldown=60
ScaleInCooldown=300

# Cluster - Managed Scaling habilitado (automático para prod)
DesiredCapacity=2
MinSize=2
MaxSize=8
# TargetCapacity=70 (configurado no template)
```

### Como Habilitar Auto Scaling

#### Via Console AWS (CloudFormation)

1. Acesse a stack da Task Definition
2. Clique em **Update**
3. Selecione **Use current template**
4. Altere `EnableServiceAutoScaling` para `true`
5. Ajuste `MinTaskCount`, `MaxTaskCount` conforme necessário
6. Complete o update

#### Via CLI

```bash
# Habilitar Auto Scaling para OMS em prod
aws cloudformation update-stack \
  --stack-name sentry-task-oms-prod \
  --use-previous-template \
  --parameters \
    ParameterKey=Environment,UsePreviousValue=true \
    ParameterKey=ECSClusterStackName,UsePreviousValue=true \
    ParameterKey=ECRRepositoryUri,UsePreviousValue=true \
    ParameterKey=AppSecretArn,UsePreviousValue=true \
    ParameterKey=EnableServiceAutoScaling,ParameterValue=true \
    ParameterKey=MinTaskCount,ParameterValue=2 \
    ParameterKey=MaxTaskCount,ParameterValue=8 \
  --capabilities CAPABILITY_NAMED_IAM
```

### Monitoramento do Auto Scaling

#### Métricas CloudWatch

```bash
# CPU média do serviço (dispara task scaling)
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS \
  --metric-name CPUUtilization \
  --dimensions Name=ClusterName,Value=sentry-oms-prod Name=ServiceName,Value=sentry-oms-service-prod \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 \
  --statistics Average

# Capacidade do cluster (dispara instance scaling)
aws ecs describe-clusters \
  --clusters sentry-oms-prod \
  --include STATISTICS \
  --query 'clusters[0].statistics'
```

#### Logs de Scaling

```bash
# Ver eventos de scaling de tasks
aws application-autoscaling describe-scaling-activities \
  --service-namespace ecs \
  --resource-id service/sentry-oms-prod/sentry-oms-service-prod

# Ver eventos de scaling de instâncias
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name sentry-oms-asg-prod \
  --max-items 10
```

### Ajuste Fino do Scaling

#### Scaling mais agressivo (tasks)
```bash
TargetCPUUtilization=50    # Escala mais cedo
ScaleOutCooldown=30        # Escala mais rápido
```

#### Scaling mais conservador (tasks)
```bash
TargetCPUUtilization=80    # Escala mais tarde
ScaleInCooldown=600        # Demora mais para remover tasks
```

#### Scaling mais proativo (instâncias)
```bash
TargetCapacity=60          # Mantém 40% de buffer
```

#### Scaling mais econômico (instâncias)
```bash
TargetCapacity=85          # Mantém apenas 15% de buffer
```

### Troubleshooting Auto Scaling

#### Tasks não estão escalando
1. Verifique se `EnableServiceAutoScaling=true`
2. Confirme que a métrica CPU está acima do threshold
3. Verifique se não está no período de cooldown
4. Consulte os logs de Application Auto Scaling

#### Instâncias não estão escalando (prod)
1. Verifique se o ambiente é `prod` (Managed Scaling só ativa em prod)
2. Confirme que há tasks pendentes que não cabem nas instâncias
3. Verifique os limites de MaxSize no Auto Scaling Group
4. Consulte os eventos do ASG

#### Scaling muito lento
1. Reduza `ScaleOutCooldown` para tasks
2. Reduza `TargetCapacity` para instâncias (mais buffer)
3. Aumente `MaximumScalingStepSize` para instâncias

#### Custos altos inesperados
1. Verifique se `EnableServiceAutoScaling=false` em dev/staging
2. Confirme que o ambiente está correto (prod vs dev)
3. Revise os valores de `MinTaskCount` e `MinSize`

---

## Adicionando Novos Recursos

1. Crie o template em `templates/<categoria>/<nome>.json`
2. Adicione ao diagrama de dependências neste documento
3. Adicione à seção de Ordem de Deployment
4. Adicione os comandos de deploy
5. Atualize o checklist

---

## Estrutura de Arquivos

```
infra/cloudformation/
├── docs/
│   └── DEPLOYMENT.md          # Este arquivo
├── templates/
│   ├── networking/
│   │   ├── vpc-nonprod.json
│   │   └── vpc-prod.json
│   ├── compute/
│   │   ├── ecs-cluster-oms.json
│   │   ├── ecs-cluster-frontend.json
│   │   ├── task-definition-oms.json
│   │   ├── task-definition-frontend-admin.json
│   │   └── task-definition-websocket.json
│   ├── data/
│   │   └── elasticache-redis.json
│   ├── dynamodb/
│   │   ├── oms-config-table.json
│   │   └── oms-trading-table.json
│   ├── messaging/
│   │   └── sqs-queue.json
│   ├── secrets/
│   │   └── secrets-manager.json
│   ├── cognito/
│   │   └── cognito-user-pool.json
│   └── cicd/
│       ├── codepipeline.json
│       └── codepipeline-codestar.json
└── architecture.drawio
```
