# Fluxo 2 - Checkout Hotmart → Keycloak: Gerenciamento de Acesso

Gerenciar o ciclo de vida de acesso de assinantes da Hotmart à comunidade Circle, criando/atualizando usuários no Keycloak e adicionando/removendo do grupo **"Assinantes Circle"** baseado em eventos de compra aprovada ou cancelamento.

## Objetivo e Contexto

### Controle de Acesso via Grupos no Keycloak

O **Keycloak** é nosso **provedor de identidade e autorização**. Ele controla quem pode acessar a comunidade Circle através de:

- **Grupos**: Agregações de usuários com permissões comuns
- **Grupo "Assinantes Circle"**: Aqueles com acesso ativo à comunidade
- **SSO (Single Sign-On)**: Autenticação centralizada com token JWT

**Fluxo de autorização:**
```
Usuário loga via SSO Keycloak
  ↓
Keycloak verifica: está no grupo "Assinantes Circle"?
  ├─ SIM → Gera JWT com acesso
  └─ NÃO → Nega acesso ao Circle
```

**Este fluxo garante:** Quando alguém compra na Hotmart, ele é automaticamente adicionado ao grupo (ativação de acesso). Quando cancela, é removido (revogação de acesso).

### Eventos Suportados

| Evento Hotmart | Ação | Resultado |
|----|----|----|
| `PURCHASE_APPROVED` | Criar usuário OU conceder acesso | Usuário adicionado ao grupo Assinantes Circle |
| `SUBSCRIPTION_CANCELLATION` | Remover acesso | Usuário removido do grupo Assinantes Circle |

## Macro Arquitetura

| Componente | Responsabilidade | Ferramenta |
|----|----|----| 
| **Webhook** | Receber eventos de checkout | Hotmart (POST → N8N) |
| **Extração de dados** | Normalizar email e evento | N8N (Code) |
| **Autenticação** | Obter token de acesso | Keycloak API |
| **Busca de usuário** | Verificar se usuário já existe | Keycloak API |
| **Roteamento** | Direcionar para ação correta | N8N (Switch) |
| **Gerenciamento de grupo** | Adicionar/remover do grupo | Keycloak API |
| **Convite de acesso** | Enviar primeiro acesso | Keycloak Magic Link |

## Fluxo Técnico Detalhado

### 1. Webhook: Receber Eventos Hotmart

**URL de entrada:**
```
POST https://n8n.bonde.org/webhook/checkout/auth
```

**Eventos capturados:**
```json
{
  "event": "PURCHASE_APPROVED" | "SUBSCRIPTION_CANCELLATION",
  "data": {
    "buyer": {
      "email": "cliente@example.com",
      "first_name": "João",
      "last_name": "Silva",
      "checkout_phone": "...",
      "address": { ... }
    },
    "subscriber": {
      "email": "cliente@example.com"
    },
    "subscription": {
      "status": "ACTIVE"
    },
    "purchase": {
      "recurrence_number": 1,
      "approved_date": 1712345678000,
      "status": "APPROVED"
    }
  }
}
```

### 2. Normalização: Extrair Dados Relevantes

**O que faz:** Padroniza o payload para facilitar processamento nos próximos nós

**Lógica (JavaScript):**
```javascript
const event = webhook.body.event;
const data = webhook.body.data;

if (event === 'SUBSCRIPTION_CANCELLATION') {
  email = data.subscriber.email; // Cancelamento usa subscriber
} else if (event === 'PURCHASE_APPROVED') {
  email = data.buyer.email;      // Compra usa buyer
}

return {
  email: email,
  event: event
};
```

**Saída:**
```json
{
  "email": "cliente@example.com",
  "event": "PURCHASE_APPROVED"
}
```

### 3. Autenticação: Obter Token Keycloak

**Endpoint:**
```
POST https://auth.bonde.org/realms/bonde/protocol/openid-connect/token
```

**Credenciais (Client Credentials Flow):**
```json
{
  "grant_type": "client_credentials",
  "client_id": "n8n",
  "client_secret": "GZIT95PlJdCe61iV1yivmvMmhqfCtHV9"
}
```

**Resposta:**
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldWIiwia2lkIiA6ICJ...",
  "expires_in": 300,
  "token_type": "Bearer"
}
```

**Por quê:** Token permite que N8N execute ações no Keycloak como autoridade administrativa.

### 4. Buscar Usuário: Verificar Existência

**Endpoint:**
```
GET https://auth.bonde.org/admin/realms/bonde/users?email={{email}}&exact=true
```

**Headers:**
```
Authorization: Bearer {{ access_token }}
```

**Resposta (usuário existe):**
```json
[
  {
    "id": "17896b50-8e7a-4715-b217-3caf12f090de",
    "email": "cliente@example.com",
    "firstName": "João",
    "lastName": "Silva",
    "enabled": true,
    "groups": ["/Assinantes Circle"]
  }
]
```

**Resposta (usuário não existe):**
```json
[]
```

### 5. Roteamento: Pipeline com 5 Saídas

**Lógica do Switch:**

```
IF event == "SUBSCRIPTION_CANCELLATION" AND usuário existe
  → Output 0: Remover Acesso
