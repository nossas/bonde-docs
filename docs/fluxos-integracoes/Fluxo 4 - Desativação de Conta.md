# Fluxo 4 - Desativação de Conta: Integração Keycloak + Circle

Revogar o acesso automaticamente quando um usuário é removido do grupo **"Assinantes Circle"** no Keycloak, desconectando-o da comunidade Circle e encerrando suas sessões ativas.

## Objetivo e Contexto

### Por que Desativar Contas?

Quando um cliente cancela sua assinatura ou tem acesso revogado, precisamos garantir que:

1. **Acesso ao Circle é imediatamente cortado** → Não consegue logar
2. **Sessões ativas são encerradas** → Se estiver logado, é desconectado
3. **Dados de membro são removidos** → Não aparece mais na comunidade

**Gatilho:** Remoção do grupo no Keycloak (que acontece quando `SUBSCRIPTION_CANCELLATION` é processado pelo [Fluxo 2](Fluxo%202%20-%20Gerenciamento%20de%20Acesso.md)).

### Sequência Esperada

```
HOTMART: SUBSCRIPTION_CANCELLATION
    ↓
FLUXO 2: Remove usuário do grupo "Assinantes Circle"
    ↓
DATABASE: Trigger detecta DELETE em user_group_membership
    ↓
FLUXO 4: Desativa conta (desconecta Circle + Keycloak)
```

## Macro Arquitetura

| Componente | Responsabilidade | Ferramenta |
|----|----|----| 
| **Gatilho** | Monitorar remoção de usuário do grupo | Postgres Trigger (DELETE) |
| **Validação** | Confirmar que é o grupo Assinantes Circle | N8N (condicional) |
| **Busca de dados** | Recuperar ID do membro no Circle | Keycloak (Query Postgres) |
| **Remover do Circle** | Deletar membro da comunidade | Circle API (DELETE) |
| **Encerrar sessão** | Logout remoto do usuário | Keycloak API (POST logout) |

## Fluxo Técnico Detalhado

### 1. Gatilho: Monitoramento de Remoção de Grupo

**O que dispara:**
- **DELETE** em `user_group_membership` (um usuário foi removido de um grupo)
- Não INSERT, apenas DELETE

**Dados capturados:**
```json
{
  "group_id": "f8a91e89-f8dc-459d-9c59-3b1aaf945b28",   // Qual grupo?
  "user_id": "abf27447-cb28-4c67-8094-e1c8beb56dbb",    // Qual usuário?
  "membership_type": "UNMANAGED"
}
```

**Importante:** Este trigger é passivo → dispara quando `SUBSCRIPTION_CANCELLATION` remove o usuário (via Fluxo 2).

### 2. Validação: É o Grupo Assinantes Circle?

**Condicional:**
```
IF group_id == "f8a91e89-f8dc-459d-9c59-3b1aaf945b28" THEN
  → Continuar com desativação
ELSE
  → Parar (No Operation)
```

**Motivo:** Vários grupos existem. Apenas Assinantes Circle dispara desativação.

### 3. Busca de Dados do Usuário

**Query executada:**
```sql
SELECT 
    ue.id AS user_id,
    ue.email AS email,
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
| `user_id` | UUID | Identificador no Keycloak |
| `email` | string | Identificador do usuário |
| `attributes.circle_community_member_id` | UUID | ID do membro no Circle |

### 4. Verificar Existência de Circle ID

**Condicional:**
```
IF attributes.circle_community_member_id EXISTS THEN
  → Usuário foi vinculado ao Circle
  → Prosseguir com desativação
ELSE
  → Usuário nunca foi vinculado ao Circle
  → Monitorar (pode indicar problema de sincronização)
```

**Importância:** Nem todo usuário no Keycloak necessariamente foi ativado no Circle (pode ter falhas de rede).

---

### 5. Desconectar do Circle

**Endpoint:**
```
DELETE https://app.circle.so/api/admin/v2/community_members/{{ circle_member_id }}
```

**Headers:**
```
Authorization: Bearer {{ circle_admin_token }}
```

**Parâmetros:**
| Parâmetro | Valor |
|----|----|
| `circle_member_id` | ID do membro no Circle (armazenado em Keycloak) |

**Resposta (sucesso):**
```
Status: 204 No Content
```

**O que acontece:**
- Membro é removido da comunidade
- Sessão do usuário no Circle é encerrada
- Dados do membro (mensagens, posts) permanecem mas sem autor associado

**Importante:** Remover um membro já desconecta a sessão no Circle (segundo suporte da Circle).

### 6. Obter Token de Admin (Keycloak)

**Endpoint:**
```
POST https://auth.bonde.org/realms/bonde/protocol/openid-connect/token
```

**Credenciais:**
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
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "expires_in": 300,
  "token_type": "Bearer"
}
```

