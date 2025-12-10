# Guia de Deployment

## Pré-requisitos

- AWS CLI configurado com credenciais válidas
- Permissões IAM para criar recursos CloudFormation, Cognito e DynamoDB
- AWS CLI versão 2.x ou superior

## Verificação de Pré-requisitos

```bash
# Verificar instalação AWS CLI
aws --version

# Verificar credenciais
aws sts get-caller-identity

# Verificar região configurada
aws configure get region
```

## Ordem de Deploy

O deploy deve ser realizado na seguinte ordem:

1. Cognito User Pool
2. DynamoDB Config Table
3. DynamoDB Trading Table

## Deploy por Ambiente

### Desenvolvimento (dev)

#### 1. Cognito User Pool

```bash
aws cloudformation create-stack \
  --stack-name oms-cognito-dev \
  --template-body file://templates/cognito/cognito-user-pool.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=AdminEmail,ParameterValue=admin@example.com \
  --capabilities CAPABILITY_IAM \
  --region us-east-1
```

Aguarde a conclusão:
```bash
aws cloudformation wait stack-create-complete \
  --stack-name oms-cognito-dev
```

#### 2. Config Table

```bash
aws cloudformation create-stack \
  --stack-name oms-config-dev \
  --template-body file://templates/dynamodb/oms-config-table.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=TableName,ParameterValue=oms_config \
    ParameterKey=EnablePITR,ParameterValue=false \
  --region us-east-1
```

Aguarde a conclusão:
```bash
aws cloudformation wait stack-create-complete \
  --stack-name oms-config-dev
```

#### 3. Trading Table

```bash
aws cloudformation create-stack \
  --stack-name oms-trading-dev \
  --template-body file://templates/dynamodb/oms-trading-table.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=TableName,ParameterValue=oms_trading_data \
    ParameterKey=EnablePITR,ParameterValue=false \
  --region us-east-1
```

Aguarde a conclusão:
```bash
aws cloudformation wait stack-create-complete \
  --stack-name oms-trading-dev
```

### Staging

Mesmo processo do dev, substituindo:
- Stack name: `oms-*-staging`
- Environment: `staging`

### Produção (prod)

Para produção, habilite PITR:

```bash
# Cognito
aws cloudformation create-stack \
  --stack-name oms-cognito-prod \
  --template-body file://templates/cognito/cognito-user-pool.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=prod \
    ParameterKey=AdminEmail,ParameterValue=admin@production.com \
  --capabilities CAPABILITY_IAM

# Config Table com PITR
aws cloudformation create-stack \
  --stack-name oms-config-prod \
  --template-body file://templates/dynamodb/oms-config-table.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=prod \
    ParameterKey=TableName,ParameterValue=oms_config \
    ParameterKey=EnablePITR,ParameterValue=true

# Trading Table com PITR
aws cloudformation create-stack \
  --stack-name oms-trading-prod \
  --template-body file://templates/dynamodb/oms-trading-table.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=prod \
    ParameterKey=TableName,ParameterValue=oms_trading_data \
    ParameterKey=EnablePITR,ParameterValue=true
```

## Obter Outputs dos Stacks

### Cognito Outputs

```bash
aws cloudformation describe-stacks \
  --stack-name oms-cognito-dev \
  --query 'Stacks[0].Outputs' \
  --output table
```

Outputs importantes:
- `UserPoolId`: ID do User Pool
- `UserPoolClientId`: Client ID para aplicação
- `CognitoRegion`: Região do Cognito

### DynamoDB Outputs

```bash
# Config Table
aws cloudformation describe-stacks \
  --stack-name oms-config-dev \
  --query 'Stacks[0].Outputs' \
  --output table

# Trading Table
aws cloudformation describe-stacks \
  --stack-name oms-trading-dev \
  --query 'Stacks[0].Outputs' \
  --output table
```

## Carregar Dados de Exemplo

Após criar a Config Table, carregue os dados de exemplo:

```bash
# Extrair itens do arquivo JSON
cd data

# Inserir Product 1
aws dynamodb put-item \
  --table-name oms_config_dev \
  --item file://product-1-item.json

# Inserir Product 2
aws dynamodb put-item \
  --table-name oms_config_dev \
  --item file://product-2-item.json
```

## Atualização de Stacks

### Update Stack

```bash
aws cloudformation update-stack \
  --stack-name oms-cognito-dev \
  --template-body file://templates/cognito/cognito-user-pool.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
  --capabilities CAPABILITY_IAM
```

### Validar Template Antes do Deploy

```bash
aws cloudformation validate-template \
  --template-body file://templates/cognito/cognito-user-pool.json
```

## Rollback

### Rollback Automático

Por padrão, CloudFormation faz rollback automático em caso de falha.

