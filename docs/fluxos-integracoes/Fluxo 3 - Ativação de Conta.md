# Fluxo 3 - Ativação de Conta: Integração Keycloak + Circle

Vincular automaticamente um usuário à comunidade Circle quando ele é adicionado ao grupo **"Assinantes Circle"** no Keycloak, garantindo acesso imediato e sincronização de dados de perfil.

## Contexto e Arquitetura de Segurança

### SSO (Single Sign-On) com Controle de Acesso via Keycloak

Utilizamos **Keycloak como provedor SSO** para autenticação na plataforma Circle. Como o **Circle não implementa controle de acesso nativo** (roles/permissions), implementamos uma camada de segurança diretamente no Keycloak:

- **Executor `Restrict Client Auth`**: Apenas usuários que são membros do grupo **"Assinantes Circle"** conseguem acessar o cliente OIDC do Circle
- **Autorização exclusiva por grupo**: Independente de outros papéis ou permissões
- **Sincronização em tempo real**: Quando adicionado/removido do grupo, o acesso é ativado/desativado imediatamente

## Macro Arquitetura

| Componente | Responsabilidade | Ferramenta |
|----|----|----| 
| **Gatilho** | Monitorar novo membro em grupo | Postgres Trigger (user_group_membership) |
| **Validação** | Confirmar que é o grupo certo | N8N (condicional) |
| **Busca de dados** | Recuperar dados do usuário | Keycloak (Query Postgres) |
| **Integração Circle** | Verificar/criar membro na comunidade | Circle API |
| **Atualização de perfil** | Sincronizar IDs entre sistemas | Keycloak API |

## Fluxo Técnico Detalhado

### 1. Gatilho: Monitoramento de Grupo no Keycloak

**O que dispara:**
- Inserção de novo registro na tabela `user_group_membership` do banco de datos do Keycloak
- Representa: um usuário foi adicionado a um grupo

**Dados capturados:**
```json
{
  "group_id": "f8a91e89-f8dc-459d-9c59-3b1aaf945b28",  // ID do grupo
  "user_id": "17896b50-8e7a-4715-b217-3caf12f090de",   // ID do usuário
  "membership_type": "UNMANAGED"                        // Tipo de associação
}
```

### 2. Validação: É o Grupo Assinantes Circle?

**Condicional:**
```
IF group_id == "f8a91e89-f8dc-459d-9c59-3b1aaf945b28" THEN
  → Continuar com ativação
ELSE
  → Parar (No Operation)
```

**Por quê:** Vários grupos existem no Keycloak (admin, moderators, etc). Essa validação garante que o fluxo de ativação só dispara para o grupo certo.

### 3. Busca de Dados do Usuário

**Query executada:**
```sql
SELECT 
    ue.id AS user_id,
    ue.email AS email,
    ue.first_name AS first_name,
    ue.last_name AS last_name,
    COALESCE(
        jsonb_object_agg(ua.name, ua.value) FILTER (WHERE ua.name IS NOT NULL),
        '{}'::jsonb
    ) AS attributes
FROM user_entity ue
LEFT JOIN user_attribute ua ON ua.user_id = ue.id
INNER JOIN realm r ON r.id = ue.realm_id
WHERE ue.id = '{{ user_id }}'
  AND r.name = 'bonde'
GROUP BY ue.id, ue.email;
```

**Dados retornados:**
| Campo | Tipo | Uso |
|----|----|----| 
| `user_id` | UUID | Identificador único no Keycloak |
| `email` | string | Identificador no Circle |
| `first_name`, `last_name` | string | Dados de perfil |
| `attributes` | JSON | Metadados customizados (ex: `circle_community_id`) |

### 4. Verificar Existência do Atributo `circle_community_id`

**Condicional:**
```
IF attributes.circle_community_id EXISTS THEN
  → Usuário já foi vinculado ao Circle
  → Buscar dados do membro no Circle
ELSE
  → Primeira ativação
  → Criar novo membro no Circle
```

**Importância:** Evita criar duplicatas no Circle.

### 5A. Caminho: Membro Já Existe no Circle

**Passo 1** - Buscar membro na Circle via email:
```
GET https://app.circle.so/api/admin/v2/community_members/search?email={{user.email}}
```

