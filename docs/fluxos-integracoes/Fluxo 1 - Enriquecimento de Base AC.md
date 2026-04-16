# Fluxo 1 - Checkout Hotmart → ActiveCampaign: Enriquecimento da Base de Contatos

Capturar eventos de checkout da Hotmart (compras aprovadas, renovações, cancelamentos) e sincronizar os dados do comprador para o ActiveCampaign, enriquecendo o perfil do contato com informações de compra, endereço, telefone e histórico de transações.

## Objetivo e Contexto

### Por que Enriquecer a Base?

O **ActiveCampaign** é nosso **CRM central** onde toda a inteligência de marketing e segmentação ocorre. Cada evento de compra na Hotmart deve:

1. Encontrar ou criar o contato no AC baseado no email
2. Atualizar campos de perfil (nome, telefone, endereço)
3. Registrar dados de compra (status, preço, datas)
4. Permitir automações baseadas em histórico (ex: enviar email "bem-vindo aluno")

### Eventos Suportados

| Evento Hotmart | Significado | Campo AC Atualizado |
|----|----|----| 
| `PURCHASE_APPROVED` + `recurrence_number == 1` | **Nova Assinatura** | Todos os campos (compra + perfil) |
| `PURCHASE_APPROVED` + `recurrence_number > 1` | **Renovação** | Status + data próxima renovação |
| `PURCHASE_CANCELED` | **Cancelamento** | Status + limpa datas |
| `SUBSCRIPTION_CANCELLATION` | **Encerramento** | Status = EXPIRED |
| `PURCHASE_OUT_OF_SHOPPING_CART` | **Abandono de Carrinho** | Status = OUT_OF_SHOPPING_CART |

## Macro Arquitetura

| Componente | Responsabilidade | Ferramenta |
|----|----|----| 
| **Webhook** | Receber eventos de checkout | Hotmart (POST → N8N) |
| **Busca de contato** | Encontrar contato pelo email | ActiveCampaign API |
| **Classificação** | Diferenciar tipo de evento | N8N (Switch Dinâmico) |
| **Atualização** | Sincronizar dados | ActiveCampaign API |

## Fluxo Técnico Detalhado

### 1. Webhook: Receber Eventos Hotmart

**URL de entrada:**
```
POST https://n8n.bonde.org/webhook/checkout/crm
```

**Tipos de eventos enviados por Hotmart:**

```json
{
  "body": {
    "event": "PURCHASE_APPROVED",
    "data": {
      "buyer": {
        "email": "cliente@example.com",
        "first_name": "João",
        "last_name": "Silva",
        "checkout_phone_code": "55",
        "checkout_phone": "11999999999",
        "address": {
          "address": "Rua Exemplo",
          "number": "123",
          "complement": "Apto 45",
          "city": "São Paulo",
          "state": "SP",
          "country": "BR",
          "zipcode": "01234567"
        }
      },
      "subscriber": {
        "code": "SUB-12345",
        "email": "cliente@example.com"
      },
      "subscription": {
        "status": "ACTIVE"
      },
      "purchase": {
        "recurrence_number": 1,
        "full_price": {
          "value": 99.90
        },
        "approved_date": 1712345678000,
        "payment": {
          "type": "CREDIT_CARD",
          "installments_number": 1
        },
        "origin": {
          "src": "email",
          "sck": "email",
          "xcod": "live-001"
        },
        "status": "APPROVED"
      }
    }
  }
}
```

### 2. Buscar Contato no ActiveCampaign

**Lógica:**
- Extrair email do webhook: `buyer.email` OU `subscriber.email`
- Buscar contato no AC por esse email
- Se encontrar: usar para atualização
- Se não encontrar: AC cria automaticamente (ao enviar dados)

**Query AC:**
```
GET /contacts?email={{email}}&limit=1
```

**Resposta esperada:**
```json
{
  "id": "123456",
  "email": "cliente@example.com",
  "firstName": "João",
  "lastName": "Silva",
  "phone": "+5511999999999",
  "fields": [...]
}
```

### 3. Classificação: Qual Tipo de Evento?

**Switch Dinâmico com 5 saídas:**

