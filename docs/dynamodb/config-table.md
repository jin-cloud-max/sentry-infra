# Config Table - Documentação

## Template CloudFormation

[oms-config-table.json](../../templates/dynamodb/oms-config-table.json)

## Descrição

Tabela DynamoDB para armazenamento de configurações estáticas do OMS, incluindo produtos e clientes.

## Características

- **Table Name**: `oms_config_{environment}`
- **Billing Mode**: PAY_PER_REQUEST
- **Encryption**: KMS (Server-Side Encryption)
- **PITR**: Configurável por parâmetro
- **TTL**: Não habilitado
- **GSIs**: Nenhum

## Estrutura de Chaves

### Primary Key

```
PK (HASH): PRODUCT#{productId} | CLIENT#{clientId}
SK (RANGE): CONFIG
```

### Padrão de Nomenclatura

- **PRODUCT**: `PRODUCT#{id}`
- **CLIENT**: `CLIENT#{id}`
- **SK**: Sempre `CONFIG`

## Modelo de Dados

### Entidade: PRODUCT

```json
{
  "PK": "PRODUCT#1",
  "SK": "CONFIG",
  "entity_type": "PRODUCT",
  "id": "1",
  "name": "Product Type 3",
  "type": "TYPE_3",
  "factor": 10,
  "multiplier": 50,
  "market_type": "futures",
  "position_mode": "hedging",
  "symbols": ["BTCUSDT", "ETHUSDT"],
  "real_account": true,
  "status": "active",
  "created_at": "28/10/2025 00:00:00",
  "updated_at": "28/10/2025 00:00:00",
  "metadata": {
    "analyst": {
      "id": "analyst_001",
      "exchange": "bitget",
      "credentials": {
        "api_key": "bg_xxx",
        "secret_key": "xxx",
        "passphrase": "xxx"
      }
    }
  }
}
```

#### Atributos

| Atributo | Tipo | Descrição |
|----------|------|-----------|
| PK | String | Partition Key: PRODUCT#{id} |
| SK | String | Sort Key: CONFIG |
| entity_type | String | Tipo da entidade: PRODUCT |
| id | String | Identificador único do produto |
| name | String | Nome descritivo do produto |
| type | String | Tipo do produto (TYPE_1, TYPE_2, TYPE_3) |
| factor | Number | Fator de multiplicação para cálculos |
| multiplier | Number | Multiplicador para volume |
| market_type | String | Tipo de mercado (spot, futures) |
| position_mode | String | Modo de posição (hedging, one_way) |
| symbols | List | Lista de símbolos negociados |
| real_account | Boolean | Indica se é conta real ou demo |
| status | String | Status do produto (active, inactive) |
| created_at | String | Data de criação (DD/MM/YYYY HH:mm:ss) |
| updated_at | String | Data de atualização (DD/MM/YYYY HH:mm:ss) |
| metadata | Map | Dados adicionais (analista e credenciais) |

### Entidade: CLIENT

```json
{
  "PK": "CLIENT#client_001",
  "SK": "CONFIG",
  "entity_type": "CLIENT",
  "id": "client_001",
  "name": "Cliente Exemplo",
  "email": "cliente@example.com",
  "status": "active",
  "products": ["1", "2"],
  "risk_limits": {
    "max_position_size": 100000,
    "max_daily_loss": 5000
  },
  "created_at": "14/11/2025 08:00:00",
  "updated_at": "14/11/2025 08:00:00"
}
```

#### Atributos

| Atributo | Tipo | Descrição |
|----------|------|-----------|
| PK | String | Partition Key: CLIENT#{id} |
| SK | String | Sort Key: CONFIG |
| entity_type | String | Tipo da entidade: CLIENT |
| id | String | Identificador único do cliente |
| name | String | Nome do cliente |
| email | String | Email do cliente |
| status | String | Status do cliente (active, inactive) |
| products | List | Lista de IDs de produtos associados |
| risk_limits | Map | Limites de risco configurados |
| created_at | String | Data de criação (DD/MM/YYYY HH:mm:ss) |
| updated_at | String | Data de atualização (DD/MM/YYYY HH:mm:ss) |

