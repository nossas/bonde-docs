# Etapa 4 - Venda (Pós-Live)

Converter leads em alunos, rastreando a origem da venda (UTM) e atualizando o ActiveCampaign com o status de compra (aprovado/cancelado), além de gerenciar a tag de aluno.

## Canais de Venda e Tracking

| Canal | Ferramenta | Como faz tracking |
|----|----|----|
| **Grupo do WhatsApp** | SendFlow | Link com UTM → Hotmart |
| **E-mail** | ActiveCampaign | Automação: clique no link da compra dispara evento |
| **Chat do YouTube** | YouTube (link na descrição/chat) | Link com UTM → Hotmart |
| **Home do BONDE** | BONDE (site institucional) | Link com UTM → Hotmart |



### Webhook (fluxo N8N)

#### Compra Aprovada (Nova Assinatura e Renovação)

A diferença na Compra Aprovada para definir se é uma Nova Assinatura ou Renovação é o campo `purchase.recurrence_number`, sempre que ele for 1 é uma nova assinatura.

Em **Nova Assinatura** armazenamos também a campanha (`purchase.origin.xcod`) e a origem da compra (`purchase.origin.src` ou `purchase.origin.sck`), que varia entre os 4 canais *Grupo de WhatsApp*, *E-mail*, *Chat do Youtube* e *Home do BONDE*.

```json
{
    "data": {
        "product": {
            "support_email": "suporte@comunidadebonde.com.br",
            "has_co_production": false,
            "name": "Comunidade BONDE",
            "warranty_date": "2026-05-06T00:00:00Z",
            "is_physical_product": false,
            "id": 12345,
            "ucode": "comunidade-bonde-2026-001",
            "content": {
                "has_physical_products": false,
                "products": [
                    {
                        "name": "Comunidade BONDE - Assinatura Mensal",
                        "is_physical_product": false,
                        "id": 12345,
                        "ucode": "assinatura-bonde-mensal-001"
                    }
                ]
            }
        },
        "commissions": [
            {
                "currency_value": "BRL",
                "source": "PRODUCER",
                "value": 45.00
            }
        ],
        "purchase": {
            "recurrence_number": 1,
            "original_offer_price": {
                "currency_value": "BRL",
                "value": 540.00
            },
            "checkout_country": {
                "iso": "BR",
                "name": "Brasil"
            },
            "sckPaymentLink": "sckPaymentLinkTest",
            "order_bump": null,
            "variants": null,
            "approved_date": 1712345678000,
            "offer": {
                "code": "comunidade-bonde-mensal",
                "coupon_code": null
            },
            "origin": {
                "src": "chat-youtube",
                "sck": "chat-youtube",
                "xcod": "live-001"
            },
            "is_funnel": false,
            "event_tickets": null,
            "order_date": 1712345675000,
            "price": {
                "currency_value": "BRL",
                "value": 540.00
            },
            "payment": {
                "installments_number": 12,
                "type": "CREDIT_CARD"
            },
            "full_price": {
                "currency_value": "BRL",
                "value": 540.00
            },
            "business_model": "S",
            "transaction": "HPCOMUNIDADE001",
            "status": "APPROVED"
        },
        "affiliates": [],
        "producer": {
            "legal_nature": "Pessoa Física",
            "document": "12345678965",
            "name": "Produtor Comunidade BONDE"
        },
        "subscription": {
            "subscriber": {
                "code": "SUB-BONDE-001"
            },
            "plan": {
                "name": "Comunidade BONDE - Mensal",
                "id": 1001
            },
            "status": "ACTIVE"
        },
        "buyer": {
            "checkout_phone_code": "55",
            "address": {
                "zipcode": "01001000",
                "country": "Brasil",
                "number": "100",
                "address": "Avenida Paulista",
                "city": "São Paulo",
                "state": "São Paulo",
                "neighborhood": "Bela Vista",
                "complement": null,
                "country_iso": "BR"
            },
            "document": "12345678900",
            "name": "Kaique Garcia",
            "last_name": "Garcia",
            "checkout_phone": "11999999999",
            "first_name": "Kaique",
            "email": "kaique@test.com",
            "document_type": "CPF"
        }
    },
    "hottok": "WIF9p3U00RKzHeUiCGzdka9IEREght683f35ca-83f4-4b52-bb0f-237ba79683d6",
    "id": "assinatura-bonde-2026-001",
    "creation_date": 1712345678000,
    "event": "PURCHASE_APPROVED",
    "version": "2.0.0"
}
```

#### Compra cancelada

Compra cancelada não define remove de imediato o acesso a plataforma, nesse momento o nosso fluxo trabalha apenas o acesso até o final, ou seja último pagamento aprovado somado a 1 mês. Sempre que um contato realizar o cancelamento ele vai possuir data de cancelamento (`subscription_cancelled_date`).

