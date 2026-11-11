# PRD — Sistema de Webhooks de Notificação de Pedidos

> Documento de produto: responde *por que* e *o quê*. A abordagem técnica está no [RFC](RFC.md), as decisões nos [ADRs](adrs/README.md) e a implementação no [FDD](FDD.md).

## 1. Resumo e contexto da feature

Notificar sistemas de clientes B2B, por webhook, sempre que o status de um pedido deles mudar no Order Management System (OMS). A feature é só de saída: a plataforma envia, o cliente recebe ([09:02] Marcos). O OMS atual não tem notificação externa de nenhum tipo.

## 2. Problema e motivação

Atlas Comercial, MaxDistribuição e Nova Cargo pediram formalmente para serem avisados em tempo real. Hoje eles consultam `GET /orders` periodicamente, o que deixa a integração lenta e cara para eles. A Atlas indicou que pode migrar para um concorrente se a entrega não ocorrer até o fim do trimestre, e quer a feature para o fim de novembro ([09:00] Marcos, [09:45] Marcos).

## 3. Público-alvo e cenários de uso

**Público:** clientes B2B integrados via API, representados na plataforma por usuários autenticados com JWT ([09:32] Marcos). Internamente: administradores (role `ADMIN`) que reprocessam entregas falhas ([09:36] Sofia).

**Cenários**
1. A Atlas cadastra um endpoint https e escolhe ouvir apenas `SHIPPED` e `DELIVERED` ([09:33] Marcos).
2. Um pedido muda de status e o cliente recebe a notificação em poucos segundos, sem consultar `GET /orders`.
3. O sistema do cliente ficou fora do ar durante uma manutenção; a plataforma retenta automaticamente e o cliente recebe o evento quando voltar ([09:16] Diego).
4. O cliente confere o histórico das últimas 100 entregas para diagnosticar um problema ([09:34] Marcos).
5. O cliente suspeita de vazamento da secret e a rotaciona, tendo 24 h para migrar seus sistemas ([09:21] Sofia).
6. Um administrador reprocessa um evento que esgotou as tentativas ([09:18] Diego).

## 4. Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta | Fonte |
| --- | --- | --- | --- |
| Notificação em "tempo real" | Tempo entre a mudança de status e a chamada ao cliente, com o cliente saudável | Menos de 10 segundos (polling de 2 s no pior caso) | [09:02] Marcos, [09:10] Larissa |
| Não perder eventos | Eventos de mudança de status confirmados sem registro correspondente na outbox | Zero (registro atômico com a mudança de status) | [09:40] Bruno |
| Resiliência a indisponibilidade do cliente | Janela de retentativa automática | Cerca de 15 horas, em 5 tentativas | [09:17] Diego |
| Entrega no prazo | Feature em produção | Fim de novembro, em 3 sprints | [09:45] Marcos, [09:47] Larissa |
| Reduzir polling do cliente | Consultas a `GET /orders` pelos 3 clientes solicitantes | Redução esperada; **sem meta numérica definida na reunião** | [09:00] Marcos |

## 5. Escopo

### Incluído
- Cadastro, edição, listagem e remoção de webhooks por cliente.
- Seleção dos status que cada webhook quer receber.
- Entrega assinada com retry, DLQ e replay manual.
- Histórico de entregas por webhook.
- Rotação de secret com período de convivência de 24 h.

### Fora de escopo (descartado ou adiado na reunião)
- **Aviso por e-mail** ao cliente quando o webhook falha: fora desta fase, talvez na próxima ([09:37] Larissa).
- **Rate limiting de saída:** observar e decidir depois ([09:39] Larissa).
- **Dashboard visual** para o cliente: projeto separado do time de frontend ([09:40] Larissa).
- **Arquivamento** de eventos entregues (cerca de 30 dias) ([09:08] Diego).
- **Ordenação global** e múltiplos workers em paralelo ([09:13] Larissa).
- **Webhooks de entrada** (cliente enviando eventos para a plataforma) ([09:02] Marcos).
- **Exactly-once:** a garantia é at-least-once ([09:25] Diego).

## 6. Requisitos funcionais

| ID | Requisito | Fonte |
| --- | --- | --- |
| RF-01 | O cliente cadastra um webhook informando a URL; a secret é gerada pela plataforma e devolvida na criação | [09:31] Marcos |
| RF-02 | O cliente escolhe a lista de status que quer receber por webhook | [09:31] Marcos, [09:33] Marcos |
| RF-03 | O cliente edita (PATCH) e remove (DELETE) um webhook | [09:33] Bruno |
| RF-04 | O cliente lista os webhooks de um customer | [09:33] Bruno |
| RF-05 | Cada mudança de status de pedido gera um evento para os webhooks que ouvem o novo status; sem interessados, nenhum evento é gerado | [09:34] Marcos, [09:34] Bruno |
| RF-06 | O evento é enviado por HTTP com payload JSON (`order.status_changed`) e assinatura HMAC-SHA256 | [09:43] Diego, [09:20] Sofia |
| RF-07 | Falhas de entrega são retentadas até 5 vezes com backoff e, ao esgotar, vão para a DLQ | [09:17] Larissa |
| RF-08 | O cliente consulta o histórico de entregas (últimas 100) com sucesso/falha, payload, resposta e tempo de resposta | [09:34] Marcos |
| RF-09 | Um `ADMIN` reprocessa manualmente um item da DLQ e a ação é registrada em log | [09:18] Diego, [09:36] Sofia |
| RF-10 | O cliente rotaciona a secret pela API; a anterior continua válida por 24 h | [09:21] Sofia |
| RF-11 | Cada envio traz `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type` | [09:44] Diego, [09:44] Sofia |
| RF-12 | O cadastro só aceita URLs https | [09:23] Sofia |

