# Fluxo 5 - Sincronização de Perfil: Circle → Keycloak

Sincronizar automaticamente alterações de perfil do usuário feitas na comunidade Circle para o Keycloak, mantendo os dados de identidade consistentes entre os dois sistemas. Implementa debounce para evitar race conditions e merge de atributos para não sobrescrever dados existentes.

## Objetivo e Contexto

### Por que Sincronizar Perfil?

Usuários podem atualizar seus dados em dois lugares:
- **Circle:** Mudar nome, foto, bio, campos customizados
- **Keycloak:** Atributos de identidade, permissões

Sem sincronização → dados divergem:
```
Cenário problemático:
├─ Usuário muda nome no Circle
├─ Keycloak continua com nome antigo
├─ Automações AC usam nome desatualizado
└─ Emails personalizados ficam com nome errado
```

Este fluxo garante que **todas as mudanças no Circle propagam para Keycloak em tempo real**.

### Desafios Técnicos Resolvidos

1. **Race Condition:** Múltiplos webhooks do Circle chegam juntos → debounce aguarda 5 segundos
2. **Atualização Parcial:** Keycloak não aceita PATCH → merge com dados existentes
3. **Divergência de Dados:** Não sobrescrever atributos que Circle não conhece

## Macro Arquitetura

| Componente | Responsabilidade | Ferramenta |
|----|----|----| 
| **Webhook** | Receber eventos de alteração | Circle (POST → N8N) |
| **Normalização** | Extrair dados do evento | N8N (Set) |
| **Debounce** | Evitar processamento simultâneo | Postgres + Wait |
| **Busca Circle** | Obter estado completo do membro | Circle API |
| **Transformação** | Converter formato Circle → Keycloak | N8N (JavaScript) |
| **Busca Keycloak** | Localizar usuário no Keycloak | Keycloak Postgres |
| **Merge** | Combinar dados sem perder informações | N8N (JavaScript) |
| **Atualização** | Sincronizar para Keycloak | Keycloak API |

## Fluxo Técnico Detalhado

### 1. Webhook: Receber Eventos Circle

**URL de entrada:**
```
POST https://n8n.bonde.org/webhook/{{ webhookId }}
```

**Evento típico (profile_field_updated):**
```json
{
  "type": "community_member_profile_field_updated",
  "data": {
    "community_id": 465308,
    "community_member_id": 77250,
    "profile_field_id": 5222975,
    "community_member_profile_field_id": 130882535,
    "profile_field_key": "bio",
    "profile_field_value": "Desenvolvedor e formador"
  }
}
```

**Tipos de eventos suportados:**
- `community_member_profile_field_updated` → Campo de perfil alterado

**Estrutura de dados:**
| Campo | Tipo | Descrição |
|----|----|----| 
| `community_id` | number | ID da comunidade Circle |
| `community_member_id` | number | ID único do membro |
| `profile_field_id` | number | ID do campo no Circle |
| `community_member_profile_field_id` | number | ID da instância do campo para este membro |
| `profile_field_key` | string | Nome técnico do campo (ex: `bio`) |
| `profile_field_value` | string | Novo valor do campo |

### 2. Normalização: Extrair Dados Essenciais

**O que faz:** Padroniza o payload do Circle para formato interno usado pelo debounce

**Dados extraídos:**
```json
{
  "user_id": 77250,           // De: data.community_member_id
  "event_type": "community_member_profile_field_updated"
}
```

**Por quê normalizar?** Porque:
- O webhook traz muitos dados que não precisamos agora
- O debounce só precisa saber "qual usuário foi alterado"
- Reduz tamanho do payload armazenado no controle de debounce
- Facilita identificar eventos duplicados do mesmo usuário

### 3. Debounce: Upsert em Tabela de Controle

**Problema:** Usuário muda nome + foto + bio = 3 webhooks em <1 segundo

**Solução:** Postgres UPSERT com timestamp

**Tabela:**
```sql
CREATE TABLE sync_keycloak_circle_control (
  user_id BIGINT PRIMARY KEY,
  updated_at TIMESTAMP DEFAULT NOW()
);
```

**Query executada:**
```sql
INSERT INTO sync_keycloak_circle_control (user_id, updated_at)
VALUES ($1, NOW())
ON CONFLICT (user_id)
DO UPDATE SET updated_at = NOW();
```

**Resultado:** Sempre há um único registro por usuário com o timestamp do último webhook

### 4. Wait: Aguardar Debounce

