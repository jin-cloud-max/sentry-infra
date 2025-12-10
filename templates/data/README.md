# Data Layer Templates

Templates CloudFormation para camada de dados do OMS Spider (ElastiCache Redis e DynamoDB).

## Templates Disponíveis

### 1. ElastiCache Redis (`elasticache-redis.json`)

Cluster Redis para cache de:
- Mapeamentos de ordens (order mappings)
- Posições (positions)
- Instrumentos (instruments)
- Idempotência de requisições
- Leader election (para HA)

## Deployment

### ElastiCache Redis

#### Development

```bash
aws cloudformation create-stack \
  --stack-name oms-redis-dev \
  --template-body file://templates/data/elasticache-redis.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=VPCStackName,ParameterValue=oms-vpc-dev \
    ParameterKey=NodeType,ParameterValue=cache.t4g.micro \
    ParameterKey=NumCacheNodes,ParameterValue=1 \
    ParameterKey=EngineVersion,ParameterValue=7.1 \
    ParameterKey=SnapshotRetentionLimit,ParameterValue=0 \
    ParameterKey=AutomaticFailoverEnabled,ParameterValue=false \
  --region us-east-1
```

**Configuração Dev:**
- **Node Type**: `cache.t4g.micro` (512 MB RAM)
- **Custo**: ~$11/mês
- **Nodes**: 1 (sem replicas)
- **Backups**: Desabilitados
- **Failover**: Desabilitado
- **Encryption**: Desabilitada

#### Staging

```bash
aws cloudformation create-stack \
  --stack-name oms-redis-staging \
  --template-body file://templates/data/elasticache-redis.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=staging \
    ParameterKey=VPCStackName,ParameterValue=oms-vpc-staging \
    ParameterKey=NodeType,ParameterValue=cache.t4g.small \
    ParameterKey=NumCacheNodes,ParameterValue=2 \
    ParameterKey=EngineVersion,ParameterValue=7.1 \
    ParameterKey=SnapshotRetentionLimit,ParameterValue=3 \
    ParameterKey=AutomaticFailoverEnabled,ParameterValue=true \
  --region us-east-1
```

**Configuração Staging:**
- **Node Type**: `cache.t4g.small` (1.5 GB RAM)
- **Custo**: ~$44/mês
- **Nodes**: 2 (1 primary + 1 replica)
- **Backups**: 3 dias
- **Failover**: Habilitado
- **Multi-AZ**: Sim

#### Production

```bash
aws cloudformation create-stack \
  --stack-name oms-redis-prod \
  --template-body file://templates/data/elasticache-redis.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=prod \
    ParameterKey=VPCStackName,ParameterValue=oms-vpc-prod \
    ParameterKey=NodeType,ParameterValue=cache.m7g.large \
    ParameterKey=NumCacheNodes,ParameterValue=3 \
    ParameterKey=EngineVersion,ParameterValue=7.1 \
    ParameterKey=SnapshotRetentionLimit,ParameterValue=7 \
    ParameterKey=AutomaticFailoverEnabled,ParameterValue=true \
  --region us-east-1
```

**Configuração Production:**
- **Node Type**: `cache.m7g.large` (6.38 GB RAM)
- **Custo**: ~$330/mês
- **Nodes**: 3 (1 primary + 2 replicas)
- **Backups**: 7 dias
- **Failover**: Habilitado
- **Multi-AZ**: Sim
- **Encryption**: At-rest e in-transit habilitadas

## Node Types Disponíveis

### T4g - Burstable (Dev/Staging)

| Node Type | vCPU | RAM | Baseline | Custo/mês | Uso Recomendado |
|-----------|------|-----|----------|-----------|-----------------|
| cache.t4g.micro | 2 | 512 MB | 10% | ~$11 | Dev - cache leve |
| cache.t4g.small | 2 | 1.5 GB | 20% | ~$22 | Dev - cache médio |
| cache.t4g.medium | 2 | 3.1 GB | 20% | ~$44 | Staging |

### M7g - General Purpose (Production)

