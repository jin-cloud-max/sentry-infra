# Change Sets - Guia de Uso

Change Sets permitem visualizar as mudanças antes de aplicá-las à stack. Isso é **obrigatório para produção** e recomendado para outros ambientes.

## Fluxo de Change Set

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Criar     │ ──▶ │  Revisar    │ ──▶ │  Executar   │ ──▶ │   Stack     │
│ Change Set  │     │  Mudanças   │     │ Change Set  │     │ Atualizada  │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
                          │
                          ▼ (se não aprovar)
                    ┌─────────────┐
                    │   Deletar   │
                    │ Change Set  │
                    └─────────────┘
```

## Comandos Básicos

### 1. Criar Change Set

```bash
# Variáveis
STACK_NAME="sentry-vpc-dev"
CHANGE_SET_NAME="update-$(date +%Y%m%d-%H%M%S)"
TEMPLATE="templates/networking/vpc-nonprod.json"

# Criar change set
aws cloudformation create-change-set \
  --stack-name ${STACK_NAME} \
  --change-set-name ${CHANGE_SET_NAME} \
  --template-body file://${TEMPLATE} \
  --parameters file://params/dev/vpc.json \
  --description "Descrição das mudanças"

# Aguardar criação
aws cloudformation wait change-set-create-complete \
  --stack-name ${STACK_NAME} \
  --change-set-name ${CHANGE_SET_NAME}
```

### 2. Revisar Mudanças

```bash
# Ver resumo das mudanças
aws cloudformation describe-change-set \
  --stack-name ${STACK_NAME} \
  --change-set-name ${CHANGE_SET_NAME} \
  --query 'Changes[].{
    Action: ResourceChange.Action,
    LogicalId: ResourceChange.LogicalResourceId,
    Type: ResourceChange.ResourceType,
    Replacement: ResourceChange.Replacement
  }' \
  --output table
```

Exemplo de output:
```
-------------------------------------------------------------------
|                       DescribeChangeSet                          |
+---------+---------------------+---------------------------+------+
| Action  |      LogicalId      |           Type            |Replace|
+---------+---------------------+---------------------------+------+
| Modify  | SecurityGroup       | AWS::EC2::SecurityGroup   | False |
| Add     | NewResource         | AWS::EC2::Instance        | None  |
| Remove  | OldResource         | AWS::EC2::Instance        | None  |
+---------+---------------------+---------------------------+------+
```

### 3. Executar ou Cancelar

```bash
# APROVAR - Executar change set
aws cloudformation execute-change-set \
  --stack-name ${STACK_NAME} \
  --change-set-name ${CHANGE_SET_NAME}

# REJEITAR - Deletar change set
aws cloudformation delete-change-set \
  --stack-name ${STACK_NAME} \
  --change-set-name ${CHANGE_SET_NAME}
```

## Scripts Úteis

### Script: Criar e Revisar Change Set

```bash
#!/bin/bash
# changeset-create.sh

STACK_NAME=$1
TEMPLATE=$2
PARAMS_FILE=$3
DESCRIPTION=${4:-"Change set update"}

if [ -z "$STACK_NAME" ] || [ -z "$TEMPLATE" ]; then
  echo "Uso: $0 <stack-name> <template-file> [params-file] [description]"
  exit 1
fi

CHANGE_SET_NAME="cs-$(date +%Y%m%d-%H%M%S)"

echo "Criando change set: ${CHANGE_SET_NAME}"

# Montar comando
CMD="aws cloudformation create-change-set \
  --stack-name ${STACK_NAME} \
  --change-set-name ${CHANGE_SET_NAME} \
  --template-body file://${TEMPLATE} \
  --description \"${DESCRIPTION}\""

if [ -n "$PARAMS_FILE" ]; then
  CMD="$CMD --parameters file://${PARAMS_FILE}"
fi

# Executar
eval $CMD

echo "Aguardando criação..."
aws cloudformation wait change-set-create-complete \
  --stack-name ${STACK_NAME} \
  --change-set-name ${CHANGE_SET_NAME}

echo ""
echo "=== MUDANÇAS PROPOSTAS ==="
aws cloudformation describe-change-set \
  --stack-name ${STACK_NAME} \
  --change-set-name ${CHANGE_SET_NAME} \
  --query 'Changes[].{
    Action: ResourceChange.Action,
    LogicalId: ResourceChange.LogicalResourceId,
    Type: ResourceChange.ResourceType,
    Replacement: ResourceChange.Replacement
  }' \
  --output table

