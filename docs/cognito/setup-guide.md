# Cognito - Guia de Configuração

## Template CloudFormation

[cognito-user-pool.json](../../templates/cognito/cognito-user-pool.json)

## Deploy do User Pool

### Parâmetros do Template

| Parâmetro | Tipo | Default | Descrição |
|-----------|------|---------|-----------|
| Environment | String | dev | Ambiente (dev/staging/prod) |
| AdminEmail | String | "" | Email do primeiro admin (opcional) |

### Deploy via CLI

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

### Aguardar Conclusão

```bash
aws cloudformation wait stack-create-complete \
  --stack-name oms-cognito-dev
```

### Obter Outputs

```bash
aws cloudformation describe-stacks \
  --stack-name oms-cognito-dev \
  --query 'Stacks[0].Outputs' \
  --output table
```

Exemplo de output:
```
---------------------------------------------------------------------------
|                           DescribeStacks                                |
+-------------------+-----------------------------------------------------+
|  CognitoRegion    |  us-east-1                                         |
|  GroupAdmin       |  oms-admin                                         |
|  GroupOperator    |  oms-operator                                      |
|  GroupViewer      |  oms-viewer                                        |
|  UserPoolArn      |  arn:aws:cognito-idp:us-east-1:123456789:userpool |
|  UserPoolClientId |  abc123def456ghi789jkl                             |
|  UserPoolId       |  us-east-1_XXXXXXXXX                               |
+-------------------+-----------------------------------------------------+
```

## Configuração Inicial

### 1. Criar Primeiro Admin

Se não foi criado automaticamente via parâmetro:

```bash
# Criar usuário
aws cognito-idp admin-create-user \
  --user-pool-id us-east-1_XXXXXXXXX \
  --username admin@example.com \
  --user-attributes \
    Name=email,Value=admin@example.com \
    Name=email_verified,Value=true \
    Name=custom:role,Value=admin \
  --desired-delivery-mediums EMAIL

# Adicionar ao grupo admin
aws cognito-idp admin-add-user-to-group \
  --user-pool-id us-east-1_XXXXXXXXX \
  --username admin@example.com \
  --group-name oms-admin
```

### 2. Configurar MFA

O usuário deve configurar MFA no primeiro login:

1. Fazer login com senha temporária
2. Criar nova senha
3. Escanear QR code com app autenticador (Google Authenticator, Authy, etc)
4. Inserir código de verificação

Apps autenticadores recomendados:
- Google Authenticator (iOS/Android)
- Microsoft Authenticator (iOS/Android)
- Authy (iOS/Android/Desktop)

### 3. Configurar Aplicação

Adicionar variáveis de ambiente na aplicação Next.js:

```env
# .env.local
NEXT_PUBLIC_AWS_REGION=us-east-1
NEXT_PUBLIC_COGNITO_USER_POOL_ID=us-east-1_XXXXXXXXX
NEXT_PUBLIC_COGNITO_CLIENT_ID=abc123def456ghi789jkl
```

## Gerenciamento de Usuários

### Criar Operador

```bash
# Criar usuário
aws cognito-idp admin-create-user \
  --user-pool-id us-east-1_XXXXXXXXX \
  --username operator@example.com \
  --user-attributes \
    Name=email,Value=operator@example.com \
    Name=email_verified,Value=true \
    Name=custom:role,Value=operator \
  --desired-delivery-mediums EMAIL

# Adicionar ao grupo
aws cognito-idp admin-add-user-to-group \
  --user-pool-id us-east-1_XXXXXXXXX \
  --username operator@example.com \
  --group-name oms-operator
```

### Criar Viewer

```bash
# Criar usuário
aws cognito-idp admin-create-user \
  --user-pool-id us-east-1_XXXXXXXXX \
  --username viewer@example.com \
  --user-attributes \
    Name=email,Value=viewer@example.com \
    Name=email_verified,Value=true \
    Name=custom:role,Value=viewer \
  --desired-delivery-mediums EMAIL

# Adicionar ao grupo
aws cognito-idp admin-add-user-to-group \
  --user-pool-id us-east-1_XXXXXXXXX \
  --username viewer@example.com \
  --group-name oms-viewer
```

### Listar Usuários de um Grupo

```bash
aws cognito-idp list-users-in-group \
  --user-pool-id us-east-1_XXXXXXXXX \
  --group-name oms-admin
```

### Resetar Senha

```bash
aws cognito-idp admin-reset-user-password \
  --user-pool-id us-east-1_XXXXXXXXX \
  --username user@example.com
```

O usuário receberá um email com nova senha temporária.

### Alterar Grupo do Usuário

```bash
# Remover do grupo atual
aws cognito-idp admin-remove-user-from-group \
  --user-pool-id us-east-1_XXXXXXXXX \
  --username user@example.com \
  --group-name oms-viewer

# Adicionar ao novo grupo
aws cognito-idp admin-add-user-to-group \
  --user-pool-id us-east-1_XXXXXXXXX \
  --username user@example.com \
  --group-name oms-operator

# Atualizar atributo role
aws cognito-idp admin-update-user-attributes \
  --user-pool-id us-east-1_XXXXXXXXX \
  --username user@example.com \
  --user-attributes Name=custom:role,Value=operator
```