| Node Type | vCPU | RAM | Network | Custo/mês | Uso Recomendado |
|-----------|------|-----|---------|-----------|-----------------|
| cache.m7g.large | 2 | 6.38 GB | 12.5 Gbps | ~$110 | Prod - small |
| cache.m7g.xlarge | 4 | 12.93 GB | 12.5 Gbps | ~$220 | Prod - medium |

### R7g - Memory Optimized (High Memory)

| Node Type | vCPU | RAM | Network | Custo/mês | Uso Recomendado |
|-----------|------|-----|---------|-----------|-----------------|
| cache.r7g.large | 2 | 13.07 GB | 12.5 Gbps | ~$150 | Prod - memory intensive |
| cache.r7g.xlarge | 4 | 26.32 GB | 12.5 Gbps | ~$300 | Prod - high memory |

## Redis Configuration

### Parameter Group Settings

O template cria um parameter group customizado com:

```
maxmemory-policy: allkeys-lru
  → Remove keys menos recentes quando memória cheia
  → Ideal para cache de dados temporários

timeout: 300
  → Fecha conexões idle após 5 minutos

tcp-keepalive: 300
  → Mantém conexões TCP ativas

maxmemory-samples: 5
  → Número de keys analisadas para eviction LRU
```

### Engine Versions

- **Redis 7.1** (Recomendado)
  - Redis Functions
  - Melhor performance
  - Active-Active Geo replication support

- **Redis 7.0**
  - Redis Functions (experimental)
  - Melhor que 6.2 em performance

- **Redis 6.2**
  - Estável
  - ACLs melhorados

## Uso no Código

### Connection String

Após deploy, obtenha o endpoint:

```bash
aws cloudformation describe-stacks \
  --stack-name oms-redis-dev \
  --query 'Stacks[0].Outputs[?OutputKey==`RedisEndpoint`].OutputValue' \
  --output text
```

### Go Redis Client

```go
import "github.com/go-redis/redis/v8"

// Single node (dev)
client := redis.NewClient(&redis.Options{
    Addr: "oms-redis-dev.abc123.0001.use1.cache.amazonaws.com:6379",
    DB: 0,
})

// Cluster com replicas (prod)
client := redis.NewClient(&redis.Options{
    Addr: "oms-redis-prod.abc123.ng.0001.use1.cache.amazonaws.com:6379",
    DB: 0,
    ReadTimeout: 3 * time.Second,
    WriteTimeout: 3 * time.Second,
    PoolSize: 100,
    MinIdleConns: 10,
})
```

### Environment Variables

Configurar no ECS Task Definition:

```json
{
  "name": "REDIS_HOST",
  "value": "oms-redis-dev.abc123.0001.use1.cache.amazonaws.com"
},
{
  "name": "REDIS_PORT",
  "value": "6379"
}
```

## Outputs do Template

| Output | Descrição | Export Name |
|--------|-----------|-------------|
| RedisEndpoint | Primary endpoint (write) | `${StackName}-RedisEndpoint` |
| RedisPort | Porta Redis (6379) | `${StackName}-RedisPort` |
| RedisReaderEndpoint | Reader endpoint (read replicas) | `${StackName}-RedisReaderEndpoint` |
| RedisSecurityGroupId | Security Group ID | `${StackName}-RedisSecurityGroupId` |
| RedisReplicationGroupId | Replication Group ID | `${StackName}-RedisReplicationGroupId` |

## Casos de Uso OMS

Baseado na estrutura do projeto, o Redis será usado para:

### 1. Order Mappings
```
Key: order:{exchange}:{exchangeOrderId}
Type: Hash
TTL: 24 horas
Tamanho estimado: 1-2 KB por ordem
```

### 2. Positions
```
Key: position:{clientId}:{symbol}
Type: Hash
TTL: Persist (até fechamento)
Tamanho estimado: 500 bytes por posição
```

### 3. Instruments
```
Key: instrument:{symbol}
Type: Hash
TTL: 1 hora
Tamanho estimado: 2-3 KB por instrumento
```

### 4. Idempotency
```
Key: idempotency:{requestId}
Type: String
TTL: 5 minutos
Tamanho estimado: 100 bytes
```

