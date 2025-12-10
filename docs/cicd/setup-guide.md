# CI/CD Pipeline - Guia de Configuração

## Template CloudFormation

[codepipeline.json](../../templates/cicd/codepipeline.json)

## Pré-requisitos

### 1. Repositório ECR Existente

Verificar se o repositório ECR já existe:

```bash
aws ecr describe-repositories \
  --repository-names oms-spider \
  --region us-east-1
```

Se não existir, criar:

```bash
aws ecr create-repository \
  --repository-name oms-spider \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=AES256 \
  --region us-east-1
```

### 2. GitHub Personal Access Token

Criar token no GitHub com as seguintes permissões:
- `repo` (acesso completo a repositórios privados)
- `admin:repo_hook` (criar webhooks)

Passos:
1. GitHub → Settings → Developer settings → Personal access tokens → Generate new token
2. Selecionar scopes: `repo` e `admin:repo_hook`
3. Copiar e guardar o token com segurança

### 3. Repositório GitHub

Ter um repositório com:
- Dockerfile na raiz ou em subdiretório
- buildspec.yml (será criado a seguir)
- Código da aplicação

## Deploy da Pipeline

### Parâmetros do Template

| Parâmetro | Descrição | Exemplo |
|-----------|-----------|---------|
| Environment | Ambiente (dev/staging/prod) | dev |
| ProjectName | Nome do projeto | oms-spider |
| ECRRepositoryName | Nome do repositório ECR | oms-spider |
| GitHubOwner | Owner do repositório GitHub | seu-usuario |
| GitHubRepo | Nome do repositório | oms-spider |
| GitHubBranch | Branch para trigger | main |
| GitHubToken | Token de acesso do GitHub | ghp_xxx |
| BuildspecPath | Caminho do buildspec | buildspec.yml |

### Deploy via CLI

```bash
aws cloudformation create-stack \
  --stack-name oms-pipeline-dev \
  --template-body file://templates/cicd/codepipeline.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=ProjectName,ParameterValue=oms-spider \
    ParameterKey=ECRRepositoryName,ParameterValue=oms-spider \
    ParameterKey=GitHubOwner,ParameterValue=seu-usuario \
    ParameterKey=GitHubRepo,ParameterValue=oms-spider \
    ParameterKey=GitHubBranch,ParameterValue=main \
    ParameterKey=GitHubToken,ParameterValue=ghp_seu_token_aqui \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

### Aguardar Conclusão

```bash
aws cloudformation wait stack-create-complete \
  --stack-name oms-pipeline-dev \
  --region us-east-1
```

### Obter Outputs

```bash
aws cloudformation describe-stacks \
  --stack-name oms-pipeline-dev \
  --query 'Stacks[0].Outputs' \
  --output table
```

Exemplo de output:
```
---------------------------------------------------------------------------
|                           DescribeStacks                                |
+----------------------+--------------------------------------------------+
|  ArtifactBucketName  |  oms-spider-pipeline-artifacts-dev              |
|  CodeBuildProjectName|  oms-spider-build-dev                           |
|  PipelineName        |  oms-spider-pipeline-dev                        |
|  PipelineUrl         |  https://console.aws.amazon.com/codesuite/...   |
|  WebhookUrl          |  https://webhooks.amazonaws.com/...             |
+----------------------+--------------------------------------------------+
```

## Criar buildspec.yml

Adicionar o arquivo `buildspec.yml` na raiz do seu repositório:

```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPOSITORY_NAME
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
      - echo Repository URI = $REPOSITORY_URI
      - echo Image tag = $IMAGE_TAG

  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $REPOSITORY_URI:latest .
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$ENVIRONMENT-latest

  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker images...
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - docker push $REPOSITORY_URI:$ENVIRONMENT-latest
      - echo Writing image definitions file...
      - printf '[{"name":"app","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