echo ""
echo "Para aplicar: aws cloudformation execute-change-set --stack-name ${STACK_NAME} --change-set-name ${CHANGE_SET_NAME}"
echo "Para cancelar: aws cloudformation delete-change-set --stack-name ${STACK_NAME} --change-set-name ${CHANGE_SET_NAME}"
```

### Script: Listar Change Sets Pendentes

```bash
#!/bin/bash
# changeset-list.sh

STACK_NAME=$1

if [ -z "$STACK_NAME" ]; then
  echo "Uso: $0 <stack-name>"
  exit 1
fi

aws cloudformation list-change-sets \
  --stack-name ${STACK_NAME} \
  --query 'Summaries[?Status!=`DELETE_COMPLETE`].{
    Name: ChangeSetName,
    Status: Status,
    Created: CreationTime,
    Description: Description
  }' \
  --output table
```

## Exemplos Práticos

### Atualizar Security Group da VPC

```bash
# 1. Fazer alteração no template
# (editar templates/networking/vpc-nonprod.json)

# 2. Criar change set
aws cloudformation create-change-set \
  --stack-name sentry-vpc-dev \
  --change-set-name update-sg-rules \
  --template-body file://templates/networking/vpc-nonprod.json \
  --parameters file://params/dev/vpc.json \
  --description "Adicionar regra de ingress para porta 443"

# 3. Revisar
aws cloudformation describe-change-set \
  --stack-name sentry-vpc-dev \
  --change-set-name update-sg-rules

# 4. Aplicar (se aprovado)
aws cloudformation execute-change-set \
  --stack-name sentry-vpc-dev \
  --change-set-name update-sg-rules
```

### Atualizar Task Definition

```bash
# 1. Criar change set
aws cloudformation create-change-set \
  --stack-name sentry-task-frontend-dev \
  --change-set-name update-cpu-memory \
  --template-body file://templates/compute/task-definition-frontend-admin.json \
  --parameters file://params/dev/task-frontend.json \
  --description "Aumentar CPU de 256 para 512"

# 2. Revisar e aplicar...
```

## Tipos de Mudanças

| Action | Significado | Risco |
|--------|-------------|-------|
| `Add` | Novo recurso será criado | Baixo |
| `Modify` | Recurso existente será atualizado | Médio |
| `Remove` | Recurso será deletado | Alto |

| Replacement | Significado | Risco |
|-------------|-------------|-------|
| `True` | Recurso será **recriado** (delete + create) | **ALTO** |
| `False` | Atualização in-place | Baixo |
| `Conditional` | Depende de outros fatores | Médio |

## Boas Práticas

1. **Sempre use change sets em produção** - Nunca faça `update-stack` direto
2. **Revise o campo Replacement** - Se for `True`, o recurso será recriado (downtime!)
3. **Documente as mudanças** - Use o campo `--description`
4. **Nome descritivo** - Use nomes como `add-sg-rule-443` ao invés de `update-1`
5. **Um change set por vez** - Não acumule múltiplos change sets pendentes

## Rollback

Se algo der errado após executar um change set:

```bash
# O CloudFormation faz rollback automático em caso de falha
# Para rollback manual, crie um novo change set com o template anterior:

aws cloudformation create-change-set \
  --stack-name sentry-vpc-dev \
  --change-set-name rollback-to-previous \
  --template-body file://templates/networking/vpc-nonprod.json.bak \
  --parameters file://params/dev/vpc.json.bak \
  --description "Rollback para versão anterior"
```

## Integração com Git

Recomendação: versione os templates e parâmetros no Git.

```bash
# Antes de criar change set, commitar as mudanças
git add templates/networking/vpc-nonprod.json
git commit -m "feat(vpc): add ingress rule for port 443"

# Criar change set com hash do commit na descrição
COMMIT_HASH=$(git rev-parse --short HEAD)
aws cloudformation create-change-set \
  --stack-name sentry-vpc-dev \
  --change-set-name "update-${COMMIT_HASH}" \
  --template-body file://templates/networking/vpc-nonprod.json \
  --description "Commit: ${COMMIT_HASH} - Add ingress rule for port 443"
```
