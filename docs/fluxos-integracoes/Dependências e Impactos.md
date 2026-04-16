# Dependências e Impactos Entre Fluxos

Análise das dependências e pontos críticos de falha entre os 5 fluxos de integração N8N.

## 📊 Visão Geral das Dependências

### Mapa de Fluxos

```
              HOTMART (Webhooks)
                    │
          ┌─────────┴─────────┐
          │                   │
          ▼                   ▼
     [FLUXO 1]          [FLUXO 2]
  AC Enriquecimento   Gerenciamento
  (Independente)      de Acesso
                      (Keycloak)
                            │
            ┌───────────────┼───────────────┐
            │               │               │
       INSERT               │           DELETE
       Grupo                │           Grupo
            │               │               │
            ▼               │               ▼
       [FLUXO 3]            │         [FLUXO 4]
       Ativação             │         Desativação
       Keycloak             │         Keycloak
         +                  │            +
       Circle               │          Circle
            │               │               │
            └───────────────┼───────────────┘
                    │
              (cria ID mapping)
                    │
                    ▼
               [FLUXO 5]
          Sincronização
          Circle → Keycloak
```

---

## 🔗 Matriz de Dependências

### Legenda
- ❌ = Nenhum impacto
- ✅ = Impacto crítico (dispara ou é pré-requisito obrigatório)
- ⚠️ = Impacto potencial ou race condition
- 📍 = Impacto unidirecional

### Tabela de Impactos

|       | F1 | F2 | F3 | F4 | F5 |
|-------|----|----|----|----|-----|
| **F1** | — | ❌ | ❌ | ❌ | ❌ |
| **F2** | ❌ | — | ✅📍 | ✅📍 | ❌ |
| **F3** | ❌ | ❌ | — | ✅📍 | ✅📍 |
| **F4** | ❌ | ❌ | ❌ | — | ⚠️ |
| **F5** | ❌ | ❌ | ✅🔴 | ❌ | — |

---

## 📋 Fluxos e Seus Gatilhos

| # | Fluxo | Gatilho | Origem | Tipo | Dispara |
|---|-------|---------|--------|------|---------|
| **F1** | Enriquecimento AC | PURCHASE_APPROVED, PURCHASE_CANCELED, SUBSCRIPTION_CANCELLATION, PURCHASE_OUT_OF_SHOPPING_CART | Hotmart Webhook | Independente | Nenhum |
| **F2** | Gerenciamento Keycloak | PURCHASE_APPROVED, SUBSCRIPTION_CANCELLATION | Hotmart Webhook | Independente | F3 (INSERT), F4 (DELETE) |
| **F3** | Ativação Circle | INSERT na tabela `user_group_membership` + validação group_id | Postgres Trigger (via F2) | **Dependente de F2** | Nenhum (lê resultado de F2) |
| **F4** | Desativação Circle | DELETE na tabela `user_group_membership` + validação group_id | Postgres Trigger (via F2) | **Dependente de F2 + F3** | Nenhum |
| **F5** | Sincronização Perfil | `community_member_profile_field_updated` | Circle Webhook | **Dependente de F3** | Nenhum |

---

## 🎯 Análise Detalhada de Impactos

### 1️⃣ F1 → Outros Fluxos

**Impacto: ❌ NENHUM**

F1 (Enriquecimento AC) é **completamente independente**:
- ✅ Apenas escreve em ActiveCampaign
- ✅ Não afeta Keycloak, Circle, ou Keycloak
- ✅ Não dispara triggers em outros fluxos
- ✅ Pode falhar sem impactar acesso do usuário

**Falhas em F1:**
- Cliente com dados incompletos em AC
- Automações AC funcionam com dados parciais
- Segue para F2 normalmente (independente)

---

### 2️⃣ F2 → F3 (IMPACTO CRÍTICO ✅)

**Evento:** `PURCHASE_APPROVED` + `recurrence_number == 1`

