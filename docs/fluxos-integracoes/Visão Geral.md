# Fluxos de Integração com N8N

Orquestração e sincronização de dados entre plataformas (ActiveCampaign, Hotmart, Circle, Keycloak, etc.) através de webhooks, gatilhos e automações centralizadas no N8N.

## O que é N8N neste contexto

O **N8N** funciona como o **hub central de integrações**, permitindo:

- **Captura de eventos** de diferentes fontes (webhooks do Hotmart, ActiveCampaign, Keycloak, Eventos Postgres, etc.)
- **Transformação e validação** de dados entre formatos diferentes
- **Sincronização em tempo real** entre sistemas (ex: nova compra no Hotmart → atualizar contato no ActiveCampaign)
- **Disparo de ações** automáticas baseadas em gatilhos (ex: aprovação de compra → ativar acesso no Circle)
- **Rastreamento de origem** (UTM, campanhas, fonte de tráfego)

## Integrações Principais

| Fluxo | Gatilho | Sistema de Origem | Ações Realizadas |
|----|----|----|----|----|
| **Checkout Hotmart -> ActiveCampaign:<br/>Enriquecimento da Base de Contatos** | - Assinatura aprovada/compra confirmada;<br/>- Assinatura cancelada | Hotmart (Webhook) | Atualiza os dados e metadados do lead no ActiveCampaign com base na alteração de status de assinatura da Hotmart |
| **Checkout Hotmart -> Keycloak:<br/>Gerenciamento de Acesso** | - Assinatura aprovada/compra confirmada;<br/>- Assinatura cancelada | Hotmart (Webhook) | - Cria o usuário no Keycloak;<br/>- Concede e remove acesso à Circle via grupos do Keycloak |
| **Ativação de Conta: Integração Keycloak + Circle** | Usuário é **adicionado** ao grupo de acesso 'Assinantes Circle' no Keycloak | Postgres (Trigger) / Keycloak| Vincula o usuário no Circle a atualiza os atributos no Keycloak |
| **Desativação de Conta: Integração Keycloak + Circle** | Usuário é **removido** do grupo 'Assinantes Circle' no Keycloak | Postgres (Trigger) / Keycloak | Derruba a sessão no Circle e faz logout no Keycloak |
| **Sincronização de Perfil: Circle -> Keycloak** | Alteração de campos do perfil na Circle | Circle (Webhooks) | Atualiza campos de atributos relacionados ao perfil do usuário no Keycloak |

## Macro Arquitetura

```
┌───────────────────────────┐
│   ORIGEM                  │
├───────────────────────────┤
│ • Hotmart (webhook)       │
│ • Circle (webhook)        │
│ • Keycloak (Pg Triggers)  │
└──────────┬────────────────┘
           │
           ▼
┌──────────────────────────┐
│        N8N (Hub)         │
├──────────────────────────┤
│ • Validação de dados     │
│ • Transformação de dados │
│ • Orquestração de fluxos │
│ • Rastreamento de origem │
└──────────┬───────────────┘
           │
           ▼
┌─────────────────────────────┐
│  SISTEMAS DE DESTINO        │
├─────────────────────────────┤
│ • ActiveCampaign (CRM)      │
│ • Circle (Comunidade)       │
│ • Keycloak (Autenticação)   │
│ • Postgres                  │
└─────────────────────────────┘
```

## Fluxo de Dados - Exemplo: Nova Compra

1. **Gatilho**: Hotmart envia webhook `purchase.approved` para N8N
2. **Validação**: N8N valida estrutura JSON, campos obrigatórios, campos customizados
3. **Enriquecimento**: N8N captura origem da compra (`purchase.origin.src`, `purchase.origin.xcod`)
4. **Sincronização**:
   - Atualiza contato no **ActiveCampaign** (nova tag `aluno_2026`, atualiza score)
   - Ativa acesso no **Circle** (cria usuário, adiciona a grupo)
   - Sincroniza credenciais no **Keycloak** (cria ou atualiza usuário com ID único)
5. **Log**: Armazena registro da transação para auditoria


## Próximas Etapas da Documentação

- ✅ Visão Geral (você está aqui)
- ✅ Hotmart -> ActiveCampaign: Enriqueciento de Base
- ✅ Hotmart -> Keycloak: Gerenciamento de Acesso
- ✅ Ativação de Conta: Integração Keycloak + Circle
- ✅ Desativação de Conta: Keycloak + Circle
- ⏳ Sincronização de Perfil: Circle -> Keycloak