```
IF event === 'PURCHASE_APPROVED' AND recurrence_number === 1
  → Output 0: Nova Assinatura
ELSE IF event === 'PURCHASE_APPROVED' AND recurrence_number > 1
  → Output 1: Renovação
ELSE IF event === 'PURCHASE_CANCELED'
  → Output 2: Cancelamento
ELSE IF event === 'SUBSCRIPTION_CANCELLATION'
  → Output 3: Encerramento
ELSE IF event === 'PURCHASE_OUT_OF_SHOPPING_CART'
  → Output 4: Abandono Carrinho
```

**Importância:** Cada tipo de evento dispara uma atualização diferente com campos específicos.

---

### 4A. Fluxo: Nova Assinatura

**Quando disparado:** Primeira compra de um cliente (recurrence_number == 1)

**Campos atualizados no ActiveCampaign:**

| Campo AC | ID do Campo | Dado Hotmart | Descrição |
|----|----|----|----|
| Código Assinatura | 26 | `subscription.subscriber.code` | ID único da assinatura |
| Status | 27 | `subscription.status` | ex: ACTIVE, CANCELED |
| Preço Cheio | 28 | `purchase.full_price.value` | Valor original do produto |
| Data Aprovação | 29 | `purchase.approved_date` | Quando a compra foi confirmada |
| Próxima Renovação | 30 | `approved_date + installments_number` | Data do próximo pagamento |
| Tipo de Pagamento | 31 | `purchase.payment.type` | ex: CREDIT_CARD, BOLETO |
| Origem da Venda | 33 | `purchase.origin.src` | ex: email, chat-youtube, whatsapp-group |
| Campanha | 35 | `purchase.origin.xcod` | ex: live-001, workshop-february |
| Próxima Renovação (2) | 38 | `approved_date + 1 mês` | Próxima data de renovação |
| CEP | 40 | `buyer.address.zipcode` | Código postal |
| País | 41 | `buyer.address.country` | ex: BR |
| Número | 42 | `buyer.address.number` | Número do endereço |
| Endereço | 43 | `buyer.address.address` | Rua/avenida |
| Cidade | 44 | `buyer.address.city` | ex: São Paulo |
| Estado | 45 | `buyer.address.state` | ex: SP |
| Complemento | 47 | `buyer.address.complement` | ex: Apto 45 |

**Dados de perfil também atualizados:**
```json
{
  "firstName": "{{ buyer.first_name }}",
  "lastName": "{{ buyer.last_name }}",
  "phone": "+{{ buyer.checkout_phone_code }}{{ buyer.checkout_phone }}"
}
```

**Resultado:** Contato completo e pronto para segmentação de marketing.

---

### 4B. Fluxo: Renovação

**Quando disparado:** Pagamento recorrente (recurrence_number > 1)

**Campos atualizados no ActiveCampaign:**

| Campo AC | ID | Dado Hotmart | Descrição |
|----|----|----|----|
| Status | 27 | `subscription.status` | Confirmar que continua ACTIVE |
| Próxima Renovação | 38 | `approved_date + 1 mês` | Atualizar data da próxima renovação |

**Simplificado porque:** Os dados de perfil e de compra inicial já estão salvos. Apenas confirmamos renovação.

---

### 4C. Fluxo: Cancelamento

**Quando disparado:** Cliente clica em "cancelar compra" (PURCHASE_CANCELED)

**Campos atualizados no ActiveCampaign:**

| Campo AC | ID | Valor | Descrição |
|----|----|----|----|
| Status | 27 | `purchase.status` | ex: CANCELED |
| Próxima Renovação | 38 | (limpa) | Remove data futura |
| Data Cancelamento | 30 | `approved_date + 1 mês` | Registra quando expira |
| Data Criação Cancelamento | 39 | `creation_date` | Quando o cancelamento foi criado |

---

### 4D. Fluxo: Encerramento de Assinatura

**Quando disparado:** Assinatura expirou naturalmente (SUBSCRIPTION_CANCELLATION)

**Campos atualizados no ActiveCampaign:**

| Campo AC | ID | Valor | Descrição |
|----|----|----|----|
| Status | 27 | `EXPIRED` | Marca como expirada |
| Próxima Renovação | 38 | (limpa) | Remove data futura |

---

### 4E. Fluxo: Abandono de Carrinho

**Quando disparado:** Cliente deixou produto no carrinho sem completar (PURCHASE_OUT_OF_SHOPPING_CART)

**Campos atualizados no ActiveCampaign:**

| Campo AC | ID | Valor | Descrição |
|----|----|----|----|
| Status | 27 | `OUT_OF_SHOPPING_CART` | Marca como abandono |