#### Sequência de Execução

```
1. Hotmart: Envia event PURCHASE_APPROVED
   
2. F2 (Gerenciamento Keycloak):
   ├─ Normaliza dados (email)
   ├─ Busca usuário no Keycloak
   ├─ Usuário não existe → Cria novo
   ├─ Adiciona usuário ao grupo "Assinantes Circle"
   └─ **INSERT em user_group_membership** ← DISPARA F3
   
3. Database Trigger:
   └─ Detecta INSERT na tabela user_group_membership
   
4. F3 (Ativação Circle):
   ├─ Recebe trigger
   ├─ Valida group_id == "f8a91e89-f8dc-459d-9c59-3b1aaf945b28" ✓
   ├─ Busca dados do usuário no Keycloak
   ├─ Existe circle_id? NÃO → Cria novo membro
   ├─ POST /circle/api/admin/v2/community_members
   └─ Salva "circle_community_member_id" em Keycloak
      ↑ MAPPING CRÍTICO CRIADO
```

#### Dependências

**F3 obrigatoriamente precisa:**
1. ✅ Que F2 tenha criado o usuário no Keycloak
2. ✅ Que F2 tenha adicionado ao grupo "Assinantes Circle"
3. ✅ Que F2 tenha disparado o trigger INSERT

**O que F3 cria:**
- 🔴 Mapping `circle_community_member_id` que F4 e F5 dependem

#### Cenários de Falha

| Cenário | Causa | Consequência |
|---------|-------|--------------|
| **F2 falha ao criar usuário** | Erro Keycloak API | F3 não é disparado → Sem acesso Circle |
| **F2 falha ao adicionar grupo** | Erro Keycloak API | Trigger não dispara → F3 não executa |
| **F3 falha ao criar Circle** | Erro Circle API | Usuário em Keycloak mas não em Circle |
| **F3 falha ao salvar circle_community_member_id** | Erro Keycloak API | **Mapping perdido → F4 e F5 quebram** 🔴 |
| **F3 recebe group_id errado** | Validação falha | Fluxo é ignorado (No Operation) |

#### Mitigação

```javascript
✅ Em F3, OBRIGATÓRIO validar:
   - circle_community_member_id foi salvo? (comparar antes/depois)
   - TODO: alertar se atributo não foi persistido
   
✅ Em F2, OBRIGATÓRIO em caso de erro:
   - Retry 5x com backoff
   - Alerta crítico se fails > 3
```

---

### 3️⃣ F2 → F4 (IMPACTO CRÍTICO ✅)

**Evento:** `SUBSCRIPTION_CANCELLATION`

#### Sequência de Execução

```
1. Hotmart: Envia event SUBSCRIPTION_CANCELLATION
   
2. F2 (Gerenciamento Keycloak):
   ├─ Normaliza dados (email)
   ├─ Busca usuário no Keycloak
   ├─ Usuário existe → Remove do grupo
   └─ **DELETE de user_group_membership** ← DISPARA F4
   
3. Database Trigger:
   └─ Detecta DELETE na tabela user_group_membership
   
4. F4 (Desativação Circle):
   ├─ Recebe trigger
   ├─ Valida group_id == "f8a91e89-f8dc-459d-9c59-3b1aaf945b28" ✓
   ├─ Busca dados do usuário no Keycloak
   ├─ Busca circle_community_member_id? ← PRECISA QUE F3 TENHA CRIADO
   ├─ DELETE /circle/api/admin/v2/community_members/{id}
   ├─ GET /keycloak/admin/token
   └─ POST /keycloak/.../logout
      ↑ ENCERRA SESSÃO
```

#### Dependências

**F4 obrigatoriamente precisa:**
1. ✅ Que F2 tenha removido do grupo (disparado trigger)
2. ✅ Que F3 tenha criado o `circle_community_member_id` (NO PASSADO)
3. ✅ Que circle_community_member_id ainda exista em Keycloak