**Passo 2** - Validar se encontrou:
```
IF response.error == null THEN
  → Membro encontrado, continuar
ELSE
  → Erro na busca, parar
```

**Passo 3** - Autenticar no Keycloak:
```
POST https://auth.bonde.org/realms/bonde/protocol/openid-connect/token
  grant_type: "client_credentials"
  client_id: "n8n"
  client_secret: "***"
```

**Passo 4** - Montar payload com novo ID:
```json
{
  "email": "{{ user.email }}",
  "firstName": "{{ user.first_name }}",
  "lastName": "{{ user.last_name }}",
  "attributes": {
    "...": "{{ other_attributes }}",
    "circle_community_member_id": "{{ circle_member.id }}"
  }
}
```

**Passo 5** - Atualizar usuário no Keycloak:
```
PUT https://auth.bonde.org/admin/realms/bonde/users/{{ user.id }}
  Authorization: Bearer {{ token }}
  Body: {{ payload }}
```

### 5B. Caminho: Criar Novo Membro no Circle

**Passo 1** - Criar membro na comunidade Circle:
```
POST https://app.circle.so/api/admin/v2/community_members
{
  "email": "{{ user.email }}",
  "skip_invitation": true  // Não enviar convite por email
}
```

**Resposta esperada:**
```json
{
  "community_member": {
    "id": "abc123...",
    "email": "{{ user.email }}"
  }
}
```

**Passo 2** - Autenticar no Keycloak:
```
POST https://auth.bonde.org/realms/bonde/protocol/openid-connect/token
  (mesmo que caminho anterior)
```

**Passo 3** - Montar payload com novo ID:
```json
{
  "email": "{{ user.email }}",
  "firstName": "{{ user.first_name }}",
  "lastName": "{{ user.last_name }}",
  "attributes": {
    "...": "{{ other_attributes }}",
    "circle_community_member_id": "{{ community_member.id }}"
  }
}
```

**Passo 4** - Atualizar usuário no Keycloak:
```
PUT https://auth.bonde.org/admin/realms/bonde/users/{{ user.id }}
  Authorization: Bearer {{ token }}
  Body: {{ payload }}
```

## Resultado Esperado

### Estado Antes do Fluxo
- ✗ Usuário adicionado ao grupo "Assinantes Circle" no Keycloak
- ✗ Sem vinculação no Circle
- ✗ Sem permissão para fazer login no Circle

### Estado Depois do Fluxo
- ✅ Usuário vinculado como membro da comunidade Circle
- ✅ Atributo `circle_community_member_id` registrado no Keycloak
- ✅ SSO preparado: próximo login no Circle será bem-sucedido
- ✅ Dados de perfil sincronizados em ambos os sistemas

## Possíveis Cenários de Erro

| Erro | Causa | Resolução |
|----|----|----|
| Group ID inválido | Usuário adicionado a outro grupo | O fluxo ignora, nenhuma ação é tomada |
| Usuário não encontrado no Keycloak | Falha na query | Verifique conexão com banco de dados Keycloak |
| Email duplicado no Circle | Mesmo email já existe como membro | Fluxo recupera o ID existente e sincroniza |
| Falha ao atualizar Keycloak | Token expirado ou permissões insuficientes | Verifique credenciais do cliente N8N no Keycloak |

## Conceitos-Chave

### Circle Community Member ID

Identificador único que vincula um usuário no Circle ao seu registro no Keycloak. Armazenado em:
```
Keycloak → user_attribute[circle_community_member_id]
```

**Por quê importante:** Garante sincronização bidirecional (atualizações de perfil Circle → Keycloak funcionam via esse ID).

### Skip Invitation

Flag `skip_invitation: true` ao criar membro no Circle:
- Não envia email de convite ao novo membro
- Usuário terá acesso via SSO do Keycloak (não precisa confirmar por email)

### Autorização SSO vs Autorização Circle

```
Camada 1 (Keycloak):                    Camada 2 (Circle):
├─ Está no grupo? ← Autorização        ├─ Existe membro? ← Validação
└─ SIM → gera JWT                       └─ SIM → sessão de usuário
```

Se removido do grupo Keycloak → JWT revogado → Sem acesso ao Circle (mesmo que membro exista lá).