```

### Buildspec com Multi-Stage Dockerfile

Se usar Dockerfile multi-stage para otimização:

```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPOSITORY_NAME
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}

  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image with build cache...
      - docker build --cache-from $REPOSITORY_URI:latest --build-arg BUILDKIT_INLINE_CACHE=1 -t $REPOSITORY_URI:latest .
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$ENVIRONMENT-latest

  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker images...
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - docker push $REPOSITORY_URI:$ENVIRONMENT-latest

artifacts:
  files:
    - imagedefinitions.json
```

### Buildspec com Testes

Incluindo execução de testes:

```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPOSITORY_NAME
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}

  build:
    commands:
      - echo Build started on `date`
      - echo Running tests...
      - npm install
      - npm test
      - echo Building the Docker image...
      - docker build -t $REPOSITORY_URI:latest .
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$ENVIRONMENT-latest

  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker images...
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - docker push $REPOSITORY_URI:$ENVIRONMENT-latest

artifacts:
  files:
    - imagedefinitions.json

reports:
  test_report:
    files:
      - 'test-results/**/*.xml'
    file-format: 'JUNITXML'
```

## Verificar Pipeline

### Status da Pipeline

```bash
aws codepipeline get-pipeline-state \
  --name oms-spider-pipeline-dev \
  --query 'stageStates[*].[stageName,latestExecution.status]' \
  --output table
```

### Histórico de Execuções

```bash
aws codepipeline list-pipeline-executions \
  --pipeline-name oms-spider-pipeline-dev \
  --max-items 10
```

### Detalhes de uma Execução

```bash
aws codepipeline get-pipeline-execution \
  --pipeline-name oms-spider-pipeline-dev \
  --pipeline-execution-id <execution-id>
```

## Testar Pipeline

### 1. Fazer Push no GitHub

```bash
cd seu-repositorio
git add .
git commit -m "Trigger pipeline test"
git push origin main
```

### 2. Acompanhar Execução

Via Console:
- Acessar URL do output `PipelineUrl`
- Visualizar stages em tempo real

Via CLI:
```bash
# Ver status
aws codepipeline get-pipeline-state \
  --name oms-spider-pipeline-dev

# Ver logs do CodeBuild
aws logs tail /aws/codebuild/oms-spider-dev --follow
```

### 3. Verificar Imagem no ECR

```bash
# Listar imagens
aws ecr list-images \
  --repository-name oms-spider \
  --region us-east-1

# Descrever imagem específica
aws ecr describe-images \
  --repository-name oms-spider \
  --image-ids imageTag=latest \
  --region us-east-1
```

## Executar Pipeline Manualmente

```bash
aws codepipeline start-pipeline-execution \
  --name oms-spider-pipeline-dev
```

## Configurações Avançadas

### Usar Secrets Manager para GitHub Token

1. Criar secret:

```bash
aws secretsmanager create-secret \
  --name oms/github-token \
  --secret-string '{"token":"ghp_seu_token_aqui"}' \
  --region us-east-1
```

2. Atualizar template para referenciar o secret
3. Adicionar permissão na IAM role do CodePipeline

### Adicionar Cache de Docker Layers

Atualizar CodeBuild para usar cache:

```json
"Cache": {
  "Type": "LOCAL",
  "Modes": ["LOCAL_DOCKER_LAYER_CACHE"]
}
```

### Notificações SNS

Criar tópico SNS e adicionar notification rule:

```bash
# Criar tópico
aws sns create-topic --name oms-pipeline-notifications

# Criar notification rule
aws codestar-notifications create-notification-rule \
  --name oms-pipeline-notifications \
  --event-type-ids \
    codepipeline-pipeline-pipeline-execution-failed \
    codepipeline-pipeline-pipeline-execution-succeeded \
  --resource arn:aws:codepipeline:us-east-1:123456789:oms-spider-pipeline-dev \
  --targets TargetType=SNS,TargetAddress=arn:aws:sns:us-east-1:123456789:oms-pipeline-notifications