#### Cenários de Falha

| Cenário | Causa | Consequência |
|---------|-------|--------------|
| **F3 nunca foi executado** | Usuário nunca acessou Circle | F4 não encontra circle_community_member_id → TODO node |
| **circle_community_member_id foi deletado** | Bug ou limpeza manual | F4 falha, usuário não é removido do Circle 🔴 |
| **F4 falha ao remover do Circle** | Erro Circle API | Usuário mantém acesso ao Circle 🔴 |
| **F4 falha ao fazer logout** | Erro Keycloak API | JWT ainda válido, sessão continua ativa |

#### Mitigação

```javascript
✅ Em F4, OBRIGATÓRIO validar:
   - circle_community_member_id existe?
   - TODO: monitorar usuários sem Circle ID
   - Retry 3x em caso de erro
   - Alerta CRÍTICA se logout falhar
```

---

### 4️⃣ F3 → F5 (IMPACTO CRÍTICO 🔴)

**Evento:** Qualquer alteração de perfil no Circle

#### Sequência de Execução

```
1. Circle User: Muda campo "bio", "empresa", etc

2. Circle: Dispara webhook community_member_profile_field_updated
   
3. F5 (Sincronização Perfil):
   ├─ Recebe webhook
   ├─ Normaliza dados (community_member_id)
   ├─ Debounce: aguarda 5 segundos (agrupa múltiplos webhooks)
   ├─ Verifica se é última alteração
   ├─ Busca dados completos em Circle
   ├─ Transforma atributos
   ├─ **Busca circle_community_member_id em Keycloak** ← PRECISA DE F3
   ├─ Query: WHERE a.name = 'circle_community_member_id'
   ├─ Encontra usuário Keycloak?
   │  ├─ SIM → Continua
   │  └─ NÃO 🔴 → Para (No Operation)
   ├─ Busca atributos atuais
   ├─ Merge (preserve dados, não sobrescreve)
   └─ PUT /keycloak/users/{id}
      ↑ SINCRONIZA
```

#### Dependências

**F5 obrigatoriamente precisa:**
1. ✅ Que F3 tenha criado o `circle_community_member_id`
2. ✅ Que circle_community_member_id ainda exista em Keycloak
3. ✅ Que usuário na Circle corresponda ao ID salvo

#### Cenários de Falha

| Cenário | Causa | Consequência |
|---------|-------|--------------|
| **Usuário criado no Circle manualmente** | Fora do fluxo F3 | F5 não encontra circle_community_member_id → Para |
| **circle_community_member_id foi deletado** | Bug ou limpeza manual | F5 não encontra usuário → Para |
| **F3 falhou no passado** | Erro não tratado | F5 nunca consegue sincronizar → Perfil diverge |
| **Webhook chega durante F4** | Race condition | Circle enviando dados de membro sendo deletado |

#### Descrição do Sticky Note TODO

```
arquivo: Fluxo 5 - Sincronização de Perfil.md
nota: "TODO: Monitorar usuários que não possuem Circle ID"
↑ Esta nota identifica exatamente este problema
```

#### Mitigação

```javascript
✅ Em F5, OBRIGATÓRIO validar:
   - circle_community_member_id existe primeiro?
   - Se não encontrar, log de "órfão"
   - Query periódica: SELECT COUNT(*) FROM user_attribute 
                     WHERE name = 'circle_community_member_id'
   - Dashboard: "Usuários órfãos = 0"
   - Retry 2x (menos crítico que F3/F4)
```

---

### 5️⃣ F4 → F5 (IMPACTO LATERAL ⚠️)

**Evento:** Cancelamento → F4 deleta do Circle → Circle webhook

#### Sequência Potencial

