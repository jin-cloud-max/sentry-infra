# CI/CD Pipeline com CodeStar Connection

## Template CloudFormation

[codepipeline-codestar.json](../../templates/cicd/codepipeline-codestar.json)

## Descrição

Este template utiliza AWS CodeStar Connection ao invés de GitHub OAuth token, proporcionando uma integração mais segura e gerenciada com o GitHub.

## Diferenças do Template Original

### Vantagens da CodeStar Connection

1. **Sem token manual**: Não precisa gerar e gerenciar GitHub personal access token
2. **Webhook automático**: Criado e gerenciado automaticamente pela AWS
3. **Mais seguro**: Credenciais gerenciadas pela AWS
4. **Melhor integração**: Suporte nativo a GitHub, GitLab e Bitbucket
5. **Auditoria**: Melhor rastreamento de acessos via CloudTrail

### Mudanças no Template

- **Source Provider**: `CodeStarSourceConnection` ao invés de `GitHub`
- **Parâmetros**: `CodeStarConnectionArn` e `GitHubRepoFullName` ao invés de `GitHubOwner`, `GitHubRepo` e `GitHubToken`
- **Webhook**: Gerenciado automaticamente, não precisa criar recurso separado
- **IAM**: Role do CodePipeline precisa de permissão `codestar-connections:UseConnection`

## Pré-requisitos

### 1. Criar CodeStar Connection

#### Via Console

1. Acessar AWS Console → Developer Tools → Connections
2. Clicar em "Create connection"
3. Selecionar "GitHub"
4. Nome da conexão: `oms-github-connection`
5. Clicar em "Connect to GitHub"
6. Autorizar no GitHub
7. Copiar o ARN da conexão criada

#### Via CLI

```bash
# Criar connection
aws codestar-connections create-connection \
  --provider-type GitHub \
  --connection-name oms-github-connection \
  --region us-east-1

# Output retorna o ARN
{
  "ConnectionArn": "arn:aws:codestar-connections:us-east-1:123456789:connection/xxxxx-xxxx-xxxx"
}
```

Após criar via CLI, você precisa completar o handshake no console:
1. AWS Console → Developer Tools → Connections
2. Selecionar a conexão criada
3. Clicar em "Update pending connection"
4. Autorizar no GitHub

### 2. Verificar Status da Connection

```bash
aws codestar-connections get-connection \
  --connection-arn arn:aws:codestar-connections:us-east-1:123456789:connection/xxxxx \
  --region us-east-1
```

Status deve ser `AVAILABLE`.

### 3. Obter ARN da Connection

```bash
aws codestar-connections list-connections \
  --provider-type-filter GitHub \
  --region us-east-1 \
  --query 'Connections[?ConnectionName==`oms-github-connection`].ConnectionArn' \
  --output text
```

## Deploy da Pipeline

### Parâmetros

| Parâmetro | Descrição | Exemplo |
|-----------|-----------|---------|
| Environment | Ambiente | dev |
| ProjectName | Nome do projeto | oms-spider |
| ECRRepositoryName | Nome do repositório ECR | oms-spider |
| CodeStarConnectionArn | ARN da CodeStar Connection | arn:aws:codestar-connections:... |
| GitHubRepoFullName | Repo completo (owner/repo) | seu-usuario/oms-spider |
| GitHubBranch | Branch para trigger | main |
| BuildspecPath | Caminho do buildspec | buildspec.yml |

### Deploy via CLI

```bash
aws cloudformation create-stack \
  --stack-name oms-pipeline-dev \
  --template-body file://templates/cicd/codepipeline-codestar.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=ProjectName,ParameterValue=oms-spider \
    ParameterKey=ECRRepositoryName,ParameterValue=oms-spider \
    ParameterKey=CodeStarConnectionArn,ParameterValue=arn:aws:codestar-connections:us-east-1:123456789:connection/xxxxx \
    ParameterKey=GitHubRepoFullName,ParameterValue=seu-usuario/oms-spider \
    ParameterKey=GitHubBranch,ParameterValue=main \
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

## Comparação: CodeStar Connection vs GitHub Token

### CodeStar Connection (Recomendado)

**Prós:**
- Sem gerenciamento manual de tokens
- Webhook automático
- Melhor segurança
- Suporte a múltiplos provedores
- Auditoria via CloudTrail
- Não expira (connection permanente)

**Contras:**
- Precisa criar connection previamente
- Requer autorização via console na primeira vez

### GitHub Token

**Prós:**
- Deploy totalmente automatizado
- Sem dependência de recursos externos

**Contras:**
- Token precisa ser armazenado
- Token pode expirar
- Menos seguro
- Webhook precisa ser gerenciado manualmente

## Testar Pipeline

### 1. Fazer Push no GitHub

```bash
cd seu-repositorio
git add .
git commit -m "Test CodeStar connection pipeline"
git push origin main
```

### 2. Verificar Trigger Automático

```bash
# Status da pipeline
aws codepipeline get-pipeline-state \
  --name oms-spider-pipeline-dev

# Histórico de execuções
aws codepipeline list-pipeline-executions \
  --pipeline-name oms-spider-pipeline-dev
```

### 3. Acompanhar Build

```bash
# Logs do CodeBuild
aws logs tail /aws/codebuild/oms-spider-dev --follow
```

## Gerenciar CodeStar Connection

### Listar Connections

```bash
aws codestar-connections list-connections \
  --provider-type-filter GitHub \
  --region us-east-1