## Padrões de Acesso

### 1. Buscar Produto por ID

```bash
aws dynamodb get-item \
  --table-name oms_config_dev \
  --key '{
    "PK": {"S": "PRODUCT#1"},
    "SK": {"S": "CONFIG"}
  }'
```

```javascript
// SDK JavaScript v3
import { DynamoDBClient, GetItemCommand } from '@aws-sdk/client-dynamodb';

const client = new DynamoDBClient({ region: 'us-east-1' });

const command = new GetItemCommand({
  TableName: 'oms_config_dev',
  Key: {
    PK: { S: 'PRODUCT#1' },
    SK: { S: 'CONFIG' },
  },
});

const response = await client.send(command);
```

### 2. Listar Todos os Produtos

```bash
aws dynamodb query \
  --table-name oms_config_dev \
  --key-condition-expression "begins_with(PK, :pk) AND SK = :sk" \
  --expression-attribute-values '{
    ":pk": {"S": "PRODUCT#"},
    ":sk": {"S": "CONFIG"}
  }'
```

```javascript
// SDK JavaScript v3
import { DynamoDBClient, QueryCommand } from '@aws-sdk/client-dynamodb';

const client = new DynamoDBClient({ region: 'us-east-1' });

const command = new QueryCommand({
  TableName: 'oms_config_dev',
  KeyConditionExpression: 'begins_with(PK, :pk) AND SK = :sk',
  ExpressionAttributeValues: {
    ':pk': { S: 'PRODUCT#' },
    ':sk': { S: 'CONFIG' },
  },
});

const response = await client.send(command);
```

### 3. Buscar Cliente por ID

```bash
aws dynamodb get-item \
  --table-name oms_config_dev \
  --key '{
    "PK": {"S": "CLIENT#client_001"},
    "SK": {"S": "CONFIG"}
  }'
```

### 4. Listar Todos os Clientes

```bash
aws dynamodb query \
  --table-name oms_config_dev \
  --key-condition-expression "begins_with(PK, :pk) AND SK = :sk" \
  --expression-attribute-values '{
    ":pk": {"S": "CLIENT#"},
    ":sk": {"S": "CONFIG"}
  }'
```

### 5. Buscar Produtos Ativos

```bash
aws dynamodb scan \
  --table-name oms_config_dev \
  --filter-expression "entity_type = :type AND #status = :status" \
  --expression-attribute-names '{"#status": "status"}' \
  --expression-attribute-values '{
    ":type": {"S": "PRODUCT"},
    ":status": {"S": "active"}
  }'
```

### 6. Criar Novo Produto

```bash
aws dynamodb put-item \
  --table-name oms_config_dev \
  --item '{
    "PK": {"S": "PRODUCT#3"},
    "SK": {"S": "CONFIG"},
    "entity_type": {"S": "PRODUCT"},
    "id": {"S": "3"},
    "name": {"S": "New Product"},
    "type": {"S": "TYPE_1"},
    "factor": {"N": "5"},
    "multiplier": {"N": "25"},
    "market_type": {"S": "futures"},
    "position_mode": {"S": "one_way"},
    "symbols": {"L": [{"S": "BTCUSDT"}]},
    "real_account": {"BOOL": true},
    "status": {"S": "active"},
    "created_at": {"S": "14/11/2025 12:00:00"},
    "updated_at": {"S": "14/11/2025 12:00:00"}
  }'
```

### 7. Atualizar Produto

```bash
aws dynamodb update-item \
  --table-name oms_config_dev \
  --key '{
    "PK": {"S": "PRODUCT#1"},
    "SK": {"S": "CONFIG"}
  }' \
  --update-expression "SET #status = :status, updated_at = :updated" \
  --expression-attribute-names '{"#status": "status"}' \
  --expression-attribute-values '{
    ":status": {"S": "inactive"},
    ":updated": {"S": "14/11/2025 13:00:00"}
  }'
```