```

## Ambientes Múltiplos

### Criar Pipeline para Staging

```bash
aws cloudformation create-stack \
  --stack-name oms-pipeline-staging \
  --template-body file://templates/cicd/codepipeline.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=staging \
    ParameterKey=ProjectName,ParameterValue=oms-spider \
    ParameterKey=ECRRepositoryName,ParameterValue=oms-spider \
    ParameterKey=GitHubOwner,ParameterValue=seu-usuario \
    ParameterKey=GitHubRepo,ParameterValue=oms-spider \
    ParameterKey=GitHubBranch,ParameterValue=staging \
    ParameterKey=GitHubToken,ParameterValue=ghp_seu_token_aqui \
  --capabilities CAPABILITY_NAMED_IAM
```

### Criar Pipeline para Produção

```bash
aws cloudformation create-stack \
  --stack-name oms-pipeline-prod \
  --template-body file://templates/cicd/codepipeline.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=prod \
    ParameterKey=ProjectName,ParameterValue=oms-spider \
    ParameterKey=ECRRepositoryName,ParameterValue=oms-spider \
    ParameterKey=GitHubOwner,ParameterValue=seu-usuario \
    ParameterKey=GitHubRepo,ParameterValue=oms-spider \
    ParameterKey=GitHubBranch,ParameterValue=main \
    ParameterKey=GitHubToken,ParameterValue=ghp_seu_token_aqui \
  --capabilities CAPABILITY_NAMED_IAM
```

## Troubleshooting

### Pipeline não dispara no push

Verificar webhook no GitHub:
1. GitHub → Repositório → Settings → Webhooks
2. Verificar se webhook da AWS está listado
3. Ver deliveries recentes para debugging

Recriar webhook:
```bash
aws codepipeline deregister-webhook-with-third-party \
  --webhook-name oms-spider-webhook-dev

aws codepipeline register-webhook-with-third-party \
  --webhook-name oms-spider-webhook-dev
```

### Build falha no Docker build

Ver logs detalhados:
```bash
aws codebuild batch-get-builds \
  --ids <build-id> \
  --query 'builds[0].logs.deepLink'
```

Causas comuns:
- Dockerfile com erros
- Dependências faltando
- Context do Docker incorreto

### Erro de permissão no ECR push

Verificar IAM role:
```bash
aws iam get-role-policy \
  --role-name oms-spider-codebuild-role-dev \
  --policy-name CodeBuildPolicy
```

Testar push manual:
```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  <account-id>.dkr.ecr.us-east-1.amazonaws.com

docker tag minha-imagem:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/oms-spider:test
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/oms-spider:test
```

### Artefatos muito grandes

Causas:
- Imagem Docker muito grande
- Muitos arquivos no source

Soluções:
- Usar .dockerignore
- Multi-stage builds
- Otimizar layers do Docker

## Manutenção

### Atualizar Stack

```bash
aws cloudformation update-stack \
  --stack-name oms-pipeline-dev \
  --template-body file://templates/cicd/codepipeline.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=GitHubToken,ParameterValue=novo_token \
  --capabilities CAPABILITY_NAMED_IAM
```

### Limpar Artefatos Antigos

```bash
# Listar objetos no bucket
aws s3 ls s3://oms-spider-pipeline-artifacts-dev/ --recursive

# Deletar objetos antigos (cuidado!)
aws s3 rm s3://oms-spider-pipeline-artifacts-dev/ --recursive
```

### Deletar Pipeline

```bash
aws cloudformation delete-stack \
  --stack-name oms-pipeline-dev
```

## Checklist de Deploy

### Desenvolvimento
- [ ] ECR repository criado
- [ ] GitHub token gerado
- [ ] buildspec.yml no repositório
- [ ] Dockerfile testado localmente
- [ ] Deploy do stack CloudFormation
- [ ] Verificar webhook no GitHub
- [ ] Fazer push de teste
- [ ] Verificar imagem no ECR

### Produção
- [ ] Usar AWS Secrets Manager para token
- [ ] Habilitar branch protection no GitHub
- [ ] Configurar notificações SNS
- [ ] Habilitar ECR scan on push
- [ ] Configurar alarmes CloudWatch
- [ ] Documentar processo de rollback
- [ ] Testar pipeline em staging primeiro
