# ADR-006: Reuso máximo dos padrões existentes do projeto

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (decisão em [09:30] Larissa)
- **Decisores:** Larissa, Bruno, Diego

## Contexto

O projeto segue uma estrutura consistente: cada domínio é um módulo em `src/modules/<dominio>/` com controller, service, repository, routes e schemas ([09:27] Bruno). Há uma hierarquia de erros, middleware de erro centralizado, logger Pino e autorização por role já funcionando.

## Decisão

A feature de webhooks será um **módulo igual aos demais**, sem introduzir tecnologia nova:

| Padrão existente | Como será reaproveitado | Arquivo de referência |
| --- | --- | --- |
| Estrutura modular | Novo módulo `src/modules/webhooks` com controller, service, repository, routes e schemas ([09:27] Bruno) | `src/modules/orders/` |
| Hierarquia de erros | Novas classes derivadas de `AppError`, com códigos prefixados por `WEBHOOK_` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) ([09:28] Bruno, [09:29] Larissa) | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` |
| Error middleware | Sem alteração: já trata `AppError`, Zod e Prisma ([09:29] Bruno) | `src/middlewares/error.middleware.ts` |
| Logger | Pino existente, nada novo ([09:29] Bruno) | `src/shared/logger/index.ts` |
| Validação | Schemas Zod, inclusive a regra de URL https ([09:23] Sofia) | `src/modules/orders/order.schemas.ts`, `src/middlewares/validate.middleware.ts` |
| Autorização | `requireRole('ADMIN')` no replay de DLQ ([09:36] Larissa) | `src/middlewares/auth.middleware.ts` |
| Rotas | Registro do novo router em `buildApiRouter` | `src/routes/index.ts` |
| Prisma | Mesmo banco e `DATABASE_URL`; `PrismaClient` próprio no worker ([09:30] Bruno) | `src/config/database.ts` |

## Alternativas Consideradas

1. **Serviço independente de notificações, com stack e padrões próprios.** Alternativa plausível, mas incompatível com a decisão de reuso máximo e com o tamanho do time ([09:07] Diego: "a gente é um time pequeno").
2. **Colocar a lógica de webhooks dentro do `OrderService`.** Descartada: a integração fica restrita à função `publishWebhookEvent(tx, ...)` chamada de `changeStatus` ([09:41] Diego).

## Consequências

**Positivas**
- Curva de aprendizado baixa; revisão de código simples, pois o padrão é conhecido.
- O error middleware já serializa os novos erros sem alteração.
- Logs e erros ficam consistentes com o restante da API.

**Negativas / trade-offs**
- O módulo herda as limitações atuais do projeto. Ex.: a autorização atual só distingue `ADMIN` e `OPERATOR`, e o JWT é de usuário operador, não de cliente ([09:32] Bruno). Por isso o `customer_id` vem no body ou no path, não do JWT ([09:32] Larissa).
- Endurecer o controle de acesso do CRUD de webhooks ficou para depois ([09:37] Sofia).