---

### 7. Fazer Logout no Keycloak

**Endpoint:**
```
POST https://auth.bonde.org/admin/realms/bonde/users/{{ user_id }}/logout
```

**Headers:**
```
Authorization: Bearer {{ access_token }}
Content-Type: application/json
```

**Resposta:**
```
Status: 204 No Content
```

**O que acontece:**
- Todas as sessões ativas do usuário no Keycloak são encerradas
- Qualquer token JWT emitido para esse usuário é invalidado
- Se usuário tentar acessar Circle, será redirecionado para login
- Se estiver logado em outro aplicativo (Circle), será desconectado

**Importante:** Isso afeta APENAS o Keycloak, não o Circle diretamente. Mas como Circle depende de JWT do Keycloak, o efeito é imediato.

---

## Sequência de Eventos Completa

```
1. User Group Membership DELETE trigger
   ↓
2. Validação: É Assinantes Circle? ✓
   ↓
3. Keycloak Usuário: Query dados
   ↓
4. Existe Circle ID? ✓
   ├─ SIM:
   │  ├─ DELETE /circle/community_members/{{ id }}
   │  │  ↓ Membro removido do Circle
   │  ├─ GET /keycloak/token
   │  │  ↓ Token obtido
   │  └─ POST /keycloak/logout
   │     ↓ Sessão encerrada
   └─ NÃO:
      └─ Monitorar (nenhuma ação no Circle)
```

## Resultado Esperado

### Estado Antes do Fluxo
- ✓ Usuário no grupo "Assinantes Circle"
- ✓ Membro ativo da comunidade Circle
- ✓ Sessão ativa (se logado)

### Estado Depois do Fluxo
- ✗ Usuário REMOVIDO do grupo Keycloak
- ✗ Membro REMOVIDO da comunidade Circle
- ✗ Sessão ENCERRADA (desconectado)
- ✗ Próximo acesso requer novo login (e será negado)

### Cenário: Usuário Tenta Acessar Após Desativação

```
1. Usuário clica em link da Circle
2. Circle redireciona para Keycloak SSO
3. Keycloak verifica: está no grupo "Assinantes Circle"?
4. Keycloak nega: NÃO
5. Keycloak retorna erro "Access Denied"
6. Circle mostra "Você não tem permissão para acessar"
```

---

## Mapeamento de Dados Entre Sistemas

| Sistema | Campo | Valor |
|----|----|----| 
| Postgres Keycloak | `user_entity.id` | `abf27447-cb28-4c67-8094-e1c8beb56dbb` |
| Keycloak Atributos | `circle_community_member_id` | `abc123def456...` (ID do Circle) |
| Circle | `community_members.id` | Mesmo ID do atributo Keycloak |

**Fluxo de sincronização:**
```
Fluxo 3 (Ativação) cria este mapping:
  Keycloak.user.attributes[circle_community_member_id] = Circle.member.id

Fluxo 4 (Desativação) usa este mapping:
  DELETE /circle/members/{{ Keycloak.user.attributes[circle_community_member_id] }}
```

---

## Possíveis Cenários de Erro

| Erro | Causa | Resolução |
|----|----|----|
| Circle ID não existe | Usuário nunca foi ativado no Circle | N8N monitora (TODO node) e registra |
| Falha ao deletar do Circle | Membro já foi removido | Circle retorna 404, é idempotente |
| Falha ao obter token Keycloak | Credenciais expiradas/inválidas | Verificar client_secret no Keycloak |
| Falha ao fazer logout | Usuário não tem sessão ativa | POST é idempotente, sucesso mesmo assim |
| Group ID inválido | Usuário removido de outro grupo | N8N ignora, nenhuma ação |
| Query Postgres falha | Conexão com banco Keycloak | Verificar conectividade |