### 8. Deletar Produto

```bash
aws dynamodb delete-item \
  --table-name oms_config_dev \
  --key '{
    "PK": {"S": "PRODUCT#3"},
    "SK": {"S": "CONFIG"}
  }'
```

## Deploy

### Parâmetros

| Parâmetro | Tipo | Default | Descrição |
|-----------|------|---------|-----------|
| Environment | String | dev | Ambiente (dev/staging/prod) |
| TableName | String | oms_config | Nome base da tabela |
| EnablePITR | String | false | Habilitar Point-in-Time Recovery |

### Deploy via CLI

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

### Deploy em Produção

```bash
aws cloudformation create-stack \
  --stack-name oms-config-prod \
  --template-body file://templates/dynamodb/oms-config-table.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=prod \
    ParameterKey=TableName,ParameterValue=oms_config \
    ParameterKey=EnablePITR,ParameterValue=true \
  --region us-east-1
```

## Carregar Dados de Exemplo

Ver arquivo de dados: [ANALYST-DATA-TRANSFORMED.json](../../data/ANALYST-DATA-TRANSFORMED.json)

```bash
# Extrair e inserir cada produto
aws dynamodb put-item \
  --table-name oms_config_dev \
  --item file://data/product-1.json

aws dynamodb put-item \
  --table-name oms_config_dev \
  --item file://data/product-2.json
```

## Boas Práticas

### Segurança

1. Nunca expor credenciais do analista via API
2. Usar IAM roles para acesso controlado
3. Habilitar encryption at rest (KMS)
4. Habilitar PITR em produção

### Performance

1. Usar GetItem para buscar por ID (mais eficiente)
2. Evitar Scan quando possível
3. Implementar caching para dados frequentemente acessados
4. Usar BatchGetItem para múltiplos itens

### Manutenção

1. Manter updated_at atualizado em todas as modificações
2. Fazer backup antes de alterações em produção
3. Versionar mudanças de schema
4. Documentar novos atributos adicionados

## Monitoramento

### Métricas CloudWatch

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name ConsumedReadCapacityUnits \
  --dimensions Name=TableName,Value=oms_config_prod \
  --start-time 2025-11-14T00:00:00Z \
  --end-time 2025-11-14T23:59:59Z \
  --period 3600 \
  --statistics Sum
```

### Criar Alarme

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name oms-config-high-read-capacity \
  --alarm-description "Alerta de alta leitura na config table" \
  --metric-name ConsumedReadCapacityUnits \
  --namespace AWS/DynamoDB \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 1000 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=TableName,Value=oms_config_prod
```

## Backup e Recovery

### Criar Backup

```bash
aws dynamodb create-backup \
  --table-name oms_config_prod \
  --backup-name oms-config-backup-14-11-2025
```

### Restaurar de Backup

```bash
aws dynamodb restore-table-from-backup \
  --target-table-name oms_config_restored \
  --backup-arn arn:aws:dynamodb:us-east-1:123456789:table/oms_config_prod/backup/xxx
```

### Export para S3

```bash
aws dynamodb export-table-to-point-in-time \
  --table-arn arn:aws:dynamodb:us-east-1:123456789:table/oms_config_prod \
  --s3-bucket my-backup-bucket \
  --s3-prefix oms-config-export/ \
  --export-format DYNAMODB_JSON
```

## Troubleshooting

### Erro: ConditionalCheckFailedException

Ocorre quando uma condição no update/put falha.

Solução:
- Verificar se o item existe
- Verificar condições no update-expression

### Erro: ValidationException

Ocorre quando a estrutura do item não é válida.

Solução:
- Verificar tipos de dados
- Validar formato das chaves

### Performance Lenta

Possíveis causas:
- Usando Scan ao invés de Query/GetItem
- Falta de cache
- Hot partitions

Solução:
- Implementar cache (Redis/ElastiCache)
- Usar Query/GetItem quando possível
- Revisar estratégia de particionamento
