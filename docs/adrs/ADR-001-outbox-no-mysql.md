# ADR-001: Padrão Outbox no MySQL com gravação atômica na mudança de status

**Status:** Aceito
**Data:** 11-11-2026
**ADRs relacionados:** [ADR-002](ADR-002-retry-com-backoff-e-dlq.md), [ADR-005](ADR-005-worker-separado-em-polling.md)

## Contexto e Problema

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) querem ser notificados quando o status de um pedido muda, em vez de consultar a listagem de pedidos de tempos em tempos ([09:00] Marcos). Hoje a plataforma não tem nenhum mecanismo de notificação externa, evento, fila ou webhook.

A transação de mudança de status já é pesada: atualiza o pedido, grava o histórico de status e movimenta o estoque dos produtos ([09:04] Bruno). Se uma chamada HTTP ao cliente entrasse nessa transação, um cliente lento travaria a mudança de status de outros pedidos. Se o cliente estivesse fora do ar, a única saída seria desfazer a mudança de status, o que não é aceitável ([09:04] Bruno).

O problema é como garantir que toda mudança de status confirmada produza um evento, sem acoplar a transação de pedidos à disponibilidade de terceiros.

## Direcionadores da Decisão

- Consistência: se o status mudou, o evento deve existir; se houve rollback, o evento não pode existir ([09:06] Diego).
- Isolamento: a latência ou a queda de um cliente não pode afetar a mudança de status.
- Time pequeno: evitar infraestrutura nova para operar ([09:07] Diego).
- Fidelidade do evento: o conteúdo deve refletir o estado do pedido no momento da mudança ([09:52] Larissa).
- Economia de armazenamento: não gerar evento para quem não quer ouvir aquele status ([09:34] Bruno).

## Alternativas Consideradas

1. Padrão Outbox no MySQL existente, com um worker separado lendo a tabela.
2. Chamada HTTP síncrona dentro da transação de mudança de status.
3. Redis Streams ou fila externa equivalente.

## Decisão

Opção escolhida: **Outbox no MySQL**, porque é a única que dá consistência atômica sem infraestrutura nova ([09:08] Larissa).

Dentro da mesma transação que atualiza o pedido e o histórico, a plataforma grava o evento numa tabela de outbox. Se a gravação falhar, toda a transação sofre rollback: "não pode ter caso de status mudar e evento não sair" ([09:40] Bruno). A gravação é feita por uma função que recebe o cliente da transação corrente, sem injetar um repositório inteiro no serviço de pedidos ([09:41] Bruno, [09:41] Diego).

Três detalhes foram fechados junto:
- O filtro por status acontece na inserção: se nenhum webhook do cliente ouve o novo status, nada é gravado ([09:34] Bruno).
- O conteúdo é gravado já renderizado (snapshot), e não recalculado no envio ([09:52] Larissa, [09:52] Diego).
- Os identificadores são UUID, seguindo o padrão do projeto ([09:51] Larissa).

## Prós e Contras das Alternativas

**Outbox no MySQL**
- Bom: consistência garantida entre a mudança de status e o evento.
- Bom: sem nova infraestrutura, reaproveitando banco e ORM existentes.
- Bom: cliente lento ou offline não afeta pedidos.
- Ruim: uma escrita a mais na transação, que já é pesada.

**Chamada HTTP síncrona**
- Bom: implementação simples e entrega imediata.
- Ruim: a latência de terceiros trava a transação de pedidos.
- Ruim: cliente offline obriga a escolher entre perder o evento e desfazer a mudança de status.

**Redis Streams ou fila externa**
- Bom: desacopla produtor e consumidor com ferramenta própria para isso.
- Ruim: infraestrutura nova para um time pequeno, considerada excessiva ([09:07] Larissa, [09:07] Diego).
- Ruim: a consistência com a transação do banco exigiria coordenação extra.

## Consequências

A consistência passa a ser garantida pelo banco: confirmou a transação, o evento existe. Em troca, a mudança de status fica um pouco mais cara e a falha na gravação do evento impede a mudança, comportamento intencional.

A latência de notificação passa a depender do ritmo de leitura do worker ([ADR-005](ADR-005-worker-separado-em-polling.md)). A tabela cresce com o tempo; arquivar eventos entregues (cerca de 30 dias) ficou fora do escopo desta feature ([09:08] Diego). O formato de tabela, índices por status e por data de criação foram citados na reunião e ficam para o FDD.

## Referências

- `src/modules/orders/order.service.ts:126` (mudança de status e sua transação)
- `src/modules/orders/order.service.ts:159` (gravação do histórico dentro da transação)
- `prisma/schema.prisma:26` (padrão de identificador UUID dos modelos)