---

## Mapeamento de Campos AC

A Hotmart não envia nome/ID dos campos, apenas valores. Usamos IDs numéricos do AC:

```
26:  Código Assinatura
27:  Status Assinatura
28:  Preço Cheio
29:  Data Aprovação
30:  Data Próxima Renovação
31:  Tipo de Pagamento
33:  Origem da Venda
35:  Código da Campanha
38:  Próxima Renovação (redundante com 30)
39:  Data Criação Cancelamento
40:  CEP
41:  País
42:  Número
43:  Endereço
44:  Cidade
45:  Estado
47:  Complemento do Endereço
```

**Nota:** Esses IDs são específicos da configuração do seu AC. Se campos forem adicionados/removidos, esses IDs mudarão.

## Resultado Esperado

### Estado Antes do Fluxo
- ✗ Compra realizada na Hotmart
- ✗ Dados do cliente não estão no AC
- ✗ Impossível segmentar por cliente/compra no AC

### Estado Depois do Fluxo (Nova Assinatura)
- ✅ Contato criado/atualizado no AC com:
  - Email, nome, telefone
  - Endereço completo
  - Status da assinatura
  - Data de renovação
  - Origem da venda e campanha
- ✅ Pronto para automações (ex: "Enviar email bem-vindo aluno")
- ✅ Dados prontos para relatórios e análise

### Estado Depois do Fluxo (Renovação)
- ✅ Data de renovação atualizada
- ✅ Status confirmado como ACTIVE
- ✅ Histórico de renovação registrado

### Estado Depois do Fluxo (Cancelamento/Encerramento)
- ✅ Status modificado
- ✅ Datas futuras limpas
- ✅ Possibilidade de disparar email "sentiremos sua falta"

## Possíveis Cenários de Erro

| Erro | Causa | Resolução |
|----|----|----|
| Email inválido | Hotmart enviou email malformado | Webhook rejeita e registra erro |
| Contato não encontrado | Primeira compra, novo cliente | AC cria contato automaticamente |
| Campo AC não existe | ID do campo foi deletado/mudou | Contato não atualiza naquele campo |
| Falha de conexão Hotmart | Webhook não consegue validar assinatura | N8N rejeita por segurança |
| Dados incompletos | Faltam campos obrigatórios | Fluxo continua com valores parciais |

## Conceitos-Chave

### Recurrence Number (Hotmart)

Campo essencial para diferenciar nova compra de renovação:
- `recurrence_number == 1` → **Primeira compra** (dispara fluxo completo)
- `recurrence_number > 1` → **Renovação automática** (atualiza apenas status/datas)

**Por quê importante:** Evita duplicar dados e disparar email de boas-vindas em renovações.

### Origem da Venda (src vs xcod)

Dois campos rastreiam a origem:
- `src/sck` → **Canal de venda** (email, chat-youtube, whatsapp-group)
- `xcod` → **Código da campanha** (live-001, workshop-february)

**Uso:** Segmentar clientes por canal e campanha → personalizar comunicação.

### Email como Primary Key

AC busca contato pelo email porque:
- É único por cliente
- Hotmart garante que sempre envia
- Funciona mesmo se cliente mudar nome/telefone

**Risco:** Se cliente usar emails diferentes em Hotmart = cria múltiplas cópias no AC.

### Status da Assinatura

Valores possíveis no campo 27:
- `ACTIVE` → Assinatura válida
- `CANCELED` → Cliente cancelou
- `EXPIRED` → Assinatura expirou naturalmente
- `OUT_OF_SHOPPING_CART` → Abandono de carrinho

Cada status dispara automações diferentes no AC.

## Fluxo de Automação Recomendado no AC

Após essa integração, configure automações no AC:

```
IF status == "ACTIVE" AND recurrence_number == 1
  → Enviar "Bem-vindo aluno!"
  → Adicionar tag "aluno_2026"
  → Incrementar score (+20 pontos)
  → Inscrever em sequência "Onboarding Aluno"

IF status == "EXPIRED"
  → Enviar "Sentiremos sua falta"
  → Remover tag "aluno"
  → Inscrever em sequência "Win-back"

IF status == "OUT_OF_SHOPPING_CART"
  → Enviar "Você deixou um item no carrinho"
  → Incluir link direto para checkout
```
