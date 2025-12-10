# Configuração de Variáveis de Ambiente via S3

As Task Definitions suportam carregar variáveis de ambiente de arquivos armazenados no S3.

## Como Funciona

1. Crie um arquivo `.env` com suas variáveis
2. Faça upload para o S3
3. Passe o ARN do arquivo como parâmetro na stack da Task Definition
4. O ECS carrega automaticamente as variáveis no container

## Formato do Arquivo .env

```env
# Exemplo de arquivo .env
DATABASE_URL=postgresql://user:password@host:5432/dbname
REDIS_HOST=evo-redis-dev.abc123.0001.use1.cache.amazonaws.com
REDIS_PORT=6379
API_KEY=seu-api-key-aqui
LOG_LEVEL=info
MAX_WORKERS=10
```

**Importante:**
- Uma variável por linha
- Formato: `NOME_VARIAVEL=valor`
- Sem espaços ao redor do `=`
- Sem aspas (elas serão incluídas no valor)
- Comentários começam com `#`

## Upload para S3

### 1. Criar Bucket (se ainda não existir)

```bash
aws s3 mb s3://oms-spider-config-dev --region us-east-1
```

### 2. Upload do arquivo .env

```bash
# Upload do .env para OMS
aws s3 cp oms.env s3://oms-spider-config-dev/oms/dev.env

# Upload do .env para Frontend
aws s3 cp frontend.env s3://oms-spider-config-dev/frontend/dev.env

# Upload do .env para WebSocket
aws s3 cp websocket.env s3://oms-spider-config-dev/websocket/dev.env
```

### 3. Verificar Upload

```bash
aws s3 ls s3://oms-spider-config-dev/oms/
```

## Deploy da Task Definition com Environment File

### OMS Service

```bash
aws cloudformation create-stack \
  --stack-name oms-task-def-dev \
  --template-body file://templates/compute/task-definition-oms.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=ECSClusterStackName,ParameterValue=oms-cluster-dev \
    ParameterKey=ECRRepositoryUri,ParameterValue=123456789.dkr.ecr.us-east-1.amazonaws.com/oms-service \
    ParameterKey=ImageTag,ParameterValue=latest \
    ParameterKey=EnvFileS3Arn,ParameterValue=arn:aws:s3:::oms-spider-config-dev/oms/dev.env \
  --capabilities CAPABILITY_IAM \
  --region us-east-1
```

### Frontend/WebSocket Service

```bash
aws cloudformation create-stack \
  --stack-name frontend-task-def-dev \
  --template-body file://templates/compute/task-definition-frontend.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=ECSClusterStackName,ParameterValue=frontend-cluster-dev \
    ParameterKey=ECRRepositoryUriFrontend,ParameterValue=123456789.dkr.ecr.us-east-1.amazonaws.com/frontend \
    ParameterKey=ECRRepositoryUriWebSocket,ParameterValue=123456789.dkr.ecr.us-east-1.amazonaws.com/websocket \
    ParameterKey=EnvFileS3ArnFrontend,ParameterValue=arn:aws:s3:::oms-spider-config-dev/frontend/dev.env \
    ParameterKey=EnvFileS3ArnWebSocket,ParameterValue=arn:aws:s3:::oms-spider-config-dev/websocket/dev.env \
  --capabilities CAPABILITY_IAM \
  --region us-east-1
```

## Permissões Necessárias

O Task Execution Role precisa ter permissão para ler do S3:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": [
        "arn:aws:s3:::oms-spider-config-dev/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::oms-spider-config-dev"
      ]
    }
  ]
}
```

Essas permissões já estão incluídas no Task Execution Role criado pelos templates dos ECS Clusters.

## Atualizar Variáveis de Ambiente

### 1. Editar o arquivo .env local

```bash
vim oms.env
# Faça suas alterações
```

### 2. Upload do arquivo atualizado

```bash
aws s3 cp oms.env s3://oms-spider-config-dev/oms/dev.env
```

### 3. Forçar nova Task Definition

```bash
# Option 1: Update stack (força nova revisão)
aws cloudformation update-stack \
  --stack-name oms-task-def-dev \
  --use-previous-template \
  --parameters \
    ParameterKey=Environment,UsePreviousValue=true \
    ParameterKey=ECSClusterStackName,UsePreviousValue=true \
    ParameterKey=ECRRepositoryUri,UsePreviousValue=true \
    ParameterKey=ImageTag,ParameterValue=latest-$(date +%s) \
    ParameterKey=EnvFileS3Arn,UsePreviousValue=true \
  --capabilities CAPABILITY_IAM

