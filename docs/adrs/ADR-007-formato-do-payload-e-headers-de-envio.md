# ADR-007: Formato do payload, headers de envio, timeout e limite de tamanho

**Status:** Aceito
**Data:** 11-11-2026
**ADRs relacionados:** [ADR-003](ADR-003-hmac-sha256-secret-por-endpoint.md), [ADR-004](ADR-004-entrega-at-least-once-com-x-event-id.md), [ADR-001](ADR-001-outbox-no-mysql.md)

## Contexto e Problema

Os clientes precisam de um contrato de envio estável e enxuto para integrar. As escolhas de conteúdo, headers, tempo limite e teto de tamanho foram feitas em blocos separados da reunião, e precisam de um registro único para que o contrato fique em um só lugar.

## Direcionadores da Decisão

- Manter o payload enxuto, para o cliente consultar detalhes só quando precisar ([09:43] Diego).
- Permitir que o cliente autentique, deduplique, identifique o cadastro e detecte replay apenas pelos headers ([09:44] Diego).
- Não deixar um cliente lento prender o worker ([09:42] Sofia).
- Tratar como erro, e não mascarar, um evento de tamanho anormal ([09:23] Sofia).

## Alternativas Consideradas

1. Payload enxuto, sem a lista de itens, com timeout de 10 segundos e teto de 64 KB.
2. Payload completo, incluindo os itens do pedido.
3. Truncar payloads que passam do limite em vez de recusá-los.

## Decisão

Opção escolhida: **payload enxuto, com cinco headers definidos, timeout de 10 segundos e limite de 64 KB que recusa o envio** ([09:43] Diego, [09:24] Larissa).

**Conteúdo.** JSON com identificador do evento, tipo do evento (`order.status_changed`), horário em ISO 8601, identificador e número do pedido, status de origem e de destino, identificador do cliente e os campos básicos do pedido, como o valor total. Não inclui os itens; quem precisar consulta o pedido depois ([09:43] Diego).

**Headers.** `Content-Type: application/json`, `X-Event-Id` (identificador do evento, [ADR-004](ADR-004-entrega-at-least-once-com-x-event-id.md)), `X-Signature` (assinatura do corpo, [ADR-003](ADR-003-hmac-sha256-secret-por-endpoint.md)), `X-Timestamp` (momento do envio, para detectar replay) e `X-Webhook-Id` (identificador do cadastro) ([09:44] Diego, [09:44] Sofia, [09:45] Diego).

**Timeout e tamanho.** Uma chamada sem resposta em 10 segundos é falha e entra no fluxo de retry ([09:42] Diego). Payload acima de 64 KB não é enviado e gera erro, sem truncar ([09:23] Sofia, [09:24] Diego, [09:24] Larissa). Esses limites são requisitos não funcionais, não decisões arquiteturais separadas ([09:24] Larissa); estão aqui apenas para manter o contrato num único lugar.

## Prós e Contras das Alternativas

**Payload enxuto com limites explícitos**
- Bom: mensagens pequenas e previsíveis, que mantêm leve o snapshot guardado na outbox.
- Bom: o cliente cobre autenticação, deduplicação e detecção de replay só com os headers.
- Ruim: quem precisa dos itens faz uma segunda chamada à API.

**Payload completo com itens**
- Bom: o cliente recebe tudo de uma vez.
- Ruim: infla as mensagens e aproxima os eventos do teto de tamanho ([09:43] Diego, [09:44] Bruno).

**Truncar acima do limite**
- Bom: o envio nunca falha por tamanho.
- Ruim: entrega dados incompletos sem aviso; "se chegou nesse tamanho, tem algo errado" ([09:23] Sofia).

## Consequências

O contrato de saída fica pequeno, documentável e estável. Um cliente que demora mais de 10 segundos para responder é tratado como falha mesmo que tenha processado o evento, o que reforça a necessidade da deduplicação do [ADR-004](ADR-004-entrega-at-least-once-com-x-event-id.md). O teto de 64 KB é folgado para os eventos previstos; um evento que o ultrapasse indica problema e não é enviado.

## Referências

- `src/modules/orders/order.schemas.ts:19` (valores de status de pedido que alimentam os campos de status do payload)
- `prisma/schema.prisma:74` (campos básicos do pedido que compõem o payload)
