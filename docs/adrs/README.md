# Architectural Decision Records

Decisões arquiteturais da feature **Sistema de Webhooks de Notificação de Pedidos**, extraídas da reunião técnica (`TRANSCRICAO.md`) e ancoradas no código existente.

| ADR | Decisão |
| --- | --- |
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL, gravado na mesma transação do `changeStatus` |
| [ADR-002](ADR-002-retry-com-backoff-e-dlq.md) | Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada |
| [ADR-003](ADR-003-hmac-sha256-secret-por-endpoint.md) | HMAC-SHA256, secret por endpoint, rotação com grace period de 24 h |
| [ADR-004](ADR-004-entrega-at-least-once-com-x-event-id.md) | Entrega at-least-once com deduplicação por `X-Event-Id` |
| [ADR-005](ADR-005-worker-separado-em-polling.md) | Worker em processo separado, polling de 2 s, single-worker |
| [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md) | Reuso dos padrões existentes (módulo, `AppError`, Pino, `requireRole`) |
| [ADR-007](ADR-007-formato-do-payload-e-headers-de-envio.md) | Payload, headers, timeout de 10 s e limite de 64 KB |

Nomeação: `ADR-NNN-titulo-em-kebab-case.md`.
