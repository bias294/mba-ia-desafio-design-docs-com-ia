# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| Autor | Larissa (Tech Lead), responsável por abrir o documento de design ([09:50] Larissa) |
| Status | Em revisão |
| Data | 2026-11-11 |
| Revisores | Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Segurança), Marcos (PM) |
| Documentos relacionados | [PRD](PRD.md), [FDD](FDD.md), [ADRs](adrs/README.md), [Tracker](TRACKER.md) |

## 1. Resumo executivo (TL;DR)

Propomos notificar clientes B2B por **webhooks outbound** sempre que o status de um pedido mudar. O evento é gravado numa tabela **outbox no MySQL, dentro da mesma transação do `changeStatus`**, e um **worker em processo separado** (polling a cada 2 s) faz o envio HTTP assinado com **HMAC-SHA256** e secret por endpoint. Falhas seguem **5 tentativas com backoff exponencial** e depois vão para uma **DLQ** com replay manual por um administrador. A garantia de entrega é **at-least-once**, com `X-Event-Id` para deduplicação. Tudo reaproveita os padrões existentes do projeto, sem infraestrutura nova. Estimativa: **3 sprints**, incluindo a revisão de segurança ([09:47] Larissa).

## 2. Contexto e problema

Atlas Comercial, MaxDistribuição e Nova Cargo consultam `GET /orders` de tempos em tempos para saber se o status mudou, o que deixa a integração lenta e cara; a Atlas indicou que pode migrar para um concorrente se não houver entrega até o fim do trimestre ([09:00] Marcos). Para eles, "tempo real" é qualquer coisa abaixo de 10 segundos ([09:02] Marcos). O fluxo é só de saída: eles recebem, não enviam ([09:02] Marcos).

O sistema atual não tem eventos, filas, notificações externas nem webhooks. A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), numa transação que atualiza o pedido, grava `order_status_history` e movimenta o estoque. Essa transação não pode depender de um terceiro: um cliente lento ou offline travaria outras mudanças, e dar rollback por falha do cliente não é aceitável ([09:04] Bruno).

## 3. Proposta técnica

Visão geral, sem entrar no detalhe de implementação (ver [FDD](FDD.md)):

1. **Registro do evento (Outbox).** Dentro da transação do `changeStatus`, é inserida uma linha em `webhook_outbox` por webhook interessado, com o payload já renderizado. Falha na inserção desfaz a mudança de status. Webhooks que não ouvem aquele status não geram linha ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)).
2. **Worker separado.** Uma nova entry-point (`src/worker.ts`, `npm run worker`) consulta a outbox a cada 2 s e envia os eventos pendentes mais antigos. Single-worker, com ordenação por `order_id` apenas nessa condição ([ADR-005](adrs/ADR-005-worker-separado-em-polling.md)).
3. **Segurança.** Assinatura HMAC-SHA256 do corpo em `X-Signature`, secret única por endpoint, rotação pela API com grace period de 24 h, URL obrigatoriamente https ([ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)).
4. **Confiabilidade.** Timeout de 10 s por chamada; 5 tentativas com backoff de 1 min, 5 min, 30 min, 2 h e 12 h; esgotadas, o evento vai para a DLQ (`webhook_dead_letter`) e pode ser reprocessado por um `ADMIN` ([ADR-002](adrs/ADR-002-retry-com-backoff-e-dlq.md)).
5. **Contrato de entrega.** At-least-once; cada evento tem um UUID em `X-Event-Id` e o cliente deduplica ([ADR-004](adrs/ADR-004-entrega-at-least-once-com-x-event-id.md)). Payload enxuto, sem `items` ([ADR-007](adrs/ADR-007-formato-do-payload-e-headers-de-envio.md)).
6. **Superfície de API.** Novo módulo `src/modules/webhooks` no mesmo padrão dos demais: CRUD de configuração, histórico de entregas por webhook e replay de DLQ restrito a `ADMIN` ([ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)).

## 4. Alternativas consideradas

| Alternativa | Por que foi descartada | Fonte |
| --- | --- | --- |
| **Chamada HTTP síncrona no `OrderService`** | A transação de status já é pesada; um cliente lento travaria outros pedidos e um cliente offline forçaria rollback. | [09:04] Bruno, [09:06] Diego |
| **Redis Streams ou fila similar** | Exigiria infraestrutura nova para um time pequeno ("overengineering"); o outbox no MySQL existente resolve. | [09:07] Larissa, [09:07] Diego |
| **Trigger no banco para acionar o worker** | O MySQL não tem listener nativo; a trigger só executa SQL e não avisa processo externo. Polling de 2 s atende o limite de 10 s. | [09:09] Diego |
| **Retry indefinido** | Deixaria evento pendurado para sempre se o cliente sumiu. | [09:15] Diego |
| **3 tentativas em vez de 5** | Cobriria cerca de 30 min e perderia o evento numa manutenção de 2 h. | [09:16] Diego |
| **Exactly-once** | Exigiria coordenação dos dois lados e muito mais complexidade; at-least-once com `event_id` resolve "99% dos casos". | [09:25] Diego |
| **Truncar payload acima do limite** | Evento desse tamanho indica problema; a decisão é errar e não enviar. | [09:23] Sofia |