**Duração:** 5 segundos

**Propósito:** Permitir que múltiplos webhooks cheguem antes de processar

**Comportamento:**
```
t=0s:   Webhook 1 chega → Atualiza timestamp
t=0.5s: Webhook 2 chega → Atualiza timestamp
t=1s:   Webhook 3 chega → Atualiza timestamp
t=6s:   Wait termina → Processa (pega são timestamp mais recente)
```

### 5. Verificar se É o Evento Mais Recente

**Query:**
```sql
SELECT *
FROM sync_keycloak_circle_control
WHERE user_id = $1
AND updated_at <= NOW() - INTERVAL '5 seconds';
```

**Lógica:**
- Se registro foi atualizado há mais de 5 segundos → É o evento mais recente ✅
- Se registro foi atualizado há menos de 5 segundos → Outro webhook chegou depois ❌

**Resultado:** Apenas 1 de 3 webhooks será processado (o último)

### 6. Condicional: Processar Ou Ignorar?

```
IF evento é o mais recente THEN
  → Continuar com sincronização
ELSE
  → Ignorar (outro webhook virá depois)
```

---

### 7. Buscar Dados Completos do Membro Circle

**Por quê?** O webhook traz apenas UM campo atualizado, mas precisamos do estado completo do perfil.

**Endpoint:**
```
GET https://comunidade.bonde.org/api/admin/v2/community_members/{{ member_id }}
```

**Headers:**
```
Authorization: Bearer {{ circle_admin_token }}
```

**Resposta com todos os campos:**
```json
{
  "id": 77250,
  "email": "usuario@example.com",
  "name": "João Silva",
  "profile_image_url": "https://...",
  "username": "joao-silva",
  "flattened_profile_fields": {
    "bio": "Desenvolvedor e formador",
    "empresa": "BONDE Brasil",
    "phone": "11999999999",
    "website": "https://example.com",
    "linkedin": "joaosilva",
    "github": "joaosilva"
  }
}
```

**Por quê buscar de novo?** Porque:
- Webhook traz apenas `{ profile_field_key: "bio", profile_field_value: "..." }`
- Sem buscar, só sincronizaríamos 1 campo e perderia os outros
- Debounce pode agregar múltiplos webhooks de campos diferentes
- Estado completo garante sincronização correta de todo o perfil

### 8. Transformação: Converter Formato Circle → Keycloak

**Contexto:** Após buscar dados completos do membro no Circle, recebemos `flattened_profile_fields` que contém os campos de perfil do usuário.

**Problema:** Keycloak representa atributos como arrays de strings, Circle como objeto plano

**Circle (retorno da API):**
```json
{
  "flattened_profile_fields": {
    "bio": "Desenvolvedor e formador",
    "empresa": "BONDE",
    "phone": "11999999999"
  }
}
```

**Keycloak formato esperado:**
```json
{
  "attributes": {
    "bio": ["Desenvolvedor e formador"],
    "empresa": ["BONDE"],
    "phone": ["11999999999"],
    "circle_community_member_id": ["77250"]
  }
}
```

**Transformação (JavaScript):**
```javascript
const attributes = {};
const fields = flattened_profile_fields || {};

// Converte cada campo para array de string (formato Keycloak)
for (const key in fields) {
  const value = fields[key];
  if (value !== null && value !== undefined) {
    attributes[key] = [String(value)];
  }
}

// Adiciona sempre o circle_community_member_id (para identificar membro no Circle)
attributes["circle_community_member_id"] = [String(circle_member_id)];

return {
  attributes: attributes,
  email: email
};
```

**Nota importante:** O webhook individual de campo (`community_member_profile_field_updated`) traz apenas 1 campo, mas ao buscar o membro completo no GET, obtemos todos os `flattened_profile_fields` no estado atual. Isso garante sincronização mesmo que múltiplos campos sejam alterados rapidamente.

### 9. Buscar Usuário no Keycloak

**Query Postgres (Keycloak DB):**
```sql
SELECT u.id
FROM user_entity u
JOIN user_attribute a ON u.id = a.user_id
WHERE a.name = 'circle_community_member_id'
AND a.value = $1
LIMIT 1;
```

**Lógica:** Usa `circle_community_member_id` como chave estrangeira para encontrar o usuário

**Resultado:**
```json
{
  "id": "abf27447-cb28-4c67-8094-e1c8beb56dbb"
}
```

### 10. Validação: Usuário Existe?

