# Quick Start - CodeStar Connection

Guia rápido para obter o ARN da sua CodeStar Connection existente e fazer deploy da pipeline.

## Passo 1: Obter ARN da Connection

### Via Console AWS

1. Acesse: AWS Console → Developer Tools → Connections
2. Localize sua connection do GitHub
3. Copie o ARN (formato: `arn:aws:codestar-connections:us-east-1:123456789:connection/xxxxx`)

### Via AWS CLI

```bash
# Listar todas as connections
aws codestar-connections list-connections \
  --provider-type-filter GitHub \
  --region us-east-1 \
  --output table

# Obter ARN de uma connection específica
aws codestar-connections list-connections \
  --provider-type-filter GitHub \
  --region us-east-1 \
  --query 'Connections[?ConnectionName==`sua-connection-name`].ConnectionArn' \
  --output text
```

## Passo 2: Verificar Repositório ECR

```bash
# Verificar se o repositório ECR existe
aws ecr describe-repositories \
  --repository-names oms-spider \
  --region us-east-1

# Se não existir, criar
aws ecr create-repository \
  --repository-name oms-spider \
  --image-scanning-configuration scanOnPush=true \
  --region us-east-1
```

## Passo 3: Deploy da Pipeline

```bash
aws cloudformation create-stack \
  --stack-name oms-pipeline-dev \
  --template-body file://templates/cicd/codepipeline-codestar.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=ProjectName,ParameterValue=oms-spider \
    ParameterKey=ECRRepositoryName,ParameterValue=oms-spider \
    ParameterKey=CodeStarConnectionArn,ParameterValue=SEU_ARN_AQUI \
    ParameterKey=GitHubRepoFullName,ParameterValue=seu-usuario/seu-repo \
    ParameterKey=GitHubBranch,ParameterValue=main \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

Substitua:
- `SEU_ARN_AQUI`: ARN da sua CodeStar Connection
- `seu-usuario/seu-repo`: Nome completo do repositório GitHub
- `main`: Branch que você quer monitorar

## Passo 4: Aguardar Deploy

```bash
aws cloudformation wait stack-create-complete \
  --stack-name oms-pipeline-dev \
  --region us-east-1
```

## Passo 5: Obter URL da Pipeline

```bash
aws cloudformation describe-stacks \
  --stack-name oms-pipeline-dev \
  --query 'Stacks[0].Outputs[?OutputKey==`PipelineUrl`].OutputValue' \
  --output text
```

## Passo 6: Adicionar buildspec.yml ao Repositório

Copie o arquivo de exemplo para seu repositório:

```bash
cp examples/buildspec.yml /caminho/do/seu/repositorio/
cd /caminho/do/seu/repositorio/
git add buildspec.yml
git commit -m "Add buildspec for CodeBuild"
git push origin main
```

## Passo 7: Testar Pipeline

A pipeline será disparada automaticamente no push. Acompanhe:

```bash
# Ver status
aws codepipeline get-pipeline-state \
  --name oms-spider-pipeline-dev

# Ver logs do build
aws logs tail /aws/codebuild/oms-spider-dev --follow
```

## Exemplo Completo

```bash
#!/bin/bash
set -e

# Variáveis
ENVIRONMENT="dev"
PROJECT_NAME="oms-spider"
ECR_REPO="oms-spider"
CONNECTION_NAME="minha-github-connection"
GITHUB_REPO="meu-usuario/meu-repo"
BRANCH="main"
REGION="us-east-1"

# 1. Obter ARN da Connection
echo "Obtendo ARN da CodeStar Connection..."
CONNECTION_ARN=$(aws codestar-connections list-connections \
  --provider-type-filter GitHub \
  --region $REGION \
  --query "Connections[?ConnectionName=='$CONNECTION_NAME'].ConnectionArn" \
  --output text)

if [ -z "$CONNECTION_ARN" ]; then
  echo "Erro: Connection '$CONNECTION_NAME' não encontrada"
  exit 1
fi

echo "Connection ARN: $CONNECTION_ARN"

# 2. Verificar ECR
echo "Verificando repositório ECR..."
aws ecr describe-repositories \
  --repository-names $ECR_REPO \
  --region $REGION || \
aws ecr create-repository \
  --repository-name $ECR_REPO \
  --image-scanning-configuration scanOnPush=true \
  --region $REGION

# 3. Deploy Pipeline
echo "Fazendo deploy da pipeline..."
aws cloudformation create-stack \
  --stack-name ${PROJECT_NAME}-pipeline-${ENVIRONMENT} \
  --template-body file://templates/cicd/codepipeline-codestar.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=$ENVIRONMENT \
    ParameterKey=ProjectName,ParameterValue=$PROJECT_NAME \
    ParameterKey=ECRRepositoryName,ParameterValue=$ECR_REPO \
    ParameterKey=CodeStarConnectionArn,ParameterValue=$CONNECTION_ARN \
    ParameterKey=GitHubRepoFullName,ParameterValue=$GITHUB_REPO \
    ParameterKey=GitHubBranch,ParameterValue=$BRANCH \
  --capabilities CAPABILITY_NAMED_IAM \
  --region $REGION

# 4. Aguardar conclusão
echo "Aguardando criação do stack..."
aws cloudformation wait stack-create-complete \
  --stack-name ${PROJECT_NAME}-pipeline-${ENVIRONMENT} \
  --region $REGION

# 5. Obter outputs
echo "Pipeline criada com sucesso!"
aws cloudformation describe-stacks \
  --stack-name ${PROJECT_NAME}-pipeline-${ENVIRONMENT} \
  --region $REGION \
  --query 'Stacks[0].Outputs' \
  --output table

echo "Done!"
```

Salve como `deploy-pipeline.sh` e execute:

```bash
chmod +x deploy-pipeline.sh
./deploy-pipeline.sh
```

## Troubleshooting Rápido

### Connection não encontrada

```bash
# Listar todas as connections disponíveis
aws codestar-connections list-connections --region us-east-1
```

### ECR não existe

```bash
# Criar repositório ECR
aws ecr create-repository \
  --repository-name oms-spider \
  --region us-east-1
```

### Stack já existe

```bash
# Deletar stack existente
aws cloudformation delete-stack \
  --stack-name oms-pipeline-dev

# Aguardar deleção
aws cloudformation wait stack-delete-complete \
  --stack-name oms-pipeline-dev

# Recriar
./deploy-pipeline.sh
```

### Pipeline não dispara

Verificar se a connection está ativa:

```bash
aws codestar-connections get-connection \
  --connection-arn SEU_ARN_AQUI \
  --query 'Connection.ConnectionStatus'
```

Deve retornar `AVAILABLE`.

## Próximos Passos

Após o deploy bem-sucedido:

1. Verificar webhook no GitHub (criado automaticamente)
2. Fazer um push de teste no branch configurado
3. Acompanhar execução da pipeline no console AWS
4. Verificar imagem no ECR
5. Configurar alarmes e notificações (opcional)

## Recursos

- [Template CodeStar](../../templates/cicd/codepipeline-codestar.json)
- [Documentação Completa](codestar-connection.md)
- [Buildspec Examples](../../examples/)
