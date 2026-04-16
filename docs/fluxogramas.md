## Fluxo completo da jornada do Contato (Aluno potencial)

```mermaid
flowchart TD
    A[📱 Link com UTM1<br>origem: rede social/WhatsApp/e-mail] --> B[📝 Página de formulário de inscrição]
    
    B --> C{Preenche e envia?}
    C -->|Sim| D[✅ Página Obrigado]
    C -->|Não| B
    
    D --> E[⏱️ Redirecionamento automático<br>após X segundos]
    E --> F[📱 Grupo de WhatsApp]
    
    F --> G[📧 Estratégia de comunicação<br>pré-Live]
    G --> H[🎥 Live - Gatilho de venda]
    
    H --> I[📧 Estratégia de comunicação<br>pós-Live]
    I --> J[🔗 Link de venda com UTM2<br>origem: chat_live, campanha curso-a]
    
    J --> K[💳 Página de checkout/assinatura]
    K --> L{Conclui assinatura?}
    
    L -->|Sim - Caminho do sucesso| M[✅ Confirmação de assinatura]
    L -->|Não| K
    
    M --> N[📧 E-mail com link de acesso<br>à Comunidade Circle]
    M --> O[📊 ActiveCampaign atualizado<br>• Dados da assinatura<br>• UTM2 de venda]
    
    N --> P[🌐 Contato acessa a Comunidade]
    O --> Q[🏁 Fim da jornada de conversão]
```

## Fluxo detalhado com os dois momentos de UTM

```mermaid
flowchart LR
    subgraph Fase1[Fase 1: Aquisição]
        A1[Link UTM1<br>facebook, Instagram, WhatsApp, e-mail]
        A2[Formulário]
        A3[Obrigado → Grupo WhatsApp]
        A4[ActiveCampaign<br>salva UTM1 + dados]
    end
    
    subgraph Fase2[Fase 2: Pré-live → Live]
        B1[Comunicação automática<br>e-mails de aquecimento]
        B2[🎥 Live<br>gatilho de venda]
        B3[Comunicação pós-live<br>e-mails de follow-up]
    end
    
    subgraph Fase3[Fase 3: Venda]
        C1[Link UTM2<br>chat_live, campanha curso-a]
        C2[Checkout]
        C3[Assinatura concluída]
        C4[E-mail Circle]
        C5[ActiveCampaign<br>salva assinatura + UTM2]
    end
    
    A1 --> A2 --> A3 --> A4 --> B1 --> B2 --> B3 --> C1 --> C2 --> C3 --> C4
    C3 --> C5
```

## Fluxo de dados no ActiveCampaign

```mermaid
flowchart TD
    subgraph CRM[ActiveCampaign - Contato]
        D1[📝 Dados pessoais<br>nome, e-mail, telefone]
        D2[🏷️ UTMs de origem<br>utm_source, utm_medium, utm_campaign]
        D3[🏷️ Tags<br>• assistiu_live<br>• assinante_curso_a]
        D4[📊 Dados de assinatura<br>plano, valor, data, status]
        D5[🔗 Link de acesso Circle<br>campo personalizado]
    end
    
    E1[Formulário preenchido] --> D1
    E1 --> D2
    
    E2[Participação na Live] --> D3
    
    E3[Assinatura concluída] --> D4
    E3 --> D5
    E3 --> D3
```

## Fluxo de e-mails e comunicações

```mermaid
flowchart TD
    C[Contato] --> E1[📧 E-mail 1: Confirmação de inscrição]
    C --> E2[📧 E-mail 2: Lembrete Live - 1 dia]
    C --> E3[📧 E-mail 3: Lembrete Live - 1 hora]
    C --> E4[🎥 Live]
    C --> E5[📧 E-mail 4: Obrigado por assistir + link venda UTM2]
    C --> E6[📧 E-mail 5: Última chance - link venda UTM2]
    C --> E7[💳 Compra]
    C --> E8[📧 E-mail 6: Bem-vindo à Circle + link de acesso]
```

## Fluxo de decisão da Live para a Venda

```mermaid
flowchart TD
    Start[🎥 Live termina] --> Question{Contato comprou?}
    
    Question -->|Sim| Success[✅ Entra no fluxo Circle<br>CRM marcado como assinante]
    Question -->|Não| FollowUp[📧 Entra em fluxo de nutrição<br>e-mails com link UTM2]
    
    FollowUp --> Click{Clicou no link de venda?}
    Click -->|Sim| Checkout[💳 Página de checkout]
    Click -->|Não| FollowUp
    
    Checkout --> Buy{Comprou?}
    Buy -->|Sim| Success
    Buy -->|Não| Abandoned[🔄 Fluxo de recuperação<br>carrinho abandonado]
```