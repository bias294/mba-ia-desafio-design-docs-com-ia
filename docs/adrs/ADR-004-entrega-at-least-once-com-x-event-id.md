# ADR-004: Garantia de entrega at-least-once com deduplicação por X-Event-Id

**Status:** Aceito
**Data:** 11-11-2026
**ADRs relacionados:** [ADR-002](ADR-002-retry-com-backoff-e-dlq.md), [ADR-003](ADR-003-hmac-sha256-secret-por-endpoint.md), [ADR-007](ADR-007-formato-do-payload-e-headers-de-envio.md)

## Contexto e Problema

Com retries e um worker em polling, a plataforma pode enviar o mesmo evento mais de uma vez, por exemplo quando o cliente processa o evento mas a resposta se perde por timeout. É preciso declarar aos clientes qual garantia de entrega existe e como lidar com duplicidade.

## Direcionadores da Decisão

- Não perder eventos silenciosamente, o que exige retry ([09:14] Larissa).
- Evitar coordenação complexa entre plataforma e cliente ([09:25] Diego).
- Seguir o padrão de mercado, conhecido dos clientes ([09:25] Diego).
- Dar ao cliente uma forma simples de reconhecer eventos repetidos ([09:25] Diego).
- Permitir identificar qual cadastro de webhook originou cada envio ([09:44] Sofia).

## Alternativas Consideradas

1. At-least-once, com identificador único por evento para deduplicação no cliente.
2. Exactly-once.
3. At-most-once, sem retry.

## Decisão

Opção escolhida: **at-least-once com deduplicação pelo cliente por meio do header `X-Event-Id`**, porque resolve "99% dos casos" sem coordenação entre as duas pontas ([09:25] Diego, [09:26] Larissa).

Cada evento recebe um UUID gerado quando entra na outbox e enviado no header `X-Event-Id`; é único por evento e se mantém igual entre reenvios. O cliente deduplica por esse valor ([09:25] Diego). Também vai no envio o header `X-Webhook-Id`, com o identificador do cadastro de webhook, para clientes com vários endpoints ([09:44] Sofia, [09:45] Diego). Marcos documentará o comportamento de forma destacada no portal de desenvolvedor ([09:26] Marcos).

## Prós e Contras das Alternativas

**At-least-once com identificador por evento**
- Bom: simples de implementar e de explicar.
- Bom: mesmo modelo usado por Stripe e GitHub, citados na reunião ([09:25] Diego).
- Bom: nenhum evento é perdido sem passar pela DLQ.
- Ruim: o cliente precisa deduplicar.

**Exactly-once**
- Bom: elimina duplicidade para o cliente.
- Ruim: exige coordenação dos dois lados e é muito mais complexo ([09:25] Diego).

**At-most-once sem retry**
- Bom: sem duplicidade e sem estado de retry.
- Ruim: incompatível com o requisito de tolerar cliente indisponível ([09:14] Larissa).

## Consequências

O contrato com o cliente fica simples e previsível: pode haver repetição e o identificador do evento é a chave para ignorá-la. A responsabilidade de deduplicar passa para o cliente ([09:25] Sofia), o que torna a documentação no portal parte do trabalho de entrega.

Um timeout do lado do cliente, mesmo com o evento processado, gera reenvio; esse caso é coberto pela deduplicação, não por mudança no servidor.

## Referências

- `src/middlewares/request-logger.middleware.ts:2` (uso do pacote de UUID já adotado no projeto)
