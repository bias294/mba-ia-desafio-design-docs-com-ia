# ADR-001: Padrão Outbox no MySQL com gravação atômica no `changeStatus`

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (decisão em [09:08] Larissa)
- **Decisores:** Larissa, Diego, Bruno

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) querem ser notificados quando o status de um pedido muda, em vez de consultar `GET /orders` periodicamente ([09:00] Marcos). A transação de `changeStatus` em `src/modules/orders/order.service.ts` já atualiza `orders`, insere em `order_status_history` e movimenta o estoque de `products` ([09:04] Bruno). Colocar uma chamada HTTP nessa transação deixaria um cliente lento travando mudanças de status de outros pedidos, e um cliente fora do ar forçaria rollback da mudança de status ([09:04] Bruno).

## Decisão

Adotar o **padrão Outbox no MySQL existente**. Dentro da mesma transação de `changeStatus`, é inserida uma linha na tabela `webhook_outbox` com o evento já renderizado (snapshot do estado do pedido no momento da mudança). Se a inserção falhar, a transação inteira sofre rollback ([09:40] Bruno). Um worker separado lê a tabela e faz as chamadas HTTP (ver [ADR-005](ADR-005-worker-separado-em-polling.md)).

Detalhes decididos junto:

- A inserção é feita por uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `tx` da transação corrente, sem injetar um repository inteiro no `OrderService` ([09:41] Bruno, [09:41] Diego).
- O filtro de eventos por webhook é aplicado **na inserção**: se nenhum webhook do customer quer aquele status, nada é inserido ([09:34] Bruno).
- O payload é gravado **renderizado (snapshot)** no momento da inserção, para refletir o estado de quando o status mudou ([09:52] Larissa, [09:52] Diego).
- Chaves primárias em **UUID**, seguindo o padrão do projeto ([09:51] Larissa; `prisma/schema.prisma` usa `@default(uuid()) @db.Char(36)` em todos os modelos).
- A tabela tem índices em `status` (pendente, processando, falhou, entregue) e em `created_at` ([09:08] Diego).

## Alternativas Consideradas

1. **Chamada HTTP síncrona dentro do `OrderService`.** Descartada: acopla a latência de terceiros à transação de pedidos e não tem resposta boa para cliente offline ([09:04] Bruno, [09:06] Diego).
2. **Redis Streams ou fila equivalente.** Descartada: exigiria subir infraestrutura nova para um time pequeno; "Redis Cluster pra isso é overengineering" ([09:07] Larissa, [09:07] Diego).
3. **Guardar apenas `order_id` e renderizar o payload no envio.** Descartada: o evento poderia refletir um estado posterior do pedido, não o da mudança ([09:52] Larissa).

## Consequências

**Positivas**
- Consistência: se a transação commitou, o evento existe; se deu rollback, o evento some junto ([09:06] Diego).
- Nenhuma infraestrutura nova: reaproveita MySQL e Prisma.
- Cliente lento ou offline não afeta a mudança de status.

**Negativas / trade-offs**
- Acrescenta uma escrita a mais na transação de `changeStatus`, que já é pesada ([09:04] Bruno).
- A latência passa a depender do polling do worker (ver ADR-005).
- A tabela cresce; arquivar linhas entregues (cerca de 30 dias) fica **fora do escopo** desta feature ([09:08] Diego).
- Falha na inserção do outbox impede a mudança de status. Isso é intencional: "não pode ter caso de status mudar e evento não sair" ([09:40] Bruno).