**Condicional:**
```
IF user_id EXISTS THEN
  → Continuar com atualização
ELSE
  → Parar (usuário não foi criado no Keycloak ainda)
```

**Cenário de erro:** Usuário criado no Circle sem passar pelo Fluxo 2 (assim não tem usuário Keycloak).

---

### 11. Obter Token de Admin Keycloak

**Endpoint:**
```
POST https://auth.bonde.org/realms/bonde/protocol/openid-connect/token
```

**Credenciais (Client Credentials):**
```json
{
  "grant_type": "client_credentials",
  "client_id": "n8n",
  "client_secret": "GZIT95PlJdCe61iV1yivmvMmhqfCtHV9"
}
```

### 12. Buscar Atributos Atuais do Usuário Keycloak

**Endpoint:**
```
GET https://auth.bonde.org/admin/realms/bonde/users/{{ user_id }}
```

**Headers:**
```
Authorization: Bearer {{ access_token }}
```

**Resposta:**
```json
{
  "id": "abf27447-cb28-4c67-8094-e1c8beb56dbb",
  "email": "usuario@example.com",
  "firstName": "João",
  "lastName": "Silva",
  "enabled": true,
  "attributes": {
    "circle_community_member_id": ["12345"],
    "empresa": ["Empresa Antiga"],
    "interno_campo": ["Não sobrescrever isso"]
  }
}
```

**Por quê buscar?** Porque Keycloak não aceita PATCH → precisamos enviar objeto completo. Se não buscarmos antes, perdemos dados existentes.

### 13. Merge de Atributos

**Problema:** Circle envia apenas seus campos. Keycloak pode ter outros campos não gerenciados pelo Circle.

**Solução:** Merge preservador

**JavaScript:**
```javascript
const user = keycloak_user;
const currentAttributes = user.attributes || {};
const incomingAttributes = transform_attributes || {};

// Merge: novos sobrescrevem antigos, mas antigos que não estão em novos são preservados
const mergedAttributes = {
  ...currentAttributes,      // Começa com tudo que já existe
  ...incomingAttributes      // Sobrescreve apenas com novos valores
};

user.attributes = mergedAttributes;

return user;
```

**Exemplo:**

**Antes (Keycloak):**
```json
{
  "circle_community_member_id": ["12345"],
  "empresa": ["BONDE"],
  "departamento": ["Desenvolvimento"],
  "interno_id": ["ABC123"]
}
```

**Chegando (Circle):**
```json
{
  "circle_community_member_id": ["12345"],
  "empresa": ["BONDE Novembro"],
  "bio": ["Dev Senior"]
}
```

**Resultado (Merge):**
```json
{
  "circle_community_member_id": ["12345"],
  "empresa": ["BONDE Novembro"],        // Sobrescrito
  "departamento": ["Desenvolvimento"],   // Preservado
  "interno_id": ["ABC123"],             // Preservado
  "bio": ["Dev Senior"]                 // Novo
}
```

### 14. Atualizar Usuário Keycloak

**Endpoint:**
```
PUT https://auth.bonde.org/admin/realms/bonde/users/{{ user_id }}
```

**Headers:**
```
Authorization: Bearer {{ access_token }}
Content-Type: application/json
```

**Body:** Objeto completo do usuário com attributes mesclados

**Resposta:**
```
Status: 204 No Content
```

---

## Resultado Esperado

### Estado Antes do Fluxo
- ✓ Usuário atualiza campo de perfil no Circle (ex: bio, empresa)
- Circle envia webhook `community_member_profile_field_updated`
- ✗ Keycloak ainda tem valor antigo do campo

### Estado Depois do Fluxo
- ✅ Keycloak tem dados sincronizados com Circle
- ✅ Perfil completo preservado (todos os campos)
- ✅ Nenhum dado foi perdido
- ✅ Pronto para automações AC usarem dados atualizados
- ✅ Independente de quantos webhooks chegaram, processamento foi feito apenas 1 vez

### Exemplo Prático

**Usuário executa:**
1. Clica em perfil no Circle
2. Muda campo "bio" de "Dev Junior" para "Dev Senior"
3. Muda campo "empresa" de "Anterior" para "BONDE Brasil"
4. Clica "Salvar"

**Circle envia 2 webhooks:**
```
t=0s: Webhook 1 → bio mudou
t=0.05s: Webhook 2 → empresa mudou
```

