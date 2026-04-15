# Etapa 1 – Aquisição e Qualificação Inicial (ActiveCampaign Native)

Registrar um lead (e-mail + metadados) a partir de diferentes fontes, garantindo rastreabilidade (UTM, tags) e disparando ações pós-conversão (página obrigado, remarketing, redirecionamento para WhatsApp).

## Macro arquitetura proposta

| Componente | Responsabilidade | Ferramenta |
|----|----|----|
| **Página de captura** | Hospedar formulário, coletar e-mail e UTM | ActiveCampaign Forms (embed ou página própria) |
| **Banco de dados de leads** | Armazenar contatos, tags, histórico | ActiveCampaign CRM |
| **Pós-submissão** | Disparar redirect, e-mail, webhook | Automação do ActiveCampaign |
| **Remarketing** | Disparar eventos de conversão | Meta Pixel + ActiveCampaign (via integração ou evento) |
| **WhatsApp** | Redirecionamento pós-inscrição | Link SendFlow na página de obrigado

## Fluxo técnico detalhado (ActiveCampaign-first)

### Criação da página de inscrição

- Criar um **formulário nativo do ActiveCampaign** (não um HTML puro do Webflow/WordPress).
- Vantagens:
    - Captura automática de **UTM parameters** (via site tracking). [1](help.activecampaign.com/hc/en-us/articles/115000686370-How-do-I-setup-ActiveCampaign-touchpoints-traffic-sources-for-Attribution)
    - Criação de **contato** e **tag** em tempo real.
    - Redirecionamento nativo pós-submit.

**Configurações de tracking:**

- Instalar o **site tracking snippet** do ActiveCampaign em todas as páginas da landing page. [1](help.activecampaign.com/hc/en-us/articles/115000686370-How-do-I-setup-ActiveCampaign-touchpoints-traffic-sources-for-Attribution)
- O ActiveCampaign automaticamente anexa UTMs como touchpoints ao contato.

### Campos e metadados a serem capturados

| Campo | Obrigatório | Como capturar no ActiveCampaign |
|----|----|----|
| **Email** | Sim | Campo padrão do form |
| **Nome** | Não | Campo personalizado |
| **UTM Parameters** | Sim | Tracking automático via site tracking [1](help.activecampaign.com/hc/en-us/articles/115000686370-How-do-I-setup-ActiveCampaign-touchpoints-traffic-sources-for-Attribution) |
| **Fonte original** | Sim | Tag ex: `origem_facebook_live` |
| **ID ManyChat** | Não | Campo oculto preenchido via URL parameter | 

### Automação pós-submissão

Após o envio do formulário, a **automação do ActiveCampaign** deve:

1. **Adicionar tags** ao contato (ex: lead_live_2026, interesse_curso_x).
2. **Atualizar score** do lead (ex: +10 pontos por inscrição) [2](https://help.activecampaign.com/hc/en-us/articles/20316341549212-Contact-Scoring-in-ActiveCampaign)
3. ~~**Enviar webhook** para sistemas externos (ex: ManyChat, Slack, planilha)~~
4. **Redirecionar o usuário** (controlado pelo próprio form action):
    - Para página de obrigado (se for página ActiveCampaign)
    - Ou para URL externa com parâmetros (ex: SendFlow do WhatsApp)

### Página de Obrigado

Após a submissão do formulário no ActiveCampaign, o usuário é direcionado para uma página de agradecimento que:

1. Confirma a inscrição.
2. Exibe um contador regressivo (ex: 5 segundos).
3. Redireciona automaticamente para um **link de grupo do WhatsApp**.
4. Mantém um link manual ("ou clique aqui") caso o redirecionamento automático falhe.

**Script de redirecionamento automático para WhatsApp**

```html
<!-- COLE ESTE BLOCO INTEIRO ONDE QUISER NA PÁGINA -->
<div id="whatsapp-contador"></div>
<script>
    (function() {
        // ========== SÓ MEXER AQUI ==========
        const URL_WHATSAPP = "https://sndflw.com/i/SEU_ID"; // ⚠️ TROCAR
        const TEMPO_SEGUNDOS = 5;
        // ===================================
        
        let segundos = TEMPO_SEGUNDOS;
        const container = document.getElementById('whatsapp-contador');
        
        // Renderizar o HTML do contador
        container.innerHTML = `
            <div style="text-align: center; font-family: Arial, sans-serif; padding: 20px;">
                <p>✅ Inscrição confirmada! Redirecionando em <strong id="segundos-numero">${segundos}</strong> segundos...</p>
                <div style="width: 100%; background: #e0e0e0; border-radius: 10px; margin-top: 10px;">
                    <div id="barra-progresso" style="width: 0%; height: 8px; background: #25D366; border-radius: 10px;"></div>
                </div>
                <p style="margin-top: 15px; font-size: 12px;">
                    <a href="${URL_WHATSAPP}" id="link-manual" style="color: #25D366;">Clique aqui</a> se não redirecionar automaticamente
                </p>
            </div>
        `;
        
        // Pegar elementos após renderizar
        const segundosSpan = document.getElementById('segundos-numero');
        const barra = document.getElementById('barra-progresso');
        const linkManual = document.getElementById('link-manual');
        
        // Garantir que o link manual tem a URL correta
        if (linkManual) linkManual.href = URL_WHATSAPP;
        
        // Iniciar contador
        const intervalo = setInterval(() => {
            segundos--;
            
            if (segundosSpan) segundosSpan.textContent = segundos;
            if (barra) {
                const percentual = ((TEMPO_SEGUNDOS - segundos) / TEMPO_SEGUNDOS) * 100;
                barra.style.width = percentual + '%';
            }
            
            if (segundos <= 0) {
                clearInterval(intervalo);
                window.location.href = URL_WHATSAPP;
            }
        }, 1000);
        
        // Fallback: redirecionar mesmo se o intervalo travar
        setTimeout(() => {
            if (segundos > 0) window.location.href = URL_WHATSAPP;
        }, TEMPO_SEGUNDOS * 1000 + 500);
    })();
</script>
```


## Links úteis
1. https://help.activecampaign.com/hc/en-us/articles/115000686370-How-do-I-setup-ActiveCampaign-touchpoints-traffic-sources-for-Attribution
2. https://help.activecampaign.com/hc/en-us/articles/20316341549212-Contact-Scoring-in-ActiveCampaign