## Integração com Next.js

### Instalação

```bash
npm install next-auth @aws-sdk/client-cognito-identity-provider
```

### Configuração NextAuth

```javascript
// pages/api/auth/[...nextauth].js
import NextAuth from 'next-auth';
import CognitoProvider from 'next-auth/providers/cognito';

export default NextAuth({
  providers: [
    CognitoProvider({
      clientId: process.env.NEXT_PUBLIC_COGNITO_CLIENT_ID,
      clientSecret: process.env.COGNITO_CLIENT_SECRET,
      issuer: `https://cognito-idp.${process.env.NEXT_PUBLIC_AWS_REGION}.amazonaws.com/${process.env.NEXT_PUBLIC_COGNITO_USER_POOL_ID}`,
    }),
  ],
  callbacks: {
    async jwt({ token, user, account }) {
      if (account && user) {
        token.accessToken = account.access_token;
        token.idToken = account.id_token;
        token.refreshToken = account.refresh_token;
        token.role = user['custom:role'];
      }
      return token;
    },
    async session({ session, token }) {
      session.accessToken = token.accessToken;
      session.idToken = token.idToken;
      session.role = token.role;
      return session;
    },
  },
  pages: {
    signIn: '/login',
    error: '/auth/error',
  },
});
```

### Página de Login

```javascript
// pages/login.jsx
import { signIn } from 'next-auth/react';
import { useState } from 'react';

export default function LoginPage() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  const handleLogin = async (e) => {
    e.preventDefault();
    const result = await signIn('cognito', {
      email,
      password,
      callbackUrl: '/dashboard',
    });
  };

  return (
    <form onSubmit={handleLogin}>
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Email"
      />
      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        placeholder="Senha"
      />
      <button type="submit">Login</button>
    </form>
  );
}
```

### Proteção de Rotas

```javascript
// middleware.js
import { withAuth } from 'next-auth/middleware';

export default withAuth({
  callbacks: {
    authorized: ({ token, req }) => {
      const path = req.nextUrl.pathname;

      // Admin only routes
      if (path.startsWith('/admin')) {
        return token?.role === 'admin';
      }

      // Operator and above
      if (path.startsWith('/trading')) {
        return ['admin', 'operator'].includes(token?.role);
      }

      // All authenticated users
      return !!token;
    },
  },
});

export const config = {
  matcher: ['/dashboard/:path*', '/admin/:path*', '/trading/:path*'],
};
```

## Testes

### Testar Login via CLI

```bash
# Iniciar autenticação
aws cognito-idp initiate-auth \
  --auth-flow USER_PASSWORD_AUTH \
  --client-id abc123def456ghi789jkl \
  --auth-parameters \
    USERNAME=user@example.com,PASSWORD=TempPassword123!
```

### Testar MFA

```bash
# Responder ao desafio MFA
aws cognito-idp respond-to-auth-challenge \
  --client-id abc123def456ghi789jkl \
  --challenge-name SOFTWARE_TOKEN_MFA \
  --session <session-from-initiate-auth> \
  --challenge-responses \
    USERNAME=user@example.com,SOFTWARE_TOKEN_MFA_CODE=123456
```

## Troubleshooting

### Erro: User pool client does not exist

Verificar se o Client ID está correto:
```bash
aws cognito-idp describe-user-pool-client \
  --user-pool-id us-east-1_XXXXXXXXX \
  --client-id abc123def456ghi789jkl
```

### Erro: Invalid username or password

Possíveis causas:
1. Credenciais incorretas
2. Usuário desabilitado
3. Usuário não existe

Verificar status do usuário:
```bash
aws cognito-idp admin-get-user \
  --user-pool-id us-east-1_XXXXXXXXX \
  --username user@example.com
```

### Erro: MFA code mismatch

Verificar:
1. Relógio do dispositivo está sincronizado
2. Código foi gerado recentemente
3. MFA está configurado corretamente

### Erro: Password does not conform to policy

A senha deve atender aos requisitos:
- Mínimo 12 caracteres
- Ao menos 1 letra maiúscula
- Ao menos 1 letra minúscula
- Ao menos 1 número
- Ao menos 1 caractere especial

## Manutenção

### Backup de Configuração

```bash
# Descrever User Pool
aws cognito-idp describe-user-pool \
  --user-pool-id us-east-1_XXXXXXXXX \
  > cognito-config-backup-DD-MM-YYYY.json

# Exportar lista de usuários
aws cognito-idp list-users \
  --user-pool-id us-east-1_XXXXXXXXX \
  > users-backup-DD-MM-YYYY.json
```

### Monitoramento

Criar alarme para falhas de login:

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name oms-cognito-signin-failures \
  --alarm-description "Alerta de falhas de login" \
  --metric-name SignInThrottles \
  --namespace AWS/Cognito \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions <SNS_TOPIC_ARN>
```

### Limpeza de Usuários Inativos

Listar usuários que não fizeram login nos últimos 90 dias:

```bash
aws cognito-idp list-users \
  --user-pool-id us-east-1_XXXXXXXXX \
  --filter "cognito:user_status = 'CONFIRMED'" \
  --query "Users[?UserLastModifiedDate<'2024-08-14']"
```
