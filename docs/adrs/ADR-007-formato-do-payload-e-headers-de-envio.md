# ADR-007: Formato do payload, headers de envio, timeout e limite de tamanho

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (decisões entre [09:23] e [09:45])
- **Decisores:** Diego, Sofia, Marcos, Larissa

## Contexto

Os clientes precisam de um contrato estável e enxuto para integrar. O "como" do envio (campos, headers, teto de tamanho e tempo limite) foi decidido em blocos separados na reunião e precisa de um registro único.

## Decisão

**Payload** em JSON, com os campos `event_id`, `event_type` (ex.: `"order.status_changed"`), `timestamp` (ISO 8601), `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e os campos básicos do pedido, como `total_cents`. **Não inclui `items`**: o cliente consulta `GET /orders/:id` se precisar de detalhes ([09:43] Diego).

**Headers** ([09:44] Diego, [09:44] Sofia, [09:45] Diego):

| Header | Valor |
| --- | --- |
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID do evento ([ADR-004](ADR-004-entrega-at-least-once-com-x-event-id.md)) |
| `X-Signature` | HMAC-SHA256 do corpo ([ADR-003](ADR-003-hmac-sha256-secret-por-endpoint.md)) |
| `X-Timestamp` | Momento do envio, para detecção de replay |
| `X-Webhook-Id` | Id do cadastro de webhook |

**Timeout** da chamada HTTP: **10 segundos**; excedido, vira falha e segue para retry ([09:42] Diego).

**Limite de payload**: **64 KB**; se ultrapassar, o evento **não é enviado** e gera erro, sem truncar ([09:23] Sofia, [09:24] Diego, [09:24] Larissa).

## Alternativas Consideradas

1. **Incluir `items` no payload.** Descartada para manter o payload enxuto ([09:43] Diego, [09:44] Bruno).
2. **Truncar payloads acima do limite.** Descartada: "se chegou nesse tamanho, tem algo errado" ([09:23] Sofia).
3. **Teto maior que 64 KB (a discussão citou 500 KB como exemplo extremo).** Descartada: 64 KB já é um teto generoso e nenhum evento chega perto ([09:24] Diego).

## Consequências

**Positivas**
- Payload pequeno e previsível; o snapshot na outbox ([ADR-001](ADR-001-outbox-no-mysql.md)) fica leve.
- O cliente consegue autenticar, deduplicar, identificar o cadastro e detectar replay só pelos headers.

**Negativas / trade-offs**
- O cliente que precisa dos itens faz uma segunda chamada à API.
- A regra de 64 KB e a de URL https são requisitos não funcionais, não decisões arquiteturais separadas ([09:24] Larissa); estão neste ADR apenas para que o contrato de envio fique num só lugar.
- Cliente que leva mais de 10 s para responder é tratado como falha, mesmo que processe o evento (reforça a necessidade de deduplicação, ADR-004).