## 7. Requisitos não funcionais

| ID | Requisito | Fonte |
| --- | --- | --- |
| RNF-01 | Latência de notificação menor que 10 s; polling de 2 s | [09:02] Marcos, [09:09] Diego |
| RNF-02 | Consistência: o evento é gravado na mesma transação da mudança de status | [09:06] Diego, [09:40] Bruno |
| RNF-03 | Entrega at-least-once; deduplicação pelo cliente via `X-Event-Id` | [09:24] Diego |
| RNF-04 | Timeout de 10 s por chamada ao cliente | [09:42] Diego |
| RNF-05 | Payload de no máximo 64 KB; acima disso, não é enviado | [09:24] Larissa |
| RNF-06 | TLS obrigatório no endpoint do cliente | [09:23] Sofia |
| RNF-07 | Secret única por endpoint | [09:21] Sofia |
| RNF-08 | O envio roda em processo separado da API | [09:11] Diego |
| RNF-09 | Reuso dos padrões do projeto (`AppError`, Pino, error middleware, módulos, códigos `WEBHOOK_*`) | [09:30] Larissa |
| RNF-10 | Ordenação por pedido apenas com worker único; sem garantia global | [09:13] Larissa |

## 8. Decisões e trade-offs principais

| Decisão | Trade-off | ADR |
| --- | --- | --- |
| Outbox no MySQL em vez de síncrono ou Redis | Simplicidade e consistência, ao custo de uma escrita extra e latência de polling | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| 5 tentativas com backoff e DLQ | Cobre quedas longas; acima de cerca de 15 h exige replay manual | [ADR-002](adrs/ADR-002-retry-com-backoff-e-dlq.md) |
| HMAC-SHA256 com secret por endpoint e rotação | Segurança isolada por cliente, com mais complexidade na rotação | [ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md) |
| At-least-once com `X-Event-Id` | Evita coordenação complexa; o cliente precisa deduplicar | [ADR-004](adrs/ADR-004-entrega-at-least-once-com-x-event-id.md) |
| Worker separado com polling de 2 s | Isolamento e simplicidade; sem escala horizontal nesta fase | [ADR-005](adrs/ADR-005-worker-separado-em-polling.md) |
| Reuso dos padrões existentes | Consistência e velocidade, herdando limites atuais | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| Payload enxuto, sem `items` | Contrato leve; detalhes via `GET /orders/:id` | [ADR-007](adrs/ADR-007-formato-do-payload-e-headers-de-envio.md) |

## 9. Dependências

- Revisão de segurança de Sofia, com ao menos 2 dias úteis antes do deploy ([09:46] Sofia).
- Documentação no portal de desenvolvedor, a cargo de Marcos, incluindo o contrato at-least-once ([09:26] Marcos, [09:40] Marcos).
- Alteração no `OrderService.changeStatus` (time de Pedidos) e infraestrutura de execução do novo processo worker (time de Plataforma).
- Confirmação do prazo com a Atlas ([09:47] Marcos).

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Perda do cliente Atlas por atraso na entrega | Média | Alto | Estimativa de 3 sprints e prazo confirmado com o cliente ([09:47] Marcos) |
| Vazamento de secret por parte de um cliente | Média | Alto | Secret por endpoint e rotação com grace period; já ocorreu antes ([09:22] Diego) |
| Cliente fora do ar por mais de cerca de 15 horas | Baixa | Médio | DLQ persistida e replay manual por `ADMIN` ([09:18] Diego) |
| Cliente não deduplicar eventos repetidos | Média | Médio | Documentação destacada no portal ([09:26] Marcos) |
| Volume alto de chamadas a um único cliente | Baixa | Médio | Rate limiting adiado; monitorar e decidir depois ([09:39] Larissa) |
| Falha de segurança na implementação de HMAC e secrets | Baixa | Alto | Revisão dedicada de Sofia ([09:46] Sofia) |

## 11. Critérios de aceitação

1. Um cliente cadastra um webhook https e recebe a secret na criação; URLs `http` são recusadas.
2. Ao mudar o status de um pedido para um status que o webhook ouve, o cliente recebe a chamada em menos de 10 s, com `X-Event-Id` e assinatura válida.
3. Mudanças para status não escolhidos não geram notificação.
4. Com o cliente indisponível, a plataforma retenta 5 vezes e, esgotadas, o evento fica na DLQ.
5. Um `ADMIN` reprocessa um item da DLQ; um usuário `OPERATOR` recebe `403`.
6. O cliente vê as últimas 100 entregas de um webhook.
7. Após rotacionar a secret, a antiga é aceita por 24 h e depois deixa de valer.
8. Uma falha ao registrar o evento desfaz a mudança de status.
9. Os documentos ao cliente (portal) descrevem o contrato at-least-once.

## 12. Estratégia de testes e validação

- **Testes de integração:** `changeStatus` com gravação na outbox, incluindo rollback quando a inserção falha; filtro por status; CRUD de webhooks; autorização de replay (ADMIN x OPERATOR). O projeto já usa Vitest (`tests/`).
- **Testes do worker:** polling, envio assinado, timeout de 10 s, progressão de backoff e movimentação para a DLQ.
- **Testes de contrato de saída:** headers e assinatura verificáveis por um receptor de teste.
- **Teste ponta a ponta** de mudança de status até a entrega ([09:46] Larissa inclui testes ponta a ponta na estimativa).
- **Validação de segurança:** revisão de Sofia antes do deploy ([09:46] Sofia).
- **Validação com clientes:** Marcos confirma o fluxo com os clientes solicitantes ([09:47] Marcos).