```
1. F4 inicia exclusão do Circle
   
2. Circle API: DELETE /community_members/{id}
   ✓ Membro removido
   
3. Circle (possível): Dispara webhook de "member_deleted"
   (ou webhooks em fila antes do delete)
   
4. F5 recebe webhook:
   ├─ user_id = 77250
   ├─ Debounce: aguarda 5 segundos
   ├─ Processa após 5 segundos
   ├─ Busca membro em Circle
   └─ Pode estar deletado já
      → Possível erro 404
      → Ou dados inconsistentes
```

#### Risco

- **Timing:** Se F5 recebe webhook ANTES de F4 terminar
- **Debounce:** Mitiga pois aguarda 5 segundos (F4 leva <1s)
- **Cenário raro:** Múltiplos webhooks em fila antes do delete

#### Mitigação

```javascript
✅ Em F5, OBRIGATÓRIO validar:
   - Usuário ainda existe em Circle?
   - Se erro 404, TODO node (esperado se membro foi deletado)
   - Debounce de 5s já mitiga timing issues
```

---

## 📈 Fluxo de Vida Completo de um Usuário

### ✅ Happy Path (Tudo OK)

```
────────────────────────────────────────────────────────────────
t=0:      [HOTMART] PURCHASE_APPROVED (recurrence_number: 1)
          │
          ├─→ F1: Enriquece AC ✅
          │   └─ Contato criado/atualizado
          │
          └─→ F2: Cria usuário Keycloak + grupo ✅
              ├─ Usuário criado
              ├─ Adicionado ao grupo
              └─ INSERT trigger → dispara F3
              
t=0.5:    [DATABASE TRIGGER] INSERT detectado
          │
          └─→ F3: Ativa conta ✅
              ├─ Valida group_id ✓
              ├─ Cria membro Circle
              ├─ Salva circle_community_member_id ← MAPPING CRIADO
              └─ Status: "Usuário ativo em todos sistemas"

────────────────────────────────────────────────────────────────
t=30s:    [CIRCLE] Usuário muda bio
          │
          └─→ F5: Sincroniza ✅
              ├─ Normaliza
              ├─ Debounce (5s)
              ├─ Busca circle_community_member_id (criado por F3) ✓
              ├─ GET Circle (estado completo)
              ├─ Merge com Keycloak
              ├─ PUT Keycloak
              └─ Status: "Perfil sincronizado"

────────────────────────────────────────────────────────────────
t=60d:    [HOTMART] SUBSCRIPTION_CANCELLATION
          │
          ├─→ F1: Atualiza AC status = EXPIRED ✅
          │
          └─→ F2: Remove do grupo ✅
              ├─ Busca usuário
              ├─ Remove do grupo
              └─ DELETE trigger → dispara F4

t=60.5d:  [DATABASE TRIGGER] DELETE detectado
          │
          └─→ F4: Desativa conta ✅
              ├─ Valida group_id ✓
              ├─ Busca circle_community_member_id (criado por F3) ✓
              ├─ DELETE Circle member
              ├─ POST Keycloak logout
              └─ Status: "Usuário desativado"
```

### ❌ Error Path 1: F2 Falha ao Criar Usuário

```
t=0:      [HOTMART] PURCHASE_APPROVED
          │
          ├─→ F1: Enriquece AC ✅
          │
          └─→ F2: FALHA ao criar usuário ❌
              └─ Erro: Keycloak API indisponível
              
               ❌ F3 NUNCA É DISPARADO
               ❌ Usuário sem acesso Circle
               ❌ Membro não criado
               ❌ circle_community_member_id não existe
              
               → Retry automático em F2?
               → Manual review necessário?
               ← CRÍTICO: impacts F3 + F4 + F5
```

### ❌ Error Path 2: F3 Falha ao Salvar ID