### Rollback Manual

```bash
aws cloudformation cancel-update-stack \
  --stack-name oms-cognito-dev
```

## Exclusão de Stacks

### Desenvolvimento

```bash
# Ordem inversa ao deploy
aws cloudformation delete-stack --stack-name oms-trading-dev
aws cloudformation delete-stack --stack-name oms-config-dev
aws cloudformation delete-stack --stack-name oms-cognito-dev
```

### Produção

Para produção, considere criar um snapshot antes de deletar:

```bash
# Backup on-demand
aws dynamodb create-backup \
  --table-name oms_config_prod \
  --backup-name oms-config-prod-manual-backup-DD-MM-YYYY

aws dynamodb create-backup \
  --table-name oms_trading_data_prod \
  --backup-name oms-trading-data-prod-manual-backup-DD-MM-YYYY

# Depois deletar stacks
aws cloudformation delete-stack --stack-name oms-trading-prod
aws cloudformation delete-stack --stack-name oms-config-prod
aws cloudformation delete-stack --stack-name oms-cognito-prod
```

## Monitoramento

### Logs de Stack Events

```bash
aws cloudformation describe-stack-events \
  --stack-name oms-cognito-dev \
  --max-items 10
```

### Status do Stack

```bash
aws cloudformation describe-stacks \
  --stack-name oms-cognito-dev \
  --query 'Stacks[0].StackStatus'
```

## Troubleshooting

### Stack CREATE_FAILED

```bash
# Ver erro específico
aws cloudformation describe-stack-events \
  --stack-name oms-cognito-dev \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]'
```

### Stack UPDATE_ROLLBACK_COMPLETE

Necessário deletar e recriar:
```bash
aws cloudformation delete-stack --stack-name oms-cognito-dev
# Aguardar conclusão
aws cloudformation wait stack-delete-complete --stack-name oms-cognito-dev
# Recriar
aws cloudformation create-stack ...
```

### Permissões Insuficientes

Verifique se o usuário/role possui as seguintes permissões:
- `cloudformation:*`
- `cognito-idp:*`
- `dynamodb:*`
- `kms:*` (para criptografia)
- `iam:PassRole` (se usar roles)

## Configuração da Aplicação

Após deploy, configure a aplicação Next.js com os outputs:

```env
# .env.local
NEXT_PUBLIC_AWS_REGION=us-east-1
NEXT_PUBLIC_COGNITO_USER_POOL_ID=us-east-1_XXXXXXXXX
NEXT_PUBLIC_COGNITO_CLIENT_ID=XXXXXXXXXXXXXXXXXXXXXXXXXX

AWS_DYNAMODB_CONFIG_TABLE=oms_config_dev
AWS_DYNAMODB_TRADING_TABLE=oms_trading_data_dev
```

## Scripts de Automação

### Deploy Completo

Crie um script `deploy.sh`:

```bash
#!/bin/bash
set -e

ENV=$1
REGION=${2:-us-east-1}
ADMIN_EMAIL=$3

if [ -z "$ENV" ]; then
  echo "Uso: ./deploy.sh <env> [region] [admin-email]"
  exit 1
fi

echo "Deploying to $ENV in $REGION..."

# Cognito
aws cloudformation create-stack \
  --stack-name oms-cognito-$ENV \
  --template-body file://templates/cognito/cognito-user-pool.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=$ENV \
    ParameterKey=AdminEmail,ParameterValue=$ADMIN_EMAIL \
  --region $REGION

# Config Table
aws cloudformation create-stack \
  --stack-name oms-config-$ENV \
  --template-body file://templates/dynamodb/oms-config-table.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=$ENV \
  --region $REGION

# Trading Table
aws cloudformation create-stack \
  --stack-name oms-trading-$ENV \
  --template-body file://templates/dynamodb/oms-trading-table.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=$ENV \
  --region $REGION

echo "Deploy iniciado. Aguarde conclusão..."
```

Uso:
```bash
chmod +x deploy.sh
./deploy.sh dev us-east-1 admin@example.com
```

## Checklist de Deploy

### Desenvolvimento
- [ ] Validar templates
- [ ] Deploy Cognito
- [ ] Deploy Config Table
- [ ] Deploy Trading Table
- [ ] Verificar outputs
- [ ] Carregar dados de exemplo
- [ ] Testar autenticação
- [ ] Verificar acesso DynamoDB

### Produção
- [ ] Review de segurança
- [ ] Habilitar PITR
- [ ] Configurar alarmes CloudWatch
- [ ] Backup manual antes de mudanças
- [ ] Deploy em horário de baixo tráfego
- [ ] Validar rollback plan
- [ ] Documentar outputs
- [ ] Configurar monitoramento
