# Tracker de Rastreabilidade

Cada linha liga um item dos documentos à sua origem na transcrição (`TRANSCRICAO.md`) ou no código. Itens marcados como **(derivado)** nos documentos são escolhas de desenho do FDD sem fala literal; aparecem aqui com a fala ou o arquivo que os motivou, para ficarem visíveis na revisão.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes B2B pedem notificação em tempo real e hoje fazem polling em GET /orders | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Restrição | Atlas pode migrar para o concorrente se não houver entrega até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Escopo | Feature é só de saída (outbound); cliente não envia para a plataforma | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-01 | docs/PRD.md | Métrica | "Tempo real" significa menos de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Métrica | Meta de zero evento perdido por registro atômico com a mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-03 | docs/PRD.md | Métrica | Janela de retentativa de cerca de 15 h em 5 tentativas | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-04 | docs/PRD.md | Restrição | Prazo de fim de novembro, estimado em 3 sprints | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-05 | docs/PRD.md | Restrição | Três sprints incluindo a revisão de segurança | TRANSCRICAO | [09:47] Larissa |
| PRD-USR-01 | docs/PRD.md | Público | Clientes são representados por usuários autenticados com JWT do sistema | TRANSCRICAO | [09:32] Marcos |
| PRD-USR-02 | docs/PRD.md | Cenário | Cliente escolhe ouvir apenas SHIPPED e DELIVERED | TRANSCRICAO | [09:33] Marcos |
| PRD-USR-03 | docs/PRD.md | Cenário | Cliente indisponível por manutenção de 2 h precisa de retentativas | TRANSCRICAO | [09:16] Diego |
| PRD-OOS-01 | docs/PRD.md | Fora de escopo | Aviso por e-mail ao cliente fica para próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de escopo | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de escopo | Dashboard visual é projeto separado do frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de escopo | Arquivamento de eventos entregues (cerca de 30 dias) fora desta feature | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-05 | docs/PRD.md | Fora de escopo | Ordenação global e múltiplos workers ficam para o futuro | TRANSCRICAO | [09:13] Larissa |
| PRD-OOS-06 | docs/PRD.md | Fora de escopo | Exactly-once descartado; garantia é at-least-once | TRANSCRICAO | [09:25] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook com secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Lista de status que o webhook quer receber | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | PATCH para editar e DELETE para remover webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | GET para listar os webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de eventos na inserção da outbox; sem interessados, não insere | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Envio de JSON order.status_changed assinado com HMAC-SHA256 | TRANSCRICAO | [09:43] Diego |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | 5 tentativas com backoff e DLQ ao esgotar | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas com sucesso/falha, payload, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Replay manual de DLQ por ADMIN com log de quem executou | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Rotação de secret com a anterior válida por 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Headers X-Event-Id, X-Signature, X-Timestamp, Content-Type | TRANSCRICAO | [09:44] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Cadastro só aceita URL https (validação Zod) | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Polling de 2 s atende o limite de 10 s | TRANSCRICAO | [09:09] Diego |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Evento gravado na mesma transação do changeStatus | TRANSCRICAO | [09:40] Bruno |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | At-least-once com deduplicação pelo cliente | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 s por chamada | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Payload máximo de 64 KB, sem envio acima disso | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Secret única por endpoint | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Reuso de AppError, Pino, error middleware, módulos e códigos WEBHOOK_ | TRANSCRICAO | [09:30] Larissa |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Ordenação só por order_id com worker único | TRANSCRICAO | [09:13] Larissa |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança de Sofia com 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Marcos documenta o contrato no portal de desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| PRD-RISK-01 | docs/PRD.md | Risco | Vazamento de secret por cliente já ocorreu antes | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-02 | docs/PRD.md | Risco | Cliente não deduplicar eventos repetidos; mitigar com documentação | TRANSCRICAO | [09:26] Marcos |
| PRD-RISK-03 | docs/PRD.md | Risco | Rajada de chamadas a um cliente sem rate limiting | TRANSCRICAO | [09:38] Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Cliente fora do ar por mais de cerca de 15 h exige replay manual | TRANSCRICAO | [09:17] Marcos |
| PRD-TEST-01 | docs/PRD.md | Testes | Testes ponta a ponta incluídos na estimativa | TRANSCRICAO | [09:46] Larissa |
| PRD-TEST-02 | docs/PRD.md | Testes | Projeto já usa Vitest em tests/ | CODIGO | vitest.config.ts |
| RFC-CTX-01 | docs/RFC.md | Contexto | Transação de changeStatus já atualiza orders, history e estoque | CODIGO | src/modules/orders/order.service.ts |
| RFC-CTX-02 | docs/RFC.md | Restrição | HTTP síncrono na transação travaria outros pedidos e forçaria rollback | TRANSCRICAO | [09:04] Bruno |
| RFC-CTX-03 | docs/RFC.md | Contexto | Revisores são os participantes da reunião; Larissa abre o documento de design | TRANSCRICAO | [09:50] Larissa |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Chamada HTTP síncrona no service de orders | TRANSCRICAO | [09:06] Diego |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams ou fila similar: infraestrutura nova, overengineering | TRANSCRICAO | [09:07] Larissa |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger no banco: MySQL não tem listener nativo | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Retry indefinido deixa evento pendurado | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | 3 tentativas cobririam só cerca de 30 min | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | Exactly-once exige coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-07 | docs/RFC.md | Alternativa descartada | Truncar payload acima do limite; decisão é errar | TRANSCRICAO | [09:23] Sofia |
| RFC-OPEN-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| RFC-OPEN-02 | docs/RFC.md | Questão em aberto | E-mail de aviso adiado para próxima fase | TRANSCRICAO | [09:37] Larissa |
| RFC-OPEN-03 | docs/RFC.md | Questão em aberto | Escala para múltiplos workers (particionar por order_id ou lock pessimista) | TRANSCRICAO | [09:13] Diego |
| RFC-OPEN-04 | docs/RFC.md | Questão em aberto | Arquivamento da outbox após cerca de 30 dias | TRANSCRICAO | [09:08] Diego |
| RFC-OPEN-05 | docs/RFC.md | Questão em aberto | "5 tentativas" inclui ou não a tentativa inicial; progressão tem 5 intervalos | TRANSCRICAO | [09:17] Diego |
| RFC-OPEN-06 | docs/RFC.md | Questão em aberto | Assinatura durante o grace period de 24 h não definida | TRANSCRICAO | [09:21] Sofia |
| RFC-OPEN-07 | docs/RFC.md | Questão em aberto | Proteção da secret em repouso não discutida; revisão de Sofia | TRANSCRICAO | [09:46] Sofia |
| RFC-OPEN-08 | docs/RFC.md | Questão em aberto | Recuperação de eventos presos em PROCESSING não definida (derivado) | TRANSCRICAO | [09:08] Diego |
| RFC-OPEN-09 | docs/RFC.md | Questão em aberto | Eventos de criação de pedido não tratados; create grava PENDING no histórico | CODIGO | src/modules/orders/order.service.ts |
| RFC-RISK-01 | docs/RFC.md | Risco | Escrita extra na transação de changeStatus | TRANSCRICAO | [09:41] Bruno |
| RFC-RISK-02 | docs/RFC.md | Risco | Revisão de segurança de HMAC e geração de secret | TRANSCRICAO | [09:46] Sofia |
| RFC-RISK-03 | docs/RFC.md | Risco | Perda de ordenação ao escalar o worker | TRANSCRICAO | [09:13] Larissa |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox no MySQL no mesmo commit do changeStatus | TRANSCRICAO | [09:08] Larissa |
| ADR-001-SNAP | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Payload gravado renderizado (snapshot) na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-001-UUID | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Chaves em UUID, padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-001-FN | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Função publishWebhookEvent(tx, order, from, to) em vez de injetar repository | TRANSCRICAO | [09:41] Bruno |
| ADR-001-IDX | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Índices em status e created_at da outbox | TRANSCRICAO | [09:08] Diego |
| ADR-001-COD | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Modelos do schema usam UUID Char(36) como chave | CODIGO | prisma/schema.prisma |
| ADR-002 | docs/adrs/ADR-002-retry-com-backoff-e-dlq.md | Decisão | 5 tentativas, backoff 1m/5m/30m/2h/12h, DLQ | TRANSCRICAO | [09:17] Larissa |
| ADR-002-DLQ | docs/adrs/ADR-002-retry-com-backoff-e-dlq.md | Decisão | DLQ em tabela separada webhook_dead_letter | TRANSCRICAO | [09:18] Diego |
| ADR-002-REPLAY | docs/adrs/ADR-002-retry-com-backoff-e-dlq.md | Decisão | Replay manual via POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18] Diego |
| ADR-002-ADM | docs/adrs/ADR-002-retry-com-backoff-e-dlq.md | Decisão | Replay exige ADMIN e registra quem executou | TRANSCRICAO | [09:36] Sofia |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com grace 24 h | TRANSCRICAO | [09:22] Sofia |
| ADR-003-GLOBAL | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Trade-off | Secret global descartada: vazar uma vazaria tudo | TRANSCRICAO | [09:21] Sofia |
| ADR-003-TS | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Decisão | X-Timestamp permite detectar replay attack | TRANSCRICAO | [09:44] Diego |
| ADR-004 | docs/adrs/ADR-004-entrega-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id (UUID gerado na outbox) | TRANSCRICAO | [09:26] Larissa |
| ADR-004-WID | docs/adrs/ADR-004-entrega-at-least-once-com-x-event-id.md | Decisão | Header X-Webhook-Id identifica o cadastro | TRANSCRICAO | [09:44] Sofia |
| ADR-004-TRD | docs/adrs/ADR-004-entrega-at-least-once-com-x-event-id.md | Trade-off | Deduplicação passa a ser responsabilidade do cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-005 | docs/adrs/ADR-005-worker-separado-em-polling.md | Decisão | Worker em polling de 2 s | TRANSCRICAO | [09:10] Larissa |
| ADR-005-PROC | docs/adrs/ADR-005-worker-separado-em-polling.md | Decisão | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| ADR-005-ENT | docs/adrs/ADR-005-worker-separado-em-polling.md | Decisão | Nova entry-point src/worker.ts e script npm run worker | TRANSCRICAO | [09:11] Larissa |
| ADR-005-PRISMA | docs/adrs/ADR-005-worker-separado-em-polling.md | Decisão | PrismaClient separado no worker, mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno |
| ADR-005-ORD | docs/adrs/ADR-005-worker-separado-em-polling.md | Restrição | Ordenação por order_id apenas com single-worker | TRANSCRICAO | [09:13] Larissa |
| ADR-005-COD | docs/adrs/ADR-005-worker-separado-em-polling.md | Restrição | server.ts é a entry-point existente que serve de modelo ao worker | CODIGO | src/server.ts |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Reuso máximo dos padrões existentes | TRANSCRICAO | [09:30] Larissa |
| ADR-006-MOD | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Módulo src/modules/webhooks com controller, service, repository, routes e schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-ERR | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Erros derivados de AppError com prefixo WEBHOOK_ | TRANSCRICAO | [09:29] Larissa |
| ADR-006-COD1 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Restrição | AppError e subclasses com errorCode | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-COD2 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Restrição | Subclasses HTTP existentes (InsufficientStockError, InvalidStatusTransitionError) | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-COD3 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Restrição | requireRole e authenticate reutilizados | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-006-COD4 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Restrição | Error middleware trata AppError, Zod e Prisma | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-COD5 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Restrição | Logger Pino existente | CODIGO | src/shared/logger/index.ts |
| ADR-006-JWT | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Restrição | JWT é de usuário operador; customer_id vem no body ou path | TRANSCRICAO | [09:32] Larissa |
| ADR-007 | docs/adrs/ADR-007-formato-do-payload-e-headers-de-envio.md | Decisão | Campos do payload, sem items | TRANSCRICAO | [09:43] Diego |
| ADR-007-HDR | docs/adrs/ADR-007-formato-do-payload-e-headers-de-envio.md | Decisão | Headers X-Event-Id, X-Signature, X-Timestamp, Content-Type | TRANSCRICAO | [09:44] Diego |
| ADR-007-TMO | docs/adrs/ADR-007-formato-do-payload-e-headers-de-envio.md | Decisão | Timeout de 10 s | TRANSCRICAO | [09:42] Diego |
| ADR-007-SIZE | docs/adrs/ADR-007-formato-do-payload-e-headers-de-envio.md | Restrição | Limite de 64 KB, erro e não trunca | TRANSCRICAO | [09:24] Diego |
| FDD-CTX-01 | docs/FDD.md | Contexto | changeStatus numa transação Prisma com update, history e estoque | CODIGO | src/modules/orders/order.service.ts |
| FDD-OOS-01 | docs/FDD.md | Fora de escopo | E-mail, rate limiting, dashboard, arquivamento e multi-worker fora | TRANSCRICAO | [09:37] Larissa |
| FDD-DATA-01 | docs/FDD.md | Decisão | Tabela de configuração com url, secret, customer_id, estado ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-DATA-02 | docs/FDD.md | Decisão | Status da outbox: pendente, processando, falhou, entregue | TRANSCRICAO | [09:08] Diego |
| FDD-DATA-03 | docs/FDD.md | Decisão | DLQ guarda payload, motivo da falha e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-DATA-04 | docs/FDD.md | Decisão | Tabela de entregas sustenta o histórico (derivado) | TRANSCRICAO | [09:34] Marcos |
| FDD-DATA-05 | docs/FDD.md | Restrição | Padrão de modelos Prisma (uuid, @@map, timestamps) | CODIGO | prisma/schema.prisma |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Criação do evento dentro da transação de changeStatus | TRANSCRICAO | [09:40] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Worker lê pendentes mais antigos em batch pequeno a cada 2 s | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Timeout de 10 s vira falha e retry | TRANSCRICAO | [09:42] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | Retry com 5 tentativas e progressão 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-05 | docs/FDD.md | Fluxo | DLQ ao esgotar tentativas | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-06 | docs/FDD.md | Fluxo | Replay recoloca o evento na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-07 | docs/FDD.md | Fluxo | Payload acima de 64 KB não é enviado (destino na DLQ é derivado) | TRANSCRICAO | [09:23] Sofia |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /webhooks: url, eventos, secret devolvida | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /webhooks listando por customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | Endpoint de rotação de secret (caminho derivado) | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay com role ADMIN | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | customerId vai no body ou path, não no JWT | TRANSCRICAO | [09:32] Larissa |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | CRUD aceita qualquer role autenticada por ora | TRANSCRICAO | [09:37] Sofia |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | Prefixo /api/v1 e roteamento por módulo | CODIGO | src/app.ts |
| FDD-CONTRATO-11 | docs/FDD.md | Contrato | Formato de paginação reaproveitado | CODIGO | src/shared/http/response.ts |
| FDD-CONTRATO-12 | docs/FDD.md | Contrato | Formato de erro { error: { code, message, details } } | CODIGO | src/middlewares/error.middleware.ts |
| FDD-CONTRATO-13 | docs/FDD.md | Contrato | Payload de saída com event_id, event_type, timestamp, order_id, order_number, from/to_status, customer_id, total_cents | TRANSCRICAO | [09:43] Diego |
| FDD-ERR-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL como exemplos de código | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Erro | Prefixo WEBHOOK_ para todo o módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-03 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL para URL http (validação Zod) | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-04 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE para payload acima de 64 KB | TRANSCRICAO | [09:24] Larissa |
| FDD-ERR-05 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT para resposta acima de 10 s | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-06 | docs/FDD.md | Erro | Padrão de códigos de erro INSUFFICIENT_STOCK e INVALID_STATUS_TRANSITION | CODIGO | src/shared/errors/http-errors.ts |
| FDD-RES-01 | docs/FDD.md | Resiliência | Isolamento por processo separado | TRANSCRICAO | [09:11] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | Fallback por DLQ e replay; e-mail descartado | TRANSCRICAO | [09:37] Larissa |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logger Pino, nada novo; métricas e tracing derivados desse princípio | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Log de auditoria com o autor do replay | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | requestId gerado pelo request logger usado como correlação | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus chama publishWebhookEvent(tx, ...) | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Máquina de estados inalterada; OrderStatus alimenta o filtro | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Rotas registradas em buildApiRouter | CODIGO | src/routes/index.ts |
| FDD-INT-04 | docs/FDD.md | Integração | buildControllers instancia o novo módulo | CODIGO | src/app.ts |
| FDD-INT-05 | docs/FDD.md | Integração | worker.ts segue o bootstrap e o shutdown de server.ts | CODIGO | src/server.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Worker cria PrismaClient próprio com createPrismaClient | CODIGO | src/config/database.ts |
| FDD-INT-07 | docs/FDD.md | Integração | Novas variáveis no envSchema Zod (derivado) | CODIGO | src/config/env.ts |
| FDD-INT-08 | docs/FDD.md | Integração | validate() reaproveitado para os schemas do módulo | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Novo script worker no package.json | TRANSCRICAO | [09:11] Larissa |
| FDD-INT-10 | docs/FDD.md | Integração | Testes existentes de changeStatus devem continuar passando | CODIGO | tests/orders.test.ts |
| FDD-AC-01 | docs/FDD.md | Critério de aceite | Rollback total se a inserção na outbox falhar | TRANSCRICAO | [09:40] Bruno |
| FDD-AC-02 | docs/FDD.md | Critério de aceite | Entrega em menos de 10 s com cliente saudável | TRANSCRICAO | [09:02] Marcos |
| FDD-AC-03 | docs/FDD.md | Critério de aceite | Reiniciar a API não interrompe o worker | TRANSCRICAO | [09:11] Diego |
| FDD-RISK-01 | docs/FDD.md | Risco | Crescimento da tabela de outbox | TRANSCRICAO | [09:08] Diego |
| FDD-RISK-02 | docs/FDD.md | Risco | Revisão de segurança antes do deploy | TRANSCRICAO | [09:46] Sofia |