```
t=0.5:    [DATABASE TRIGGER] INSERT detectado
          │
          └─→ F3: Ativa conta (PARCIAL) ⚠️
              ├─ Valida group_id ✓
              ├─ Cria membro Circle ✓
              └─ FALHA ao salvar circle_community_member_id ❌
                 └─ Erro: Keycloak API timeout
                 
                 ⚠️ Membro CRIADO mas SEM MAPPING
                 ❌ F4 não conseguirá remover (sem ID)
                 ❌ F5 não conseguirá sincronizar (sem ID)
                 🔴 CRÍTICO: Usuário órfão (SECURITY ISSUE)
                 
                 → Sticky note TODO acumula estos usuários
                 → Manual linking necessário
                 ← Validação em F3 deve detectar isto!
```

### ❌ Error Path 3: Usuário Criado Manualmente no Circle

```
t=0:      [ADMIN] Cria membro manualmente no Circle
          │
          └─ Usuário adicionado sem passar por F3
             └─ circle_community_member_id NÃO existe em Keycloak
             
t=24h:    [CIRCLE] Usuário muda bio
          │
          └─→ F5: FALHA ❌
              ├─ Recebe webhook
              ├─ Debounce (5s)
              └─ Busca circle_community_member_id
                 └─ NÃO ENCONTRA ❌
                    └─ Para (No Operation)
                    
                    ⚠️ Perfil no Circle ≠ Perfil no Keycloak
                    ⚠️ Automações AC usam dados desatualizados
                    
                    → Sticky note TODO registra órfão
                    → Admin precisa linkar manualmente
                    ← Política: TODOS usuários Circle via F3!
```

---

## 🔴 Pontos Críticos de Falha

### **Crítica 1: F3 Falha ao Salvar circle_community_member_id**

```
SEVERIDADE: 🔴 CRÍTICA
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

PROBLEMA:
├─ F3 cria membro Circle ✅
└─ F3 FALHA ao salvar ID em Keycloak ❌
   → Membro criado mas SEM mapping
   
CONSEQUÊNCIAS:
├─ F4 não consegue remover (sem ID) 🔴
├─ F5 não consegue sincronizar (sem ID) 🔴
├─ Usuário órfão (member em Circle desvinculado)
└─ SECURITY: Usuário pode ter acesso sem ser removido

IMPACTO EM:
├─ F4: ❌ Quebra (não encontra circle_community_member_id)
└─ F5: ❌ Quebra (não encontra círculo_community_member_id)

DETECÇÃO:
→ Antes/depois: salvar circle_community_member_id = true?
→ Query: SELECT COUNT(*) WHERE name = 'circle_community_member_id'
→ Alert CRÍTICA se F3 salva falha

MITIGAÇÃO IMEDIATA:
→ Retry 3x em F3
→ Se ainda falha: alertar, parar fluxo (não criar membro orphan)
→ Dead letter queue: Manual review
```

---

### **Crítica 2: F4 Não Consegue Remover (Sem circle_community_member_id)**

```
SEVERIDADE: 🔴 CRÍTICA (SEGURANÇA)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

PROBLEMA:
├─ Usuário cancela assinatura
├─ F2 remove do grupo Keycloak
├─ F4 dispara mas não encontra circle_community_member_id
└─ Não consegue remover do Circle

CAUSA RAIZ:
├─ F3 falhou no passado (Crítica 1)
└─ circle_community_member_id nunca foi salvo

CONSEQUÊNCIAS:
├─ Membro MANTÉM acesso ao Circle 🔴
├─ Pode continuar vendo conteúdo
├─ Pode ser logado em sessão antiga
└─ VIOLAÇÃO DE SEGURANÇA

IMPACTO:
└─ F4: ❌ Para (No Operation), registra TODO

DETECÇÃO:
→ Query: SELECT COUNT(*) FROM n8n_logs WHERE F4_status = TODO
→ Alert se > 0 (usuários não removidos)

MITIGAÇÃO IMEDIATA:
→ Manual review: quem tem TODO pendente?
→ Link circle_community_member_id para estes usuários
→ Re-trigger F4
```

