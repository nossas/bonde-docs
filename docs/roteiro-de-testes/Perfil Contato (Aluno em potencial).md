# Roteiro de Testes – Perfil: Contato (Aluno em potencial)

## Objetivo

Validar toda a jornada do contato desde o primeiro acesso via link com UTM, passando pelo pré‑live, live, pós‑live e finally a conversão de venda com novo UTM, garantindo que os dados sejam corretamente enviados ao CRM (ActiveCampaign) e que os redirecionamentos e disparos de e‑mail funcionem.

## Cenário 1: Acesso ao link com UTM e preenchimento do formulário

**Pré-condições:**

- Link válido com parâmetros UTM (ex: `utm_source=facebook&utm_medium=social&utm_campaign=curso_a`)
- ActiveCampaign configurado para receber UTMs no contato

**Passos:**

| Ação | Resultado esperado |
| ---- | ------------------ |
| 1. Contato acessa o link (ex: via WhatsApp, Instagram, e-mail) | A página do formulário de inscrição é carregada corretamente |
| 2. Contato preenche todos os campos obrigatórios e envia | Redirecionamento para página "Obrigado" e após 5 segundos, redireciona para o grupo do WhatsApp (link válido) |
| 3. Verificar no ActiveCampaign | O contato foi criado/atualizado com os dados preenchidos + os UTMs da URL (utm_source, utm_medium, utm_campaign, etc.) |

> ✅ **Critério de aceite:** UTMs persistentes e corretamente associados ao contato no CRM.

## Cenário 2: Simulação da comunicação pré‑live e gatilho da Live

**Contexto:**

Após o preenchimento, inicia‑se uma estratégia de comunicação (e‑mails, lembretes) até o momento da **Live (gatilho de venda)**.

| Ação simulada | Resultado esperado |
| ------------- | ------------------ |
| 1. Sistema dispara e‑mails automáticos (ex: confirmação, lembrete 1 dia antes) | E‑mails entregues com conteúdo correto |
| 2. Contato assiste à Live (simular evento no sistema) | O sistema marca que o contato participou (campo/tag no ActiveCampaign) |
| 3. Ao final da Live, novo fluxo de comunicação é iniciado (pós‑live) | E‑mails de follow‑up enviados com link de venda |

> ✅ **Critério de aceite:** O contato entra no fluxo correto de pós‑live automaticamente.

## Cenário 3: Acesso ao link de venda com novos UTMs

**Pré‑condições:**

- Contato já está no fluxo pós‑live
- Link de venda gerado com UTMs específicos (ex: `utm_source=chat_live&utm_campaign=curso-a`)

**Passos:**

| Ação | Resultado esperado |
| ---- | ------------------ |
| 1. Contato clica no link de venda (recebido por e‑mail, WhatsApp, etc.) | Página de checkout/assinatura carregada |
| 2. Contato conclui a assinatura (caminho do sucesso) | <ul><li>Página de confirmação de pagamento/assinatura</li><li>E‑mail automático enviado com link de acesso à Comunidade (Circle)</li><li>Dados de assinatura (plano, data, status) atualizados no ActiveCampaign</li> |

> ✅ **Critério de aceite:**
> - Email de acesso a Circle chega corretament.
> - ActiveCampaign reflete: contato com tag "Assinante", plano adquirido, UTMs de venda registrados