## 5. Questões em aberto

Itens que a reunião deixou sem decisão, adiou ou que o documento precisa confirmar com os revisores:

1. **Rate limiting de saída por cliente.** Com 50 pedidos mudando em um minuto, o cliente recebe 50 chamadas. Ficou como "observar e decidir depois" ([09:39] Larissa). **Fora do escopo desta fase.**
2. **Aviso ao cliente por e-mail** quando o webhook falha repetidamente. Adiado para uma próxima fase, após medir o impacto ([09:37] Larissa).
3. **Escala para múltiplos workers.** A ordenação só vale por `order_id` com um único worker; particionar por `order_id` ou lock pessimista ficaram como problema futuro ([09:13] Diego).
4. **Arquivamento da outbox** (cerca de 30 dias para linhas entregues): citado como fora do escopo da feature ([09:08] Diego).
5. **"5 tentativas" inclui a tentativa inicial?** A progressão 1 min/5 min/30 min/2 h/12 h tem cinco intervalos e soma cerca de 14 h 36 min, o que sugere a tentativa inicial mais 5 retries ([09:17] Diego). O texto da reunião também cita uma janela de "12 ou 24 horas" ([09:15] Diego). **A confirmar com Diego.**
6. **Assinatura durante o grace period de 24 h.** A reunião define que a secret antiga segue válida, mas não diz se o envio manda uma ou duas assinaturas ([09:21] Sofia). **A confirmar com Sofia.**
7. **Armazenamento da secret em repouso.** O HMAC exige a secret recuperável; a proteção dela não foi discutida. Sofia pediu dois dias úteis de revisão para HMAC e geração de secret ([09:46] Sofia).
8. **Recuperação de eventos presos em `PROCESSING`.** Se o worker cair no meio de um envio, a reunião não definiu como o evento volta à fila (por exemplo, após um limite de tempo). O at-least-once tolera o reenvio, mas o critério precisa ser fechado na revisão ([FDD](FDD.md), seção 5.2).
9. **Eventos de criação de pedido.** O `create` registra `PENDING` em `order_status_history` com `fromStatus` nulo, mas a reunião só tratou o `changeStatus` ([09:40] Bruno). A proposta cobre apenas mudanças via `changeStatus`.

## 6. Impacto e riscos

**Impacto**
- Código: um novo módulo, uma nova entry-point (`src/worker.ts`), novas tabelas e uma alteração pontual em `OrderService.changeStatus`.
- Operação: passam a existir dois processos (API e worker) para implantar e monitorar.
- Clientes: precisam implementar verificação de assinatura e deduplicação por `X-Event-Id`; Marcos documenta no portal ([09:26] Marcos, [09:40] Marcos).

**Riscos**

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Inserção na outbox degradar a transação de `changeStatus` | Média | Alto | Inserção simples e indexada; teste ponta a ponta; função `publishWebhookEvent` isolada ([09:41] Bruno) |
| Falha de segurança em HMAC ou na geração/armazenamento da secret | Baixa | Alto | Revisão de Sofia (2 dias úteis) antes do deploy ([09:46] Sofia) |
| Perda de ordenação ao escalar o worker | Média | Médio | Limitação documentada; só single-worker nesta fase ([09:13] Larissa) |
| Cliente fora do ar por mais de cerca de 15 h | Baixa | Médio | DLQ persistida e replay manual por `ADMIN` ([09:18] Diego) |

## 7. Decisões relacionadas

- [ADR-001 — Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Retry com backoff e DLQ](adrs/ADR-002-retry-com-backoff-e-dlq.md)
- [ADR-003 — HMAC-SHA256 com secret por endpoint](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)
- [ADR-004 — At-least-once com `X-Event-Id`](adrs/ADR-004-entrega-at-least-once-com-x-event-id.md)
- [ADR-005 — Worker separado em polling](adrs/ADR-005-worker-separado-em-polling.md)
- [ADR-006 — Reuso dos padrões do projeto](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
- [ADR-007 — Payload, headers, timeout e limite](adrs/ADR-007-formato-do-payload-e-headers-de-envio.md)
