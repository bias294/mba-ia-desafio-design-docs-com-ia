# ADR-004: Garantia de entrega at-least-once com deduplicação por `X-Event-Id`

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (decisão em [09:26] Larissa)
- **Decisores:** Diego, Larissa, Sofia, Marcos

## Contexto

Com retries e worker em polling, a plataforma pode enviar o mesmo evento mais de uma vez (por exemplo, o cliente processa mas a resposta se perde por timeout). É preciso declarar o contrato de entrega para os clientes.

## Decisão

- Garantir **at-least-once**: o cliente pode receber o mesmo evento duas vezes e deve estar preparado ([09:24] Diego).
- Cada evento recebe um **UUID gerado quando entra na outbox**, enviado no header **`X-Event-Id`**. O cliente deduplica por esse valor ([09:25] Diego).
- Marcos documentará o comportamento de forma destacada no portal de desenvolvedor ([09:26] Marcos).
- Também vai no envio o header `X-Webhook-Id`, com o id do cadastro de webhook, para clientes com vários endpoints saberem qual cadastro originou o envio ([09:44] Sofia, [09:45] Diego).

## Alternativas Consideradas

1. **Exactly-once.** Descartada: exigiria coordenação dos dois lados e é muito mais complexo; at-least-once com `event_id` "resolve 99% dos casos" ([09:25] Diego).
2. **At-most-once (sem retry).** Incompatível com o requisito de retry com backoff ([09:14] Larissa, [09:15] Diego).

## Consequências

**Positivas**
- Modelo simples, igual ao de Stripe e GitHub, citados na reunião ([09:25] Diego).
- Nenhum evento é perdido silenciosamente.

**Negativas / trade-offs**
- A responsabilidade de deduplicar passa para o cliente ([09:25] Sofia).
- O contrato precisa ser comunicado e documentado no portal (responsabilidade de Marcos).
