# VPC Templates - OMS Spider

Templates CloudFormation para criação de VPCs segregadas para o OMS Spider.

## Estratégia de Segregação

### 📦 VPCs Disponíveis

1. **vpc-nonprod.json** - Ambientes dev e staging
   - CIDR: `10.10.0.0/16` (dev) ou `10.11.0.0/16` (staging)
   - Suporta single-AZ (dev) ou multi-AZ (staging)
   - 1-2 NAT Gateways conforme configuração

2. **vpc-prod.json** - Ambiente de produção
   - CIDR: `10.20.0.0/16`
   - Sempre multi-AZ (mínimo 2 AZs, suporta 3)
   - 2 NAT Gateways obrigatórios
   - Enhanced monitoring e compliance

## Arquitetura de Rede

### Subnets por VPC

Cada VPC contém:

```
VPC (10.x.0.0/16)
├── Public Subnets (2-3 AZs)
│   ├── Internet Gateway
│   ├── NAT Gateway 1 (Elastic IP 1) ⭐
│   └── NAT Gateway 2 (Elastic IP 2) ⭐
│
├── Private Subnets - OMS Cluster (2-3 AZs)
│   └── Rotas via NAT Gateway
│
└── Private Subnets - Frontend/WebSocket (2-3 AZs)
    └── Rotas via NAT Gateway
```

**⭐ IPs Críticos**: Os Elastic IPs do NAT Gateway devem ser whitelistados nas exchanges.

## Deploy

### Non-Production (Dev)

```bash
aws cloudformation create-stack \
  --stack-name oms-vpc-dev \
  --template-body file://vpc-nonprod.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=VpcCIDR,ParameterValue=10.10.0.0/16 \
    ParameterKey=AvailabilityZone1,ParameterValue=us-east-1a \
    ParameterKey=AvailabilityZone2,ParameterValue=us-east-1b \
    ParameterKey=EnableMultiAZ,ParameterValue=false \
  --capabilities CAPABILITY_IAM
```

### Non-Production (Staging)

```bash
aws cloudformation create-stack \
  --stack-name oms-vpc-staging \
  --template-body file://vpc-nonprod.json \
  --parameters \
    ParameterKey=Environment,ParameterValue=staging \
    ParameterKey=VpcCIDR,ParameterValue=10.11.0.0/16 \
    ParameterKey=AvailabilityZone1,ParameterValue=us-east-1a \
    ParameterKey=AvailabilityZone2,ParameterValue=us-east-1b \
    ParameterKey=EnableMultiAZ,ParameterValue=true \
  --capabilities CAPABILITY_IAM
```

### Production

```bash
aws cloudformation create-stack \
  --stack-name oms-vpc-prod \
  --template-body file://vpc-prod.json \
  --parameters \
    ParameterKey=VpcCIDR,ParameterValue=10.20.0.0/16 \
    ParameterKey=AvailabilityZone1,ParameterValue=us-east-1a \
    ParameterKey=AvailabilityZone2,ParameterValue=us-east-1b \
    ParameterKey=AvailabilityZone3,ParameterValue=us-east-1c \
  --capabilities CAPABILITY_IAM
```

## 🔑 Outputs Importantes

Após o deploy, os seguintes outputs estarão disponíveis:

### Networking
- `VPCId` - ID da VPC
- `InternetGatewayId` - ID do Internet Gateway

### Subnets
- `PublicSubnetAZ1`, `PublicSubnetAZ2` - Subnets públicas
- `PrivateSubnetOMSAZ1`, `PrivateSubnetOMSAZ2` - Subnets privadas OMS
- `PrivateSubnetFrontendAZ1`, `PrivateSubnetFrontendAZ2` - Subnets privadas Frontend

### ⚠️ CRÍTICO - IPs para Whitelist
- `ElasticIPNAT1` - **Primeiro IP para whitelist nas exchanges**
- `ElasticIPNAT2` - **Segundo IP para whitelist nas exchanges**

### Obter IPs após Deploy