**N8N processa (com debounce):**
```
t=0s: Webhook 1 (bio) → Normaliza → Upsert controle timestamp=0
t=0.05s: Webhook 2 (empresa) → Normaliza → Upsert controle timestamp=0.05

t=6s: Wait termina
     → Verifica: timestamp 0.05 < NOW() - 5s? SIM ✓
     → Processa apenas 1 vez!
     → GET /circle/members/77250
        Retorna: { flattened_profile_fields: { bio: "Dev Senior", empresa: "BONDE Brasil", ... } }
     → Transforma atributos
     → Busca usuário Keycloak
     → Busca atributos atuais
     → Merge
     → PUT para atualizar (1 requisição apenas!)

t=6.5s: ActiveCampaign automações usam dados atualizados
```

**Economia:**
- Sem debounce: 2 buscas Circle + 2 updates Keycloak = 4 requisições
- Com debounce: 1 busca Circle + 1 update Keycloak = 2 requisições (50% de redução)

---

## Debounce: Explicação Detalhada

### Por que Debounce é Necessário?

**Cenário: Usuário atualiza múltiplos campos rapidamente**

**Sem debounce:**
```
t=0s:   Webhook 1 (bio atualizado) chega
        → Normaliza → Busca Circle (estado: bio=antigo)
        → Transforma → Busca Keycloak
        → Update Keycloak
        → Salva timestamp

t=0.1s: Webhook 2 (empresa atualizado) chega
        → Normaliza → Busca Circle (estado: empresa=antigo)
        → Transforma → Busca Keycloak
        → Update Keycloak
        → Salva timestamp (substitui anterior)

t=0.2s: Webhook 3 (phone atualizado) chega
        → Normaliza → Busca Circle (estado: phone=novo)
        → Transforma → Busca Keycloak
        → Update Keycloak
        → Salva timestamp

Resultado: 3 requisições pro Circle, 3 pro Keycloak
           Possível desincronização se um falhar
```

**Com debounce:**
```
t=0s:   Webhook 1 chega → Upsert timestamp = 0s
t=0.1s: Webhook 2 chega → Upsert timestamp = 0.1s
t=0.2s: Webhook 3 chega → Upsert timestamp = 0.2s

t=6s:   Wait termina (5 segundos depois do último webhook)
        → Verifica: timestamp 0.2s < NOW() - 5s? SIM ✓
        → Processa UMA ÚNICA VEZ
        → Busca Circle uma vez (retorna: bio=novo, empresa=novo, phone=novo)
        → Busca Keycloak uma vez
        → Merge uma vez
        → Update Keycloak uma vez

Resultado: 1 requisição pro Circle, 1 pro Keycloak = 50% mais eficiente
           Garante estado completo sincronizado
```

---

## Transformação de Campos

### Estrutura de Webhook vs Estado Completo

O webhook `community_member_profile_field_updated` traz apenas um campo alterado, mas esse não é o estado completo:

**Webhook (campo único):**
```json
{
  "profile_field_key": "bio",
  "profile_field_value": "Desenvolvedor e formador"
}
```

**GET /community_members/:id (estado completo):**
```json
{
  "flattened_profile_fields": {
    "bio": "Desenvolvedor e formador",
    "empresa": "BONDE",
    "phone": "11999999999",
    "website": "https://example.com",
    "linkedin": "joaosilva"
  }
}
```

**Por quê buscar estado completo?**
- Webhook traz apenas 1 campo
- Se ignorarmos os outros, perdemos dados ao fazer merge
- Debounce pode ter agregado múltiplos webhooks em 5 segundos
- Estado completo garante sincronização correta

### Mapeamento de Campos

Campos do Circle em `flattened_profile_fields` são mapeados diretamente para atributos Keycloak:

| Campo Circle | Tipo | Atributo Keycloak | Exemplo |
|----|----|----|----|
| `bio` | string | `attributes.bio[0]` | "Desenvolvedor senior" |
| `empresa` | string | `attributes.empresa[0]` | "BONDE" |
| `phone` | string | `attributes.phone[0]` | "11999999999" |
| `website` | string | `attributes.website[0]` | "https://..." |
| Qualquer outro | string | Mesmo nome em atributos | Genérico |

**Importante:** Nenhum campo é renomeado, transformado ou validado. O valor é copiado como string (envolvido em array).

---

## Possíveis Cenários de Erro