---

### **Crítica 3: Usuários Órfãos (Sem Mapping)**

```
SEVERIDADE: 🟡 MODERADA
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

PROBLEMA:
├─ Usuário criado no Circle fora do fluxo (via API/Admin)
├─ Nunca passou por F3
└─ circle_community_member_id não existe

CENÁRIOS:
├─ Admin cria membro manualmente
├─ Integração externa cria no Circle
└─ Bug em F3 que interrompe salvamento

CONSEQUÊNCIAS:
├─ F5 não consegue sincronizar perfil
├─ Perfil Circle ≠ Perfil Keycloak
└─ Automações AC usam dados desatualizados

IMPACTO:
├─ F5: ❌ Para (No Operation)
└─ Sticky note: "TODO: Monitorar usuários sem Circle ID"

DETECÇÃO:
→ Query periódica:
  SELECT u.id, u.email
  FROM user_entity u
  WHERE NOT EXISTS (
    SELECT 1 FROM user_attribute ua
    WHERE ua.user_id = u.id
    AND ua.name = 'circle_community_member_id'
  )
  AND u.id IN (SELECT id FROM circle_members)

MITIGAÇÃO:
→ Dashboard: "Usuários órfãos = N"
→ Alert se N > 0
→ Manual linking (batch script)
→ Política: NUNCA criar usuários Circle fora de F3
```

---

## ⚡ Recommendations de Implementação

### **Retry Policies**

```javascript
F1 (AC Enriquecimento):
  retry: 3x
  backoff: exponential
  reason: independente, menos crítico

F2 (Keycloak Gerenciamento):
  retry: 5x ← AGRESSIVO! Dispara F3 + F4
  backoff: exponential
  timeout: 30s
  alert: crítica se fails > 3

F3 (Ativação Circle):
  retry: 3x
  backoff: exponential
  validation: circle_community_member_id foi salvo?
  alert: CRÍTICA se falha (quebra F4 + F5)

F4 (Desativação Circle):
  retry: 3x
  backoff: exponential
  alert: CRÍTICA se falha (security issue)

F5 (Sincronização Perfil):
  retry: 2x ← Menos crítico
  backoff: exponential
  timeout: 10s
```

---

### **Monitoramento e Alertas**

```
🔴 CRÍTICA:
├─ F2 failures > 3
├─ F3 circle_community_member_id não salvo
├─ F4 failures (usuários não removidos)
└─ F4 TODO acumulando

🟡 MODERADA:
├─ F1 failures
├─ F5 não encontra circle_community_member_id
├─ Usuários órfãos > 0
└─ Debounce não funcionando (múltiplos processamentos)

ℹ️ INFO:
├─ Taxa de sincronização F5
├─ Quantidade de webhooks agregados por debounce
├─ Tempo médio de processamento por fluxo
└─ Usuários processados com sucesso
```

---

### **Dead Letter Queues**

```
F2 FAILURES:
├─ Requeue para manual review
├─ Log: why it failed (API error, validation, timeout, etc)
└─ Action: retry manualmente ou investigar

F3 FAILURES:
├─ CRÍTICA: log para manual investigation
├─ Criar circle_community_member_id não foi persistido?
└─ Action: verificar estado final em Keycloak

F4 FAILURES + circle_community_member_id NULL:
├─ SEGURANÇA: alert imediato
├─ Usuário tem acesso ao Circle mas não ao grupo Keycloak
└─ Action: remoção manual do Circle

F5 circle_community_member_id NOT FOUND:
├─ Log como órfão
├─ Query: foi antes de F3 executar? ou usuário criado manualmente?
└─ Action: manual linking + retry
```

---

## 📊 Matriz de Impactos Consolidada