```bash
# Dev
aws cloudformation describe-stacks \
  --stack-name oms-vpc-dev \
  --query 'Stacks[0].Outputs[?OutputKey==`ElasticIPNAT1`].OutputValue' \
  --output text

# Prod
aws cloudformation describe-stacks \
  --stack-name oms-vpc-prod \
  --query 'Stacks[0].Outputs[?contains(OutputKey,`ElasticIP`)].{Key:OutputKey,IP:OutputValue}' \
  --output table
```

## Configuração de Exchanges

Após o deploy, você deve configurar os IPs nas exchanges:

### Bitget
1. Acesse Developer Settings → API Management
2. Adicione os 2 Elastic IPs à whitelist
3. Salve e aguarde propagação (~5 min)

### OKX
1. Acesse API → API Keys
2. Configure IP Whitelist
3. Adicione ambos os IPs

### BingX
1. Acesse API Management
2. Whitelist IP Configuration
3. Adicione os 2 IPs

## Características por Ambiente

### Dev (vpc-nonprod.json)
- ✅ Single-AZ (1 NAT Gateway)
- ✅ CIDR: 10.10.0.0/16
- ✅ VPC Flow Logs: 7 dias retenção
- ✅ Custo otimizado

### Staging (vpc-nonprod.json)
- ✅ Multi-AZ (2 NAT Gateways)
- ✅ CIDR: 10.11.0.0/16
- ✅ VPC Flow Logs: 14 dias retenção
- ✅ Simula produção

### Production (vpc-prod.json)
- ✅ Multi-AZ obrigatório (2-3 AZs)
- ✅ CIDR: 10.20.0.0/16
- ✅ VPC Flow Logs: 30 dias retenção
- ✅ Tags "DoNotDelete" nos Elastic IPs
- ✅ Enhanced monitoring
- ✅ Opcional: 3ª AZ para maior HA

## Custos Estimados (Mensais)

### Dev (Single-AZ)
- NAT Gateway: ~$35
- Data Processing: ~$5-15
- VPC Flow Logs: ~$2
- **Total**: ~$42-52/mês

### Staging (Multi-AZ)
- NAT Gateways (2x): ~$70
- Data Processing: ~$10-20
- VPC Flow Logs: ~$3
- **Total**: ~$83-93/mês

### Production (Multi-AZ)
- NAT Gateways (2x): ~$70
- Data Processing: ~$20-40
- VPC Flow Logs: ~$5
- Enhanced Monitoring: ~$5
- **Total**: ~$100-120/mês

## Segurança

### VPC Flow Logs
Todas as VPCs têm Flow Logs habilitados:
- **Dev**: 7 dias CloudWatch
- **Staging**: 14 dias CloudWatch
- **Prod**: 30 dias CloudWatch

### Network ACLs
- Default ACLs permitem todo tráfego
- Customizar conforme necessidade de compliance

### Security Groups
- Criar Security Groups específicos para cada workload
- Princípio do menor privilégio

## Troubleshooting

### NAT Gateway não funciona
```bash
# Verificar se NAT Gateway está ativo
aws ec2 describe-nat-gateways \
  --filter "Name=vpc-id,Values=<VPC_ID>" \
  --query 'NatGateways[*].[NatGatewayId,State]'

# Verificar Route Tables
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=<VPC_ID>" \
  --query 'RouteTables[*].Routes'
```

### IPs não estão funcionando nas exchanges
1. Confirme que os IPs foram whitelistados corretamente
2. Verifique que as instâncias estão nas private subnets
3. Confirme que o NAT Gateway está na route table
4. Teste conectividade:
   ```bash
   # De dentro da instância
   curl -s ifconfig.me
   # Deve retornar um dos Elastic IPs
   ```

### Stack creation falhou
- Verifique os limites de Elastic IPs na conta
- Confirme disponibilidade de AZs
- Verifique permissões IAM

## Próximos Passos

Após criar a VPC:

1. ✅ Anote os Elastic IPs (ElasticIPNAT1 e ElasticIPNAT2)
2. ✅ Configure whitelist nas exchanges
3. ➡️ Deploy Security Groups (próximo template)
4. ➡️ Deploy ECS Clusters
5. ➡️ Deploy Application Load Balancer

## Referências

- [AWS VPC Best Practices](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-best-practices.html)
- [NAT Gateway Pricing](https://aws.amazon.com/vpc/pricing/)
- [VPC Flow Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html)