ELSE IF event == "SUBSCRIPTION_CANCELLATION" AND usuário NÃO existe
  → Output 1: Ignorar (usuário já removido ou nunca existiu)
ELSE IF event == "PURCHASE_APPROVED" AND usuário existe
  → Output 2: Conceder Acesso (adicionar a grupo)
ELSE IF event == "PURCHASE_APPROVED" AND usuário NÃO existe
  → Output 3: Criar Usuário (novo cliente)
ELSE
  → Output 4: Evento inválido (ignorar)
```

---

### 6A. Caminho: Remover Acesso (Output 0)

**Quando disparado:** Cliente cancelou assinatura e já tinha usuário no Keycloak

**Endpoint:**
```
DELETE https://auth.bonde.org/admin/realms/bonde/users/{{ user_id }}/groups/f8a91e89-f8dc-459d-9c59-3b1aaf945b28
```

**Parâmetros:**
| Parâmetro | Valor |
|----|----|
| `user_id` | ID do usuário Keycloak |
| `group_id` | `f8a91e89-f8dc-459d-9c59-3b1aaf945b28` (Assinantes Circle) |
| `Authorization` | Bearer {{ access_token }} |

**Resultado:**
```
Status: 204 No Content
```

**O que acontece:** Usuário é removido do grupo "Assinantes Circle" → próximo SSO nega acesso ao Circle

---

### 6B. Caminho: Conceder Acesso (Output 2)

**Quando disparado:** Cliente tem assinatura ativa e já tinha usuário no Keycloak

**Endpoint:**
```
PUT https://auth.bonde.org/admin/realms/bonde/users/{{ user_id }}/groups/f8a91e89-f8dc-459d-9c59-3b1aaf945b28
```

**Parâmetros:**
| Parâmetro | Valor |
|----|----|
| `user_id` | ID do usuário Keycloak |
| `group_id` | `f8a91e89-f8dc-459d-9c59-3b1aaf945b28` (Assinantes Circle) |
| `Authorization` | Bearer {{ access_token }} |

**Resultado:**
```
Status: 204 No Content
```

**O que acontece:** Usuário é garantidamente membro do grupo → próximo SSO permite acesso ao Circle

---

### 6C. Caminho: Criar Usuário (Output 3)

**Quando disparado:** Cliente comprou pela primeira vez e não tem usuário no Keycloak

**Passo 1: Criar usuário**

Endpoint:
```
POST https://auth.bonde.org/admin/realms/bonde/users
```

Payload:
```json
{
  "email": "{{ buyer.email }}",
  "username": "{{ buyer.email }}",
  "emailVerified": true,
  "enabled": true,
  "firstName": "{{ buyer.first_name }}",
  "lastName": "{{ buyer.last_name }}",
  "groups": ["/Assinantes Circle"],
  "requiredActions": ["UPDATE_PASSWORD"]
}
```

**Campos:**
| Campo | Descrição |
|----|----| 
| `email` | Email do cliente Hotmart |
| `username` | Mesmo que email (identificador único) |
| `emailVerified` | Marca como verificado (sem confirmar por email) |
| `enabled` | Ativa o usuário imediatamente |
| `firstName`, `lastName` | Dados do perfil |
| `groups` | Adiciona ao grupo Assinantes Circle já na criação |
| `requiredActions` | Obriga mudança de senha no primeiro acesso |

**Resposta (sucesso):**
```
Status: 201 Created
Location: /admin/realms/bonde/users/{{ user_id }}
```

**Passo 2: Enviar Magic Link**

Endpoint:
```
POST https://auth.bonde.org/realms/bonde/magic-link
```

Payload:
```json
{
  "email": "{{ buyer.email }}",
  "client_id": "circle",
  "expiration_seconds": 3600,
  "redirect_uri": "https://comunidade.bonde.org",
  "send_email": true
}
```

**Campos:**
| Campo | Descrição |
|----|----| 
| `email` | Destinatário do email com link |
| `client_id` | Aplicação (Circle) para qual gerar link |
| `expiration_seconds` | Link válido por 1 hora |
| `redirect_uri` | Para onde redirecionar após login |
| `send_email` | Envia email com link automaticamente |

**Email recebido pelo cliente:**
```
Bem-vindo à Comunidade BONDE!