```json
{
    "id": "d4fb88a5-71ff-4360-9f93-8d36ae5a9fa5",
    "creation_date": 1775520628550,
    "event": "PURCHASE_CANCELED",
    "version": "2.0.0",
    "data": {
        "product": {
            "id": 0,
            "ucode": "fb056612-bcc6-4217-9e6d-2a5d1110ac2f",
            "name": "Produto test postback2",
            "warranty_date": "2017-12-27T00:00:00Z",
            "support_email": "support@hotmart.com.br",
            "has_co_production": false,
            "is_physical_product": false,
            "content": {
                "has_physical_products": true,
                "products": [
                    {
                        "id": 4774438,
                        "ucode": "559fef42-3406-4d82-b775-d09bd33936b1",
                        "name": "How to Make Clear Ice",
                        "is_physical_product": false
                    },
                    {
                        "id": 4999597,
                        "ucode": "099e7644-b7d1-43d6-82a9-ec6be0118a4b",
                        "name": "Organizador de Poeira",
                        "is_physical_product": true
                    }
                ]
            }
        },
        "affiliates": [
            {
                "affiliate_code": "Q58388177J",
                "name": "Affiliate name"
            }
        ],
        "buyer": {
            "email": "kaique@test.com",
            "name": "Teste Comprador",
            "first_name": "Teste",
            "last_name": "Comprador",
            "checkout_phone_code": "999999999",
            "checkout_phone": "99999999900",
            "address": {
                "city": "Uberlândia",
                "country": "Brasil",
                "country_iso": "BR",
                "state": "Minas Gerais",
                "neighborhood": "Tubalina",
                "zipcode": "38400123",
                "address": "Avenida Francisco Galassi",
                "number": "10",
                "complement": "Perto do shopping"
            },
            "document": "69526128664",
            "document_type": "CPF"
        },
        "producer": {
            "name": "Producer Test Name",
            "document": "12345678965",
            "legal_nature": "Pessoa Física"
        },
        "commissions": [
            {
                "value": 149.5,
                "source": "MARKETPLACE",
                "currency_value": "BRL"
            },
            {
                "value": 1350.5,
                "source": "PRODUCER",
                "currency_value": "BRL"
            }
        ],
        "purchase": {
            "approved_date": 1712345678000,
            "full_price": {
                "value": 1500,
                "currency_value": "BRL"
            },
            "price": {
                "value": 1500,
                "currency_value": "BRL"
            },
            "checkout_country": {
                "name": "Brasil",
                "iso": "BR"
            },
            "order_bump": {
                "is_order_bump": true,
                "parent_purchase_transaction": "HP02316330308193"
            },
            "event_tickets": {
                "amount": 1775520628515
            },
            "original_offer_price": {
                "value": 1500,
                "currency_value": "BRL"
            },
            "order_date": 1511783344000,
            "status": "CANCELED",
            "transaction": "HP16015479281022",
            "payment": {
                "installments_number": 12,
                "type": "CREDIT_CARD"
            },
            "offer": {
                "code": "test",
                "coupon_code": "SHHUHA"
            },
            "sckPaymentLink": "sckPaymentLinkTest",
            "is_funnel": false,
            "business_model": "I",
            "variants": {
                "sku": "HTM_SANDBOX-01",
                "attributes": [
                    {
                        "name": "Tamanho",
                        "value": "Médio"
                    },
                    {
                        "name": "Cor",
                        "value": "Azul"
                    }
                ]
            }
        },
        "shipping": {
            "cost": {
                "value": "25.50",
                "currency_value": "BRL"
            },
            "estimated_delivery_days": 7,
            "carrier": {
                "name": "CORREIOS",
                "service": "SEDEX"
            },
            "fulfillment": {
                "service": "MANUAL"
            }
        },
        "subscription": {
            "status": "ACTIVE",
            "plan": {
                "id": 123,
                "name": "plano de teste"
            },
            "subscriber": {
                "code": "I9OT62C3"
            }
        }
    }
}
```

#### Assinatura encerrada

Toda assinatura iniciada irá encerrar com status de `EXPIRED`, isso encerra o ciclo, a data em que ela encerrada será sempre a data final da assinatura (`subscription_end_date`) que inicia sendo o total de ciclos da assinatura, geralmente 12 meses, e encerra no caso do cancelamento sendo o período de acesso até o momento do cancelamento, ou seja último pagamento válido somado 1 mês.

```
{
  "id": "08f9d001-3208-432b-840f-7af9f9c82b18",
  "creation_date": 1775520628142,
  "event": "SUBSCRIPTION_CANCELLATION",
  "version": "2.0.0",
  "data": {
    "actual_recurrence_value": 64.9,
    "cancellation_date": 1609181285500,
    "date_next_charge": 1617105600000,
    "product": {
      "id": 788921,
      "name": "Product name com ç e á"
    },
    "subscriber": {
      "code": "0000aaaa",
      "name": "User name",
      "email": "kaique@test.com",
      "phone": {
        "dddPhone": "",
        "phone": "",
        "dddCell": "",
        "cell": ""
      }
    },
    "subscription": {
      "id": 4148584,
      "plan": {
        "id": 114680,
        "name": "Subscription Plan Name"
      }
    }
  }
}
```

#### Abandono do carrinho / Checkout iniciado

O hotmart possui um evento **Abandono do carrinho** que é disparado quando uma pessoa preenche os dados na página de pagamento e não finaliza a compra. [1](https://developers.hotmart.com/docs/pt-BR/2.0.0/webhook/cart-abandonment-webhook/#evento-de-abandono-de-carrinho).

```json
{
  "id": "d5c684c9-8ac4-418b-8da4-a774064c3f78",
  "creation_date": 1775520628184,
  "event": "PURCHASE_OUT_OF_SHOPPING_CART",
  "version": "2.0.0",
  "data": {
    "affiliate": false,
    "product": {
      "id": 123456,
      "name": "Produto test postback2 com ç e á"
    },
    "buyer_ip": "127.0.0.1",
    "buyer": {
      "name": "Postback2 teste",
      "email": "kaique@test.com",
      "phone": "999999999"
    },
    "offer": {
      "code": "n82b9jqz"
    },
    "checkout_country": {
      "name": "Brasil",
      "iso": "BR"
    }
  }
}
```

## Links úteis

1. https://developers.hotmart.com/docs/pt-BR/2.0.0/webhook/cart-abandonment-webhook/#evento-de-abandono-de-carrinho