---

## Conceitos-Chave

### DELETE Trigger vs INSERT Trigger

- **Ativação (Fluxo 3):** Usa **INSERT** trigger (usuário ADICIONADO a grupo)
- **Desativação (Fluxo 4):** Usa **DELETE** trigger (usuário REMOVIDO de grupo)

Assim, ambos os fluxos são automáticos e espelham as ações uma da outra.

### Logout Remoto

O endpoint `POST /users/{{ id }}/logout` é raro e poderoso:
- Invalida TODOS os tokens do usuário
- Encerra TODAS as sessões abertas
- Não requer consentimento do usuário
- Útil para desativação de conta/cancelamento

### Idempotência

Ambas as deletações são "seguras" se executadas múltiplas vezes:
```
DELETE /circle/members/{{ id }}  (já deletado)
  → API retorna 404 (não encontrado)
  → N8N continua normalmente (não é erro fatal)

POST /keycloak/logout  (já feito logout)
  → Keycloak retorna 204 (sucesso)
  → Não há problema em fazer novamente
```

### CircleID como Primary Key de Sincronização

O `circle_community_member_id` armazenado em Keycloak é crítico:
- Sem ele, não conseguimos remover membro do Circle
- Se perder sincronização, usuário fica órfão no Circle
- TODO node alerta se não existir

---

## Monitoramento e Alertas

### O que Monitorar

1. **TODO Node** → Usuários sem Circle ID
   - Indica falha de sincronização do Fluxo 3
   - Requer ação manual para limpar

2. **Erros HTTP 404** → Membro já foi deletado
   - Pode indicar exclusões duplicadas
   - Não é crítico (idempotente)

3. **Erros de Autenticação** → Token Keycloak inválido
   - Verifica saúde das credenciais N8N
   - Crítico se padrão

### Logs Recomendados

```json
{
  "event": "account_deactivation",
  "timestamp": "2026-04-15T10:30:00Z",
  "user_id": "abf27447-...",
  "email": "user@example.com",
  "circle_member_id": "abc123def456...",
  "actions": {
    "circle_delete": "success",
    "keycloak_logout": "success"
  }
}
```

---

## Fluxos Relacionados

- ✅ **Fluxo 2** (Gerenciamento de Acesso): Remove usuário do grupo (dispara este fluxo)
- ✅ **Fluxo 3** (Ativação): Cria o mapping circle_community_member_id (usado para desativar)
- ✅ **Fluxo 1** (Enriquecimento AC): Atualiza status no ActiveCampaign em paralelo

---

## Checklist de Implementação

- [ ] Postgres Trigger configurado para DELETE em user_group_membership
- [ ] Group ID verificado (`f8a91e89-f8dc-459d-9c59-3b1aaf945b28`)
- [ ] Credenciais Circle Admin Token configuradas
- [ ] Credenciais N8N Keycloak configuradas
- [ ] Teste: Remover usuário do grupo → membro deletado do Circle
- [ ] Teste: Usuário que tentava acessar Circle vê "Access Denied"
- [ ] Monitoramento de TODO node configurado
- [ ] Alertas de erro ativados no N8N

---

## Notas Técnicas

### Diferença: DELETE Membro Circle vs Disable Usuário Keycloak

Não desabilitamos o usuário no Keycloak (`enabled: false`), apenas:
- Removemos do grupo Assinantes Circle
- Logout de todas as suas sessões

**Por quê?**
- Usuário pode reativar assinatura → readmitir no grupo é simples
- Se desabilitar conta, recuperação é complexa
- Histórico dos dados do usuário permanece (auditoria)

### Magic Link Não É Enviado

Diferente do Fluxo 2 (criar usuário), aqui não enviamos novo Magic Link porque:
- Usuário está sendo desativado, não ativado
- Se reativar assinatura, vai usar Fluxo 2 novamente
- Enviar link agora causaria confusão

### Erro em Cascata se Circle falhar

```
Sequência:
1. DELETE Circle member
   ├─ SUCESSO → continua para Keycloak logout
   └─ FALHA → N8N para, não faz logout Keycloak
              Usuário fica desconectado do Circle mas
              ainda tem JWT válido (risco!)
```

**Solução:** Configurar retry automático ou alertas críticos se Circle falhar.