Clique aqui para acessar sua comunidade: 
https://auth.bonde.org/magic-link/eyJhbGc...
```

**Resultado:** Cliente recebe email com link de primeiro acesso, clica e é redirecionado para Circle

---

## Resultado Esperado

### Estado Antes do Fluxo (Compra)
- ✗ Compra confirmada na Hotmart
- ✗ Sem usuário no Keycloak
- ✗ Sem acesso à comunidade Circle

### Estado Depois do Fluxo (Compra - Novo Usuário)
- ✅ Usuário criado no Keycloak
- ✅ Adicionado ao grupo "Assinantes Circle"
- ✅ Email recebido com Magic Link
- ✅ Primeiro acesso requer mudar senha
- ✅ Após login, tem acesso total ao Circle

### Estado Antes do Fluxo (Cancelamento)
- ✗ Assinatura cancelada na Hotmart
- ✓ Usuário ainda existe no Keycloak
- ✓ Usuário ainda está no grupo Assinantes Circle

### Estado Depois do Fluxo (Cancelamento)
- ✅ Usuário removido do grupo "Assinantes Circle"
- ✅ Próximo login SSO → Keycloak nega acesso
- ✅ Usuário vê erro ao tentar acessar Circle
- ℹ️ Conta continue existindo (para histórico)

---

## Mapeamento de Groups no Keycloak

| Nome do Grupo | ID | Propósito |
|----|----|----| 
| Assinantes Circle | `f8a91e89-f8dc-459d-9c59-3b1aaf945b28` | Autorização para acessar Circle |
| (outros grupos) | ... | Para expansão futura |

**Note:** Esses IDs são específicos da configuração Keycloak. Se estrutura de grupos mudar, esses IDs precisarão ser atualizados no fluxo.

---

## Possíveis Cenários de Erro

| Erro | Causa | Resolução |
|----|----|----|
| Email inválido | Hotmart enviou email malformado | Webhook rejeita |
| Usuário duplicado | Email já existe no Keycloak | N8N trata como "usuário existe" → concede acesso |
| Falha ao criar usuário | Senha requisitada ou erro de permissão | Verificar credenciais N8N no Keycloak |
| Magic Link não chega | Email configurado incorretamente em Keycloak | Usuário pode fazer reset de senha manualmente |
| Evento não reconhecido | Hotmart enviou evento inesperado | Fluxo ignora (Output 4) |
| Falha ao remover do grupo | Usuário já não está no grupo | Operação é idempotente (sucesso/silenciosa) |

---

## Conceitos-Chave

### Client Credentials Flow

Tipo de autenticação OAuth2 para aplicações servidor-a-servidor:
```
N8N enviar credenciais (client_id + client_secret)
  ↓
Keycloak valida
  ↓
Keycloak emite token temporal
  ↓
N8N usa token em requisições subsequentes
```

**Vantagem:** Não requer interação do usuário, ideal para integrações.

### Grupos como Autorização

Em vez de usar roles (papéis), usamos **grupos** porque:
- Mais fácil gerenciar dinamicamente
- Keycloak sincroniza grupos → Circle via SSO
- Um usuário pode estar em múltiplos grupos no futuro

### Email como Chave Primária

Email é único e imutável → garantia de identificação consistente:
- Ao buscar usuário, sempre usa email
- Ao criar usuário, email é username também
- Se cliente trocasse de email, criaria duplicata (cuidado!)

### Magic Link vs Enviar Senha

**Magic Link (implementado):**
- ✅ Não expõe senha por email
- ✅ First-time setup seguro
- ✅ Redireção direta para Circle

**Enviar Senha (não implementado):**
- ✗ Menos seguro
- ✗ Cliente precisa fazer reset mesmo assim

### RequiredActions

Campo `requiredActions: ["UPDATE_PASSWORD"]` força:
- Na primeira login, Keycloak redireciona para trocar senha
- Garante que cliente tem senha única e forte
- Previne que N8N armazene senhas

---

## Fluxos Relacionados

- ✅ **Fluxo 1** (Enriquecimento de Base AC): Atualiza dados do cliente em paralelo
- ✅ **Fluxo 3** (Ativação de Conta Keycloak + Circle): O que acontece DEPOIS que usuário está no grupo
- ⏳ **Fluxo 4** (Desativação): Remove acesso quando cancelamento
- ⏳ **Fluxo 5** (Sincronização de Perfil): Atualiza profilename/avatar Circle ↔ Keycloak

---

## Checklist de Implementação

- [ ] Webhook URL configurada na Hotmart
- [ ] Credenciais N8N registradas no Keycloak (client_id + client_secret)
- [ ] Group ID verificado (`f8a91e89-f8dc-459d-9c59-3b1aaf945b28`)
- [ ] Email de Magic Link configurado no Keycloak
- [ ] Teste: Nova compra → usuário criado e recebe magic link
- [ ] Teste: Cancelamento → usuário removido do grupo
- [ ] Monitoramento de erros no N8N ativado
