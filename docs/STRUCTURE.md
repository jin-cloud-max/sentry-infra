# Estrutura do Projeto OMS Infrastructure

```
cloudformation/
│
├── README.md                           # Documentação principal
├── STRUCTURE.md                        # Este arquivo
│
├── templates/                          # Templates CloudFormation organizados por serviço
│   ├── cognito/
│   │   └── cognito-user-pool.json     # User Pool com MFA e grupos de acesso
│   │
│   ├── dynamodb/
│   │   ├── oms-config-table.json      # Tabela de configurações (produtos, clientes)
│   │   └── oms-trading-table.json     # Tabela de dados transacionais (ordens, execuções, posições)
│   │
│   └── cicd/
│       ├── codepipeline.json          # Pipeline com GitHub OAuth Token
│       └── codepipeline-codestar.json # Pipeline com CodeStar Connection (Recomendado)
│
├── data/                               # Dados de exemplo e seed
│   └── ANALYST-DATA-TRANSFORMED.json  # Produtos com credenciais de analista
│
├── examples/                           # Exemplos de configuração
│   ├── buildspec.yml                  # Build básico
│   ├── buildspec-with-tests.yml       # Build com testes
│   ├── buildspec-multistage.yml       # Build otimizado com cache
│   ├── Dockerfile.example             # Dockerfile multi-stage
│   └── .dockerignore.example          # Otimização de build
│
└── docs/                               # Documentação técnica completa
    ├── README.md                       # Índice da documentação
    │
    ├── general/                        # Documentação geral
    │   ├── architecture.md             # Diagramas e arquitetura da infraestrutura
    │   └── deployment.md               # Guia completo de deploy e operações
    │
    ├── cognito/                        # Documentação do Cognito
    │   ├── overview.md                 # Visão geral do serviço de autenticação
    │   └── setup-guide.md              # Guia de configuração e integração
    │
    ├── dynamodb/                       # Documentação do DynamoDB
    │   ├── overview.md                 # Visão geral das tabelas
    │   ├── config-table.md             # Estrutura e queries da Config Table
    │   └── trading-table.md            # Estrutura e queries da Trading Table
    │
    └── cicd/                           # Documentação CI/CD
        ├── overview.md                 # Visão geral da pipeline
        ├── setup-guide.md              # Deploy com GitHub Token
        ├── codestar-connection.md      # Deploy com CodeStar Connection
        └── quickstart-codestar.md      # Quick start com CodeStar
```

## Resumo dos Componentes

### Templates CloudFormation

#### Cognito
- **cognito-user-pool.json**: Autenticação com MFA obrigatório, 3 grupos (admin, operator, viewer)

#### DynamoDB
- **oms-config-table.json**: Armazena produtos e clientes (single-table design simples)
- **oms-trading-table.json**: Armazena ordens, execuções e posições (single-table design com 3 GSIs)

#### CI/CD
- **codepipeline.json**: Pipeline com GitHub OAuth Token (deploy automatizado)
- **codepipeline-codestar.json**: Pipeline com CodeStar Connection (recomendado para produção)

### Documentação

#### Geral
- **architecture.md**: Diagramas Mermaid da infraestrutura, fluxos de autenticação e dados
- **deployment.md**: Guia passo a passo para deploy em dev/staging/prod

#### Cognito
- **overview.md**: Características, políticas, grupos de acesso
- **setup-guide.md**: Deploy, gerenciamento de usuários, integração com Next.js

#### DynamoDB
- **overview.md**: Visão geral das duas tabelas, padrões de acesso, custos
- **config-table.md**: Modelo de dados, queries, exemplos de uso
- **trading-table.md**: Modelo de dados, GSIs, TTL, queries complexas

#### CI/CD
- **overview.md**: Visão geral da pipeline, componentes, diagramas
- **setup-guide.md**: Deploy com GitHub Token
- **codestar-connection.md**: Deploy com CodeStar Connection (recomendado)
- **quickstart-codestar.md**: Guia rápido para deploy

## Padrões Utilizados

### Nomenclatura
- **Arquivos**: kebab-case (ex: oms-config-table.json)
- **Recursos AWS**: PascalCase (ex: OMSUserPool)
- **Chaves DynamoDB**: UPPERCASE com separador # (ex: PRODUCT#1)
- **Atributos**: snake_case (ex: entity_type)

### Formato de Data
- **Armazenamento**: DD/MM/YYYY HH:mm:ss
- **Chaves DynamoDB**: ISO 8601 (YYYY-MM-DDTHH:mm:ssZ)

### Linguagem
- **Templates**: JSON (CloudFormation)
- **Documentação**: Markdown com diagramas Mermaid

## Navegação Rápida

### Deploy Rápido
```bash
# Ver README.md na raiz para comandos de deploy
```

### Documentação por Tópico
- Arquitetura geral → [docs/general/architecture.md](docs/general/architecture.md)
- Como fazer deploy → [docs/general/deployment.md](docs/general/deployment.md)
- Configurar autenticação → [docs/cognito/setup-guide.md](docs/cognito/setup-guide.md)
- Estrutura de dados → [docs/dynamodb/overview.md](docs/dynamodb/overview.md)
- Pipeline CI/CD → [docs/cicd/overview.md](docs/cicd/overview.md)
- Deploy com CodeStar → [docs/cicd/quickstart-codestar.md](docs/cicd/quickstart-codestar.md)

### Templates por Serviço
- Cognito → [templates/cognito/](templates/cognito/)
- DynamoDB → [templates/dynamodb/](templates/dynamodb/)
- CI/CD → [templates/cicd/](templates/cicd/)