```

### Ver Detalhes

```bash
aws codestar-connections get-connection \
  --connection-arn arn:aws:codestar-connections:us-east-1:123456789:connection/xxxxx
```

### Deletar Connection

```bash
aws codestar-connections delete-connection \
  --connection-arn arn:aws:codestar-connections:us-east-1:123456789:connection/xxxxx
```

## Ambientes Múltiplos

A mesma connection pode ser usada para múltiplas pipelines:

### Dev

```bash
aws cloudformation create-stack \
  --stack-name oms-pipeline-dev \
  --template-body file://templates/cicd/codepipeline-codestar.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=CodeStarConnectionArn,ParameterValue=arn:aws:codestar-connections:... \
    ParameterKey=GitHubRepoFullName,ParameterValue=seu-usuario/oms-spider \
    ParameterKey=GitHubBranch,ParameterValue=develop \
  --capabilities CAPABILITY_NAMED_IAM
```

### Staging

```bash
aws cloudformation create-stack \
  --stack-name oms-pipeline-staging \
  --template-body file://templates/cicd/codepipeline-codestar.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=staging \
    ParameterKey=CodeStarConnectionArn,ParameterValue=arn:aws:codestar-connections:... \
    ParameterKey=GitHubRepoFullName,ParameterValue=seu-usuario/oms-spider \
    ParameterKey=GitHubBranch,ParameterValue=staging \
  --capabilities CAPABILITY_NAMED_IAM
```

### Produção

```bash
aws cloudformation create-stack \
  --stack-name oms-pipeline-prod \
  --template-body file://templates/cicd/codepipeline-codestar.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=prod \
    ParameterKey=CodeStarConnectionArn,ParameterValue=arn:aws:codestar-connections:... \
    ParameterKey=GitHubRepoFullName,ParameterValue=seu-usuario/oms-spider \
    ParameterKey=GitHubBranch,ParameterValue=main \
  --capabilities CAPABILITY_NAMED_IAM
```

## Troubleshooting

### Connection Status PENDING

A connection foi criada mas não foi autorizada no GitHub.

Solução:
1. AWS Console → Developer Tools → Connections
2. Selecionar a connection
3. "Update pending connection"
4. Autorizar no GitHub

### Pipeline não dispara automaticamente

Verificar:
1. Connection status é `AVAILABLE`
2. Branch está correto
3. DetectChanges está true

```bash
# Ver configuração da pipeline
aws codepipeline get-pipeline \
  --name oms-spider-pipeline-dev \
  --query 'pipeline.stages[0].actions[0].configuration'
```

### Erro de permissão UseConnection

A role do CodePipeline não tem permissão para usar a connection.

Verificar:
```bash
aws iam get-role-policy \
  --role-name oms-spider-codepipeline-role-dev \
  --policy-name CodePipelinePolicy
```

Deve conter:
```json
{
  "Effect": "Allow",
  "Action": ["codestar-connections:UseConnection"],
  "Resource": "arn:aws:codestar-connections:..."
}
```

### Connection não encontrada

Verificar região da connection:
```bash
aws codestar-connections list-connections \
  --region us-east-1
```

A connection e a pipeline devem estar na mesma região.

## Migração de GitHub Token para CodeStar

Se você já tem uma pipeline usando GitHub token:

### 1. Criar CodeStar Connection

```bash
aws codestar-connections create-connection \
  --provider-type GitHub \
  --connection-name oms-github-connection \
  --region us-east-1
```

### 2. Autorizar no Console

### 3. Atualizar Stack

```bash
aws cloudformation update-stack \
  --stack-name oms-pipeline-dev \
  --template-body file://templates/cicd/codepipeline-codestar.json \
  --parameters \
    ParameterKey=CodeStarConnectionArn,ParameterValue=arn:aws:codestar-connections:... \
    ParameterKey=GitHubRepoFullName,ParameterValue=seu-usuario/oms-spider \
  --capabilities CAPABILITY_NAMED_IAM
```

## Segurança

### IAM Permissions

A connection usa IAM para controlar acesso:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "codestar-connections:UseConnection"
      ],
      "Resource": "arn:aws:codestar-connections:us-east-1:123456789:connection/*"
    }
  ]
}
```

### CloudTrail Audit

Todas as ações são logadas no CloudTrail:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceType,AttributeValue=AWS::CodeStarConnections::Connection \
  --max-results 10
```

### Revogar Acesso

Para revogar acesso ao GitHub:
1. Deletar a connection na AWS
2. Revogar autorização no GitHub Settings → Applications

## Boas Práticas

1. **Uma connection por conta**: Reutilizar a mesma connection para todas as pipelines
2. **Nomear claramente**: Use nomes descritivos para connections
3. **Documentar ARN**: Salvar ARN da connection em documentação
4. **Monitorar uso**: Configurar alarmes para mudanças na connection
5. **Backup**: Documentar processo de recriação da connection

## Recursos Adicionais

- [AWS CodeStar Connections Documentation](https://docs.aws.amazon.com/codepipeline/latest/userguide/connections.html)
- [GitHub App Integration](https://docs.aws.amazon.com/dtconsole/latest/userguide/connections-create-github.html)
- [Troubleshooting Connections](https://docs.aws.amazon.com/dtconsole/latest/userguide/connections-troubleshoot.html)