# Option 2: Force new deployment do service
aws ecs update-service \
  --cluster oms-spider-oms-cluster-dev \
  --service oms-spider-oms-service-dev \
  --force-new-deployment
```

## Estrutura Recomendada no S3

```
s3://oms-spider-config-{env}/
├── oms/
│   ├── dev.env
│   ├── staging.env
│   └── prod.env
├── frontend/
│   ├── dev.env
│   ├── staging.env
│   └── prod.env
└── websocket/
    ├── dev.env
    ├── staging.env
    └── prod.env
```

## Exemplo de Arquivo .env por Ambiente

### oms/dev.env

```env
# Database
DATABASE_URL=postgresql://postgres:password@localhost:5432/oms_dev

# Redis
REDIS_HOST=evo-redis-dev.abc123.0001.use1.cache.amazonaws.com
REDIS_PORT=6379

# Logging
LOG_LEVEL=debug
LOG_FORMAT=json

# App Config
MAX_WORKERS=4
QUEUE_NAME=oms-orders-queue-dev
ENABLE_METRICS=true

# Exchanges
BINANCE_API_KEY=your-dev-key
BINANCE_SECRET_KEY=your-dev-secret
BINGX_API_KEY=your-dev-key
BINGX_SECRET_KEY=your-dev-secret
```

### oms/prod.env

```env
# Database
DATABASE_URL=postgresql://user:pass@prod-db.rds.amazonaws.com:5432/oms_prod

# Redis
REDIS_HOST=evo-redis-prod.xyz789.ng.0001.use1.cache.amazonaws.com
REDIS_PORT=6379

# Logging
LOG_LEVEL=info
LOG_FORMAT=json

# App Config
MAX_WORKERS=16
QUEUE_NAME=oms-orders-queue-prod
ENABLE_METRICS=true

# Exchanges (usar Secrets Manager para prod)
# Não coloque secrets aqui em produção!
# Use AWS Secrets Manager + Secrets array na Task Definition
```

## Segurança

### ⚠️ IMPORTANTE: Secrets em Produção

**NÃO use arquivos .env para secrets sensíveis em produção!**

Para informações sensíveis (API keys, passwords, etc.), use:

1. **AWS Secrets Manager** - Via campo `Secrets` na Task Definition
2. **AWS Systems Manager Parameter Store** - Para configurações menos sensíveis

### Exemplo usando Secrets Manager

```json
{
  "Secrets": [
    {
      "Name": "BINANCE_API_KEY",
      "ValueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:prod/binance/api-key"
    },
    {
      "Name": "DATABASE_PASSWORD",
      "ValueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:prod/db/password"
    }
  ]
}
```

### Criptografia do Bucket S3

```bash
# Habilitar criptografia server-side no bucket
aws s3api put-bucket-encryption \
  --bucket oms-spider-config-dev \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'
```

### Versionamento do Bucket

```bash
# Habilitar versionamento
aws s3api put-bucket-versioning \
  --bucket oms-spider-config-dev \
  --versioning-configuration Status=Enabled
```

## Troubleshooting

### Task não inicia depois de adicionar EnvFileS3Arn

1. **Verificar permissões:**
```bash
aws iam get-role-policy \
  --role-name oms-spider-oms-task-execution-role-dev \
  --policy-name S3Access
```

2. **Verificar se arquivo existe:**
```bash
aws s3 ls s3://oms-spider-config-dev/oms/dev.env
```

3. **Verificar logs do ECS:**
```bash
aws ecs describe-tasks \
  --cluster oms-spider-oms-cluster-dev \
  --tasks task-id \
  --query 'tasks[0].stoppedReason'
```

### Variáveis não estão sendo carregadas

1. **Verificar formato do arquivo:**
   - Sem espaços ao redor do `=`
   - Uma variável por linha
   - Sem aspas desnecessárias

2. **Verificar ARN do arquivo:**
   - Deve ser `arn:aws:s3:::bucket-name/path/file.env`
   - Não apenas `s3://bucket-name/path/file.env`

3. **Forçar nova task:**
```bash
aws ecs update-service \
  --cluster cluster-name \
  --service service-name \
  --force-new-deployment
```

## Custo

- **S3 Storage:** ~$0.023/GB/mês
- **S3 Requests:** ~$0.0004/1000 GET requests
- **Estimativa:** Arquivos .env de ~10KB = praticamente grátis

**Total estimado:** < $0.01/mês por ambiente