```
┌─────────┬─────────────┬────────────────┬──────────────────┐
│ Fluxo   │ Gatilho     │ Dispara        │ Depende de       │
├─────────┼─────────────┼────────────────┼──────────────────┤
│ F1      │ Hotmart ✅  │ Nada ❌        │ Nada             │
├─────────┼─────────────┼────────────────┼──────────────────┤
│ F2      │ Hotmart ✅  │ F3 ✅ (INSERT) │ Nada             │
│         │             │ F4 ✅ (DELETE) │                  │
├─────────┼─────────────┼────────────────┼──────────────────┤
│ F3      │ DB Trigger  │ Nada ❌        │ F2 (entrada) ✅  │
│         │ (INSERT)    │                │ circle_id mapping│
├─────────┼─────────────┼────────────────┼──────────────────┤
│ F4      │ DB Trigger  │ Nada ❌        │ F2 (entrada) ✅  │
│         │ (DELETE)    │                │ F3 (circle_id)✅ │
├─────────┼─────────────┼────────────────┼──────────────────┤
│ F5      │ Circle WH   │ Nada ❌        │ F3 (circle_id) ✅│
└─────────┴─────────────┴────────────────┴──────────────────┘
```

---

## 🎬 Sequências de Eventos Esperadas

### Compra Nova (PURCHASE_APPROVED, recurrence=1)

```
HOTMART EVENT
  ↓
F1 (AC): Cria/atualiza contato ✅
F2 (Keycloak):
  ├─ Cria usuário
  ├─ Adiciona ao grupo "Assinantes Circle"
  └─ INSERT em user_group_membership
     ↓
F3 (Circle + Keycloak):
  ├─ Cria membro Circle
  └─ Salva circle_community_member_id

RESULTADO: Usuário ativo em todos os sistemas ✅
```

---

### Renovação (PURCHASE_APPROVED, recurrence>1)

```
HOTMART EVENT
  ↓
F1 (AC): Atualiza dados + datas ✅
F2 (Keycloak):
  ├─ Busca usuário (já existe)
  ├─ Concede acesso ao grupo (já está lá)
  └─ Nenhum trigger (grupo não muda)

F3, F4, F5: NÃO SÃO DISPARADOS

RESULTADO: Dados atualizados, nenhum impacto em Circle ✅
```

---

### Cancelamento (SUBSCRIPTION_CANCELLATION)

```
HOTMART EVENT
  ↓
F1 (AC): Atualiza status = EXPIRED ✅
F2 (Keycloak):
  ├─ Remove do grupo "Assinantes Circle"
  └─ DELETE em user_group_membership
     ↓
F4 (Circle + Keycloak):
  ├─ Busca circle_community_member_id (criado por F3)
  ├─ Remove membro do Circle
  └─ Logout Keycloak

RESULTADO: Usuário desativado em todos os sistemas ✅
```

---

### Atualização de Perfil (community_member_profile_field_updated)

```
CIRCLE EVENT (bio, empresa, etc)
  ↓
F5 (Keycloak):
  ├─ Normaliza
  ├─ Debounce (5s)
  ├─ Busca circle_community_member_id (criado por F3)
  ├─ GET Circle (estado completo)
  ├─ Merge
  └─ PUT Keycloak

F1, F2, F3, F4: NÃO SÃO DISPARADOS

RESULTADO: Atributos sincronizados, nenhum webhook duplicado ✅
```

---

## 📝 Checklist de Operação

- [ ] **Monitoramento F2:** Alertas para failures > 3
- [ ] **Validação F3:** circle_community_member_id foi salvo?
- [ ] **Alertas F4:** TODO acumulando? Segurança!
- [ ] **Dashboard F5:** Usuários órfãos = 0?
- [ ] **Query periódica:** Detectar orphans manualmente
- [ ] **Dead letter queue:** Revisions configuradas?
- [ ] **Retry policies:** Configuradas conforme recomendação?
- [ ] **Logs estruturados:** Rastreabilidade end-to-end?
- [ ] **Documentação:** Runbooks para falhas críticas?