| Erro | Causa | Resolução |
|----|----|----|
| Usuário não existe Keycloak | Não passou por Fluxo 2 | N8N para (sticky note TODO) |
| Webhook malformado | Circle enviou JSON inválido | Webhook rejeita |
| Debounce não funciona | Query SQL falha | Verifica DB Postgres |
| Merge sobrescreve dados | Bug em lógica merge | Verificar JS code node |
| Falha ao atualizar Keycloak | Token expirado | Retry automático |
| Circle member não encontrado | ID inválido | Webhook original foi malformado |

---

## Conceitos-Chave

### Debounce Pattern

Técnica para processar múltiplos eventos rapidamente = como 1 evento:

```
E E E E E  (5 eventos em < 100ms)
  ↓ ↓ ↓ (todos salvam timestamp)
   WAIT (aguarda intervalo)
      ✓ (processa 1 vez com estado mais recente)
```

**Casos de uso:**
- Sincronizações (este fluxo)
- Processamento de bulk
- Debounce de clicks em UI

### Merge Anti-perda

Estratégia para não perder dados ao atualizar:

```
Cenário perigoso:
  Keycloak tem: {A: 1, B: 2, C: 3}
  Circle envia: {A: 10}
  ❌ Sobrescrever direto: {A: 10}  ← Perdemos B e C!

Estratégia segura (merge):
  Keycloak tem: {A: 1, B: 2, C: 3}
  Circle envia: {A: 10}
  ✅ Merge: {A: 10, B: 2, C: 3}  ← Preserva B e C
```

### UPSERT: Update Or Insert

SQL pattern para "atualizar se existe, inserir se não":

```sql
INSERT INTO tabela (id, timestamp)
VALUES ($1, NOW())
ON CONFLICT (id)
DO UPDATE SET timestamp = NOW();
```

Evita:
- Checar se existe (SELECT)
- Branching logic
- Race conditions

### Keycloak não aceita PATCH

Limitação importante:

```
❌ PATCH /users/id
   { "attributes": { "bio": "novo" } }
   → Erro: PATCH não suportado

✅ PUT /users/id
   { "id": "...", "email": "...", "attributes": { "bio": "novo", ... } }
   → Sucesso: PUT substituir tudo
```

Por isso precisa buscar usuário antes de atualizar.

---

## Fluxos Relacionados

- ✅ **Fluxo 2** (Gerenciamento de Acesso): Cria usuário Keycloak que este fluxo sincroniza
- ✅ **Fluxo 3** (Ativação): Cria mapping circle_community_member_id usado aqui
- ✅ **Fluxo 1** (Enriquecimento AC): AC pode usar atributos sincronizados

---

## Performance e Limitações

### Taxa de Sincronização

- **Pico:** Múltiplos webhooks em paralelo → Debounce agrupa
- **Normal:** 1 atualização a cada 6 segundos (5s debounce + 1s processamento)
- **Máximo:** ~10 updates por minuto por usuário

### Limites do Circle API

```
Rate limit: 100 requests/segundo
Timeout: 5 segundos
Retry: Não automático (N8N deve implementar)
```

### Limites do Keycloak API

```
Rate limit: Ilimitado (local)
Timeout: 5 segundos
Tamanho máximo atributo: 255 caracteres
Máximo atributos por usuário: ~500
```

---

## Checklist de Implementação

- [ ] Webhook URL configurada no Circle
- [ ] Tabela `sync_keycloak_circle_control` criada no Postgres
- [ ] Circle Admin Token configurado
- [ ] Keycloak Postgres DB acessível
- [ ] N8N client credentials no Keycloak funcionando
- [ ] Teste: Alterar perfil no Circle → dados sincronizados em Keycloak
- [ ] Teste: Múltiplas alterações rápidas → apenas 1 sincronização
- [ ] Monitoramento de erros ativado
- [ ] Alertas configurados para falhas de sincronização

---

## Extensões Futuras

### Sincronização Reversa (Keycloak → Circle)

Atualmente é apenas **Circle → Keycloak**. Para fazer bidirecional:

```
1. Criar webhook Keycloak para user.updated
2. Seguir padrão similar com debounce
3. Transformar atributos Keycloak → Circle API
4. Enviar PUT /community_members/id ao Circle

⚠️  Cuidado com loops infinitos: A → B → A → B → ...
    Solução: Flag de origem no atributo
```

### Manipulação de Campos Ausentes

Sticky note menciona: "Talvez aqui seja interessante criar um fluxo para atualizar o campo caso o valor não exista"

Implementar opcional:
- Se campo existe em Keycloak mas não no Circle → deletar
- Se campo é NULL → limpar
- Se campo muda tipo → validar antes de atualizar
