# FDD — Sistema de Webhooks de Notificação de Pedidos

> Documento de implementação. A proposta e as alternativas estão no [RFC](RFC.md); cada decisão está registrada nos [ADRs](adrs/README.md). Itens marcados com **(derivado)** são escolhas de desenho feitas neste FDD para viabilizar o que a reunião decidiu, sem terem sido ditas literalmente; devem ser confirmadas na revisão.

## 1. Contexto e motivação técnica

Clientes B2B precisam saber em menos de 10 s quando o status de um pedido muda ([09:02] Marcos). Hoje a única fonte é `GET /orders` ([09:00] Marcos). A mudança de status ocorre em `OrderService.changeStatus` numa transação Prisma que atualiza `orders`, insere em `order_status_history` e debita ou repõe `stock_quantity` (`src/modules/orders/order.service.ts`). O envio HTTP não pode ficar nessa transação ([09:04] Bruno); por isso o evento é gravado numa outbox na mesma transação e enviado por um worker à parte ([ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-005](adrs/ADR-005-worker-separado-em-polling.md)).

## 2. Objetivos técnicos

- Registrar o evento de forma atômica com a mudança de status: se o pedido mudou, o evento existe; se houve rollback, o evento some ([09:06] Diego).
- Entregar em menos de 10 s no caso saudável (polling de 2 s + envio) ([09:09] Diego).
- Garantia **at-least-once** com `X-Event-Id` ([ADR-004](adrs/ADR-004-entrega-at-least-once-com-x-event-id.md)).
- Autenticidade e integridade via HMAC-SHA256 com secret por endpoint ([ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)).
- Reuso máximo dos padrões do projeto ([ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)).

## 3. Escopo e exclusões

**Dentro:** tabelas de outbox, DLQ, configuração e histórico de entregas; função `publishWebhookEvent`; módulo `src/modules/webhooks` (CRUD, rotação de secret, deliveries, replay de DLQ); worker com retry e DLQ; extensão de `changeStatus`.

**Fora (decisões da reunião):**
- Aviso por e-mail ao cliente em caso de falha ([09:37] Larissa).
- Rate limiting de saída por cliente: observar e decidir depois ([09:39] Larissa).
- Dashboard visual para o cliente; só endpoints ([09:40] Larissa).
- Arquivamento de linhas entregues da outbox ([09:08] Diego).
- Múltiplos workers e ordenação global ([09:13] Larissa).
- Webhooks de entrada (clientes enviando para a plataforma) ([09:02] Marcos).

## 4. Modelo de dados

Novas tabelas em `prisma/schema.prisma`, seguindo o padrão atual (`@id @default(uuid()) @db.Char(36)`, `@@map` em snake_case, `createdAt`/`updatedAt`). Os campos abaixo vêm da reunião, exceto os marcados **(derivado)**.

| Tabela | Campos principais |
| --- | --- |
| `webhook_endpoints` | `id`, `customer_id` (FK `customers`), `url`, `secret`, `previous_secret` e `previous_secret_expires_at` **(derivado, suporte à rotação)**, `events` (lista de `OrderStatus` ouvidos), `active`, `created_at`, `updated_at` ([09:21] Bruno, [09:33] Marcos) |
| `webhook_outbox` | `id`, `event_id` (UUID único, vai em `X-Event-Id`), `webhook_id` **(derivado: uma linha por endpoint, pois `X-Webhook-Id` identifica o cadastro)**, `order_id`, `event_type`, `payload` (JSON snapshot), `status` (`PENDING`, `PROCESSING`, `FAILED`, `DELIVERED`), `attempt_count` e `next_attempt_at` **(derivados, necessários ao backoff)**, `created_at`. Índices em `status` e `created_at` ([09:08] Diego) |
| `webhook_deliveries` **(derivado)** | `id`, `outbox_id`, `webhook_id`, `attempt`, `success`, `status_code`, `response_body` (truncado), `duration_ms`, `error_code`, `created_at`. Sustenta o histórico "sucesso/falha, payload, response, tempo de resposta" ([09:34] Marcos) |
| `webhook_dead_letter` | `id`, `outbox_id`, `webhook_id`, `payload`, `failure_reason`, `failed_at`, `replayed_at` e `replayed_by_id` **(derivados, para auditoria do replay)** ([09:18] Diego, [09:36] Sofia) |

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox

Gatilho: `OrderService.changeStatus`, após `tx.orderStatusHistory.create` e antes do `findUnique` final.

1. A transação já validou existência, transição (`canTransition`) e estoque.
2. `publishWebhookEvent(tx, order, from, to)` busca os `webhook_endpoints` ativos do `order.customerId` cujo `events` contém `to` ([09:34] Bruno).
3. Para cada endpoint encontrado, insere uma linha em `webhook_outbox` com `status = PENDING`, `event_id = uuid()` e o payload renderizado com o estado atual do pedido ([09:52] Larissa).
4. Se nenhum endpoint ouve `to`, nada é inserido.
5. Qualquer exceção na inserção propaga e desfaz a transação inteira ([09:40] Bruno).

```ts
// src/modules/orders/order.service.ts, dentro de changeStatus (esboço)
await tx.order.update({ where: { id }, data: { status: to } });
await tx.orderStatusHistory.create({ /* ...como hoje... */ });
await publishWebhookEvent(tx, { ...order, status: to }, from, to); // novo
```

### 5.2 Processamento pelo worker

`src/worker.ts` cria seu próprio `PrismaClient` ([09:30] Bruno), registra `SIGINT`/`SIGTERM` como o `src/server.ts` e entra em loop:

1. A cada 2 s, seleciona um batch pequeno de eventos `PENDING` com `next_attempt_at <= now`, ordenados por `created_at` ([09:09] Diego). O tamanho do batch é configurável; valor inicial sugerido **(derivado)**: 10.
2. Marca cada evento como `PROCESSING`.
3. Valida o tamanho: payload acima de 64 KB não é enviado; vai para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE` e sem retry **(derivado: a reunião disse "erra", sem definir onde)** ([09:23] Sofia, [09:24] Larissa).
4. Monta os headers e a assinatura (seção 6.2) e faz `POST` na URL do endpoint com timeout de 10 s ([09:42] Diego).
5. Resposta 2xx: grava um registro em `webhook_deliveries` com `success = true` e marca o evento `DELIVERED`. Qualquer outro resultado (não 2xx, timeout, erro de rede) é falha: grava a entrega com `success = false` e segue para 5.3.

Eventos `PROCESSING` esquecidos por queda do worker: o at-least-once absorve o reenvio **(derivado)**; o critério de recuperação (por exemplo, voltar a `PENDING` após um limite de tempo) está nas Questões em Aberto.

### 5.3 Retry

Falha na tentativa `n`: se ainda restam tentativas, `attempt_count` aumenta, `status` volta a `PENDING` e `next_attempt_at = now + atraso[n]`, com a progressão abaixo ([09:17] Diego):

| Retry | Atraso até a tentativa |
| --- | --- |
| 1 | 1 minuto |
| 2 | 5 minutos |
| 3 | 30 minutos |
| 4 | 2 horas |
| 5 | 12 horas |

A soma é de 14 h 36 min, os "quase 15 horas" citados na reunião. A interpretação adotada **(derivada)** é: 1 tentativa inicial mais 5 retries com esses atrasos; confirmar com Diego (RFC, questão 5).

### 5.4 DLQ

Esgotados os retries, em uma transação:

1. `webhook_outbox.status = FAILED`.
2. Insere em `webhook_dead_letter` com o payload, `failure_reason` (código `WEBHOOK_*` mais detalhe) e `failed_at` ([09:18] Diego).
3. Registra log `webhook_moved_to_dlq`.

### 5.5 Replay manual

`POST /admin/webhooks/dead-letter/:id/replay` (role `ADMIN`):

1. Carrega o registro da DLQ; inexistente gera `WEBHOOK_DEAD_LETTER_NOT_FOUND`.
2. Recoloca o evento na outbox como `PENDING`, com `attempt_count = 0` e `next_attempt_at = now`. Mantém o mesmo `event_id` **(derivado)** para que o cliente possa deduplicar um evento que talvez já tenha tratado.
3. Preenche `replayed_at` e `replayed_by_id` e registra o log `webhook_dead_letter_replayed` com o id do usuário ([09:36] Sofia).

## 6. Contratos públicos

Prefixo base: `/api/v1` (conforme `src/app.ts`). Todos exigem `Authorization: Bearer <JWT>` (`authenticate`). O formato de erro é o do error middleware: `{ "error": { "code", "message", "details?" } }`. Os caminhos de rotação de secret e de deliveries seguem a reunião quando citados; o caminho de rotação é **(derivado)**, pois a reunião só falou em "endpoint pro cliente conseguir pedir nova secret" ([09:21] Sofia).

### 6.1 Endpoints

**1. `POST /api/v1/webhooks`: cadastra um webhook** ([09:31] Marcos). Qualquer role autenticada ([09:37] Sofia). O `customerId` vai no body, não no JWT ([09:32] Larissa).

```json
// request
{
  "customerId": "5b1f2a0e-7c9d-4e55-9a3c-1f0f6a2b8d11",
  "url": "https://api.atlascomercial.example/hooks/pedidos",
  "events": ["SHIPPED", "DELIVERED"]
}
```

```json
// 201 Created (a secret só aparece nesta resposta)
{
  "id": "c2f0a7de-3b1e-4a55-8f0a-6d9a41e0b7aa",
  "customerId": "5b1f2a0e-7c9d-4e55-9a3c-1f0f6a2b8d11",
  "url": "https://api.atlascomercial.example/hooks/pedidos",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "<gerada pela plataforma>",
  "createdAt": "2026-11-11T12:00:00.000Z"
}
```

Status: `201`; `400 VALIDATION_ERROR` / `WEBHOOK_INVALID_URL` (URL não https); `404` se o customer não existe; `401`.

**2. `GET /api/v1/webhooks?customerId=...`: lista os webhooks de um customer** ([09:33] Bruno). Paginação no formato de `src/shared/http/response.ts`.

```json
// 200 OK
{
  "data": [
    {
      "id": "c2f0a7de-3b1e-4a55-8f0a-6d9a41e0b7aa",
      "customerId": "5b1f2a0e-7c9d-4e55-9a3c-1f0f6a2b8d11",
      "url": "https://api.atlascomercial.example/hooks/pedidos",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

Status: `200`; `400 VALIDATION_ERROR`; `401`. A secret nunca é devolvida em listagens.

**3. `PATCH /api/v1/webhooks/:id`: edita url, eventos ou estado ativo** ([09:33] Bruno).

```json
// request
{ "events": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
```

```json
// 200 OK
{
  "id": "c2f0a7de-3b1e-4a55-8f0a-6d9a41e0b7aa",
  "url": "https://api.atlascomercial.example/hooks/pedidos",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true
}
```

Status: `200`; `400 VALIDATION_ERROR` / `WEBHOOK_INVALID_URL`; `404 WEBHOOK_NOT_FOUND`; `401`.

**4. `DELETE /api/v1/webhooks/:id`: remove o webhook** ([09:33] Bruno). Sem body. Resposta `204 No Content` (como `DELETE /orders/:id`). Status: `204`; `404 WEBHOOK_NOT_FOUND`; `401`.

**5. `POST /api/v1/webhooks/:id/secret/rotate`: rotaciona a secret** **(caminho derivado)** ([09:21] Sofia). Sem body.

```json
// 200 OK
{
  "id": "c2f0a7de-3b1e-4a55-8f0a-6d9a41e0b7aa",
  "secret": "<nova secret>",
  "previousSecretExpiresAt": "2026-11-12T12:00:00.000Z"
}
```

A secret anterior segue válida por 24 h a partir da rotação. Status: `200`; `404 WEBHOOK_NOT_FOUND`; `401`.

**6. `GET /api/v1/webhooks/:id/deliveries`: histórico das últimas 100 entregas** ([09:34] Marcos).

```json
// 200 OK
{
  "data": [
    {
      "eventId": "7d0c4c2a-91a0-4b0c-b0a1-2c6e3b2f5d10",
      "attempt": 1,
      "success": false,
      "statusCode": 503,
      "durationMs": 1240,
      "payload": { "event_type": "order.status_changed", "to_status": "SHIPPED" },
      "responseBody": "Service Unavailable",
      "createdAt": "2026-11-11T12:05:03.000Z"
    }
  ]
}
```

Status: `200`; `404 WEBHOOK_NOT_FOUND`; `401`.

**7. `POST /api/v1/admin/webhooks/dead-letter/:id/replay`: reprocessa um item da DLQ** ([09:18] Diego, [09:35] Diego). Exige `requireRole('ADMIN')` ([09:36] Larissa). Sem body.

```json
// 200 OK
{
  "deadLetterId": "e9a1b3c4-1d2e-4f50-9a8b-7c6d5e4f3a2b",
  "eventId": "7d0c4c2a-91a0-4b0c-b0a1-2c6e3b2f5d10",
  "status": "PENDING",
  "replayedById": "a1b2c3d4-0000-4000-8000-000000000001"
}
```

Status: `200`; `401`; `403 FORBIDDEN` (role diferente de `ADMIN`); `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`; `409 WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` **(derivado)**.

### 6.2 Contrato de saída (requisição ao cliente)

`POST <url do webhook>` com ([09:44] Diego, [09:44] Sofia, [09:45] Diego):

| Header | Valor |
| --- | --- |
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID do evento |
| `X-Signature` | HMAC-SHA256 do corpo bruto, com a secret do endpoint |
| `X-Timestamp` | Momento do envio |
| `X-Webhook-Id` | Id do `webhook_endpoints` |

```json
{
  "event_id": "7d0c4c2a-91a0-4b0c-b0a1-2c6e3b2f5d10",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-11T12:05:00.000Z",
  "order_id": "9a7c2d1e-4b5f-4c6d-8e7f-0a1b2c3d4e5f",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "5b1f2a0e-7c9d-4e55-9a3c-1f0f6a2b8d11",
  "total_cents": 125000
}
```

Sem `items` ([09:43] Diego). O nome do campo do timestamp do payload (`timestamp`) e o formato ISO 8601 vêm da reunião.

Assinatura (esboço com `node:crypto`):

```ts
import { createHmac } from 'node:crypto';

const signature = createHmac('sha256', endpoint.secret).update(rawBody).digest('hex');
// header: X-Signature: <signature>
```

O formato do valor de `X-Signature` (hex, prefixo `sha256=`) **não foi definido**; confirmar com Sofia. O comportamento durante o grace period está nas Questões em Aberto do RFC.

## 7. Matriz de erros

Novas classes em `src/shared/errors/`, derivadas de `AppError` com códigos `WEBHOOK_*` ([09:28] Bruno, [09:29] Larissa).

| Código | HTTP | Onde ocorre | Origem |
| --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | CRUD, rotação e deliveries com id inexistente | [09:28] Bruno |
| `WEBHOOK_INVALID_URL` | 400 | URL cadastrada que não é https | [09:28] Bruno, [09:23] Sofia |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Replay de item inexistente | derivado |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | Replay repetido do mesmo item | derivado |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | n/a (worker) | Payload acima de 64 KB; vai para a DLQ sem retry | [09:23] Sofia, [09:24] Larissa |
| `WEBHOOK_DELIVERY_TIMEOUT` | n/a (worker) | Cliente sem resposta em 10 s; entra em retry | [09:42] Diego |
| `WEBHOOK_DELIVERY_FAILED` | n/a (worker) | Resposta não 2xx ou erro de rede; entra em retry | [09:14] Larissa |

O código `WEBHOOK_SECRET_REQUIRED` foi citado como exemplo ([09:28] Bruno), mas a reunião não definiu em que caso ele ocorre; não está na matriz até haver definição. Os códigos `n/a (worker)` não passam pelo error middleware: ficam em `webhook_deliveries.error_code` e em `webhook_dead_letter.failure_reason`. Erros de validação Zod continuam como `VALIDATION_ERROR` pelo middleware atual.

## 8. Estratégias de resiliência

| Aspecto | Estratégia |
| --- | --- |
| Timeout | 10 s por chamada HTTP; estouro vira falha e retry ([09:42] Diego) |
| Retry | 5 tentativas com backoff 1 min, 5 min, 30 min, 2 h, 12 h ([09:17] Larissa) |
| Fallback | DLQ em `webhook_dead_letter` e replay manual por `ADMIN` ([09:18] Diego); o e-mail foi descartado desta fase |
| Isolamento | Worker em processo separado: lentidão de cliente não afeta a API ([09:11] Diego) |
| Atomicidade | Evento gravado na transação de `changeStatus`; falha reverte tudo ([09:40] Bruno) |
| Duplicidade | At-least-once; cliente deduplica por `X-Event-Id` ([09:25] Diego) |
| Ordenação | Por `order_id` e só com um worker; sem garantia global ([09:13] Larissa) |
| Limite de tamanho | 64 KB; acima disso não envia ([09:24] Larissa) |

## 9. Observabilidade

A reunião só decidiu que o logger é o Pino existente e que nada novo entra ([09:29] Bruno). Métricas e tracing abaixo são **(derivados)** desse princípio e ficam sem ferramenta nova.

**Logs** (Pino de `src/shared/logger/index.ts`, nomes de evento em `snake_case` como `http_request` e `server_started`): `webhook_event_enqueued`, `webhook_delivery_attempt`, `webhook_delivery_succeeded`, `webhook_delivery_failed`, `webhook_moved_to_dlq`, `webhook_dead_letter_replayed` (com o id do usuário, para auditoria — [09:36] Sofia), `webhook_worker_started` e `webhook_worker_stopped`. Campos comuns: `eventId`, `webhookId`, `orderId`, `attempt`, `statusCode`, `durationMs`.

**Métricas** emitidas como campos dos logs acima ou calculáveis por consulta às tabelas (sem nova dependência):
- eventos `PENDING` na outbox e idade do mais antigo (sinal de que o worker parou);
- taxa de sucesso e de falha por tentativa;
- duração da chamada HTTP (`durationMs`);
- latência entre `created_at` do evento e o envio, a comparar com a meta de 10 s;
- total de itens na DLQ.

**Tracing:** não há infraestrutura de tracing no projeto. A correlação é feita por identificadores nos logs: `requestId` (já gerado em `src/middlewares/request-logger.middleware.ts`) gravado no evento ao enfileirar, e `eventId` + `webhookId` no worker.

## 10. Integração com o sistema existente

Todos os arquivos da tabela existem hoje no repositório. Os únicos caminhos que ainda **não existem** são os que a feature cria: `src/worker.ts` e a pasta `src/modules/webhooks/` (incluindo o arquivo do worker/processador).

| Arquivo | Como o módulo de webhooks se integra |
| --- | --- |
| `src/modules/orders/order.service.ts` | `changeStatus` passa a chamar `publishWebhookEvent(tx, order, from, to)` dentro de `prisma.$transaction`, depois de gravar o histórico. A assinatura do construtor de `OrderService` não muda ([09:41] Diego). |
| `src/modules/orders/order.status.ts` | Sem alteração. `canTransition` continua decidindo transições; os valores de `OrderStatus` alimentam o filtro `events` dos webhooks. |
| `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | Novas classes `Webhook*Error` estendem `AppError`, no estilo de `InsufficientStockError`, com códigos `WEBHOOK_*`; exportadas por `src/shared/errors/index.ts`. |
| `src/middlewares/error.middleware.ts` | Sem alteração: já serializa `AppError`, Zod e Prisma ([09:29] Bruno). |
| `src/middlewares/auth.middleware.ts` | `authenticate` em todas as rotas; `requireRole('ADMIN')` no replay de DLQ ([09:36] Larissa). |
| `src/middlewares/validate.middleware.ts` | Reuso do `validate({ body, query, params })` com schemas Zod do novo módulo, incluindo a regra de https. |
| `src/routes/index.ts` | `buildApiRouter` registra `/webhooks` e `/admin/webhooks` ao lado dos routers atuais; o tipo `Controllers` ganha `webhooks`. |
| `src/app.ts` | `buildControllers` instancia `WebhookRepository`, `WebhookService` e `WebhookController`, como os demais módulos. |
| `src/server.ts` | Sem alteração. É o modelo para a nova `src/worker.ts` (bootstrap, logs `*_started`, shutdown com `SIGINT`/`SIGTERM`). |
| `src/config/database.ts` | O worker chama `createPrismaClient()` e obtém uma instância própria, pois o client é por processo ([09:30] Bruno). |
| `src/config/env.ts` | Novas variáveis, se necessárias, entram no `envSchema` Zod (por exemplo intervalo de polling e tamanho do batch); valores padrão: 2 s e 10 **(derivados)**. |
| `src/shared/logger/index.ts` | Reuso do `logger` Pino no módulo e no worker. |
| `prisma/schema.prisma` | Novos modelos de `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries` e `webhook_dead_letter`, com nova migração; `Customer` ganha a relação com os endpoints. |
| `package.json` | Novo script `worker`, nos moldes de `dev`/`start` ([09:11] Larissa). |
| `tests/orders.test.ts` | Os testes existentes de `changeStatus` devem continuar passando; novos testes cobrem a gravação na outbox e o rollback quando a inserção falha. |

## 11. Dependências e compatibilidade

- Dependências existentes: MySQL via Prisma, Express, Zod, Pino, `jsonwebtoken`, `uuid` (já usado em `request-logger.middleware.ts`).
- Cliente HTTP do worker: o `fetch` nativo do Node ou biblioteca já presente; nenhuma dependência nova é exigida pela reunião.
- Sem breaking change na API existente: as respostas de `PATCH /orders/:id/status` permanecem iguais.
- Compatibilidade de operação: a API e o worker precisam apontar para a mesma `DATABASE_URL`.
- Migração Prisma: apenas adição de tabelas e relações.

## 12. Critérios de aceite técnicos

1. Uma mudança de status válida em `changeStatus` gera uma linha `PENDING` em `webhook_outbox` por webhook ativo que ouve o novo status.
2. Se a inserção na outbox falhar, o status do pedido, o histórico e o estoque permanecem como antes (rollback total).
3. Se nenhum webhook ouve o status, nada é inserido na outbox.
4. Com o worker em execução e o cliente saudável, o evento é entregue em menos de 10 s após o commit.
5. A requisição tem `X-Event-Id`, `X-Signature` (HMAC-SHA256 do corpo), `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`.
6. Resposta lenta (mais de 10 s) ou não 2xx agenda retry conforme a tabela de backoff.
7. Após esgotar as tentativas, o evento aparece em `webhook_dead_letter` e `webhook_outbox.status = FAILED`.
8. O replay exige `ADMIN`, recoloca o evento como `PENDING` e gera log com o id de quem executou.
9. URL `http://` é recusada com `WEBHOOK_INVALID_URL`.
10. Payload acima de 64 KB não é enviado.
11. A secret anterior aceita por 24 h após a rotação.
12. Reiniciar a API não interrompe o worker.

## 13. Riscos e mitigação

| Risco | Mitigação |
| --- | --- |
| Escrita extra na transação de `changeStatus` aumenta a latência | Inserção simples e indexada; teste ponta a ponta ([09:40] Bruno) |
| Worker parado sem ninguém perceber | Log e métrica de idade do evento mais antigo pendente |
| Duplicidade de eventos | At-least-once por desenho; contrato documentado por Marcos ([09:26] Marcos) |
| Falha de segurança na assinatura ou na secret | Revisão de Sofia, com 2 dias úteis antes do deploy ([09:46] Sofia) |
| Perda de ordenação em caso de múltiplos workers | Limitação documentada; só single-worker nesta fase ([09:13] Larissa) |
| Crescimento da tabela de outbox | Arquivamento (30 dias) tratado como trabalho futuro ([09:08] Diego) |