### 5. Leader Election
```
Key: leader:{service}
Type: String com SETNX
TTL: 30 segundos
```

## Estimativa de Memória

### Dev (cache.t4g.micro - 512 MB)
- 1.000 ordens ativas: ~2 MB
- 100 posições: ~50 KB
- 200 instrumentos: ~400 KB
- 500 idempotency keys: ~50 KB
- **Total usado**: ~3 MB
- **Overhead Redis**: ~40%
- **Capacidade disponível**: ~360 MB livre

✅ Suficiente para dev/testes

### Production (cache.m7g.large - 6.38 GB)
- 50.000 ordens ativas: ~100 MB
- 5.000 posições: ~2.5 MB
- 2.000 instrumentos: ~4 MB
- 10.000 idempotency keys: ~1 MB
- **Total usado**: ~107 MB
- **Overhead Redis**: ~40%
- **Capacidade disponível**: ~4.3 GB livre

✅ Excelente margem para crescimento

## Monitoramento

### CloudWatch Metrics

Métricas importantes para monitorar:

```bash
# CPU Utilization
aws cloudwatch get-metric-statistics \
  --namespace AWS/ElastiCache \
  --metric-name CPUUtilization \
  --dimensions Name=CacheClusterId,Value=oms-redis-dev-001 \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 3600 \
  --statistics Average

# Memory Usage
aws cloudwatch get-metric-statistics \
  --namespace AWS/ElastiCache \
  --metric-name DatabaseMemoryUsagePercentage \
  --dimensions Name=CacheClusterId,Value=oms-redis-dev-001 \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 3600 \
  --statistics Average
```

### Alarmes Recomendados

**Production:**
- CPUUtilization > 75%
- DatabaseMemoryUsagePercentage > 85%
- CacheHitRate < 80%
- Evictions > 1000/min

## Backup e Restore

### Manual Backup

```bash
aws elasticache create-snapshot \
  --replication-group-id oms-redis-prod \
  --snapshot-name oms-redis-backup-$(date +%Y%m%d)
```

### Restore from Snapshot

```bash
aws elasticache create-replication-group \
  --replication-group-id oms-redis-prod-restored \
  --replication-group-description "Restored from backup" \
  --snapshot-name oms-redis-backup-20240115
```

## Scaling

### Vertical Scaling (Change Node Type)

1. Modificar o parâmetro `NodeType` no CloudFormation
2. Update stack
3. Redis irá fazer failover automático (se multi-node)

### Horizontal Scaling (Add Replicas)

1. Modificar o parâmetro `NumCacheNodes`
2. Update stack
3. Novas replicas são adicionadas automaticamente

## Custos Estimados

| Ambiente | Config | Custo/mês | Detalhamento |
|----------|--------|-----------|--------------|
| **Dev** | t4g.micro x1 | ~$11 | $11 node + $0 backup |
| **Staging** | t4g.small x2 | ~$44 | $44 nodes + minimal backup |
| **Prod** | m7g.large x3 | ~$330 | $330 nodes + backup + transfer |

**Custos adicionais:**
- Backup storage: ~$0.085/GB-month (apenas storage usado)
- Data transfer out: ~$0.09/GB (tráfego para internet)
- Data transfer in: Grátis

## Troubleshooting

### Redis está lento

1. Verificar CPU: `aws cloudwatch get-metric-statistics ...`
2. Verificar memória: Checar evictions
3. Verificar conexões: `redis-cli info clients`
4. Considerar scaling vertical (node type maior)

### Conexão recusada

1. Verificar Security Group permite porta 6379
2. Verificar que aplicação está na mesma VPC
3. Verificar subnet group está correto
4. Testar com `telnet <endpoint> 6379`

### Alta eviction rate

1. Aumentar memória (node type maior)
2. Revisar TTLs das keys
3. Implementar cache warming strategy
4. Considerar Redis Cluster (sharding)

## Próximos Passos

1. ✅ Deploy do template Redis
2. ⏳ Configurar connection string no código Go
3. ⏳ Implementar health checks Redis
4. ⏳ Configurar CloudWatch alarms
5. ⏳ Testar failover (staging/prod)
