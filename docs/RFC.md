# RFC: Sistema de Webhooks de Notificação de Pedidos

Este RFC propõe a arquitetura da solução em nível de HLD (alto nível) e a submete à revisão. O detalhe de implementação está no [FDD](FDD.md) e o porquê de cada decisão está nos [ADRs](adrs/README.md). Itens marcados como **hipótese** ou **proposta** não foram ditos na reunião (`TRANSCRICAO.md`) e precisam de confirmação.

## Metadados

| Campo | Valor |
| --- | --- |
| Autor | Larissa (Tech Lead), responsável por abrir o documento de design ([09:50] Larissa) |
| Status | Em revisão |
| Data | 2026-11-11 |
| Versão | 1.0 |
| Revisores | Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Segurança), Marcos (PM) |
| Documentos relacionados | [PRD](PRD.md), [FDD](FDD.md), [ADRs](adrs/README.md), [Tracker](TRACKER.md) |

## 1. Resumo executivo (TL;DR)

Propomos notificar clientes B2B por **webhooks outbound** sempre que o status de um pedido mudar. O evento é gravado numa tabela **outbox no MySQL, dentro da mesma transação do `changeStatus`**, e um **worker em processo separado** (polling a cada 2 s) faz o envio HTTP assinado com **HMAC-SHA256** e secret por endpoint. Falhas seguem **5 tentativas com backoff exponencial** e depois vão para uma **DLQ** com replay manual por um administrador. A garantia de entrega é **at-least-once**, com `X-Event-Id` para deduplicação. Tudo reaproveita os padrões existentes do projeto, sem infraestrutura nova. Estimativa: **3 sprints**, incluindo a revisão de segurança ([09:47] Larissa).

## 2. Contexto e problema

**Objetivo técnico.** Emitir notificações de mudança de status de pedido para sistemas externos, sem acoplar a chamada HTTP à transação de pedidos e sem perder eventos.

**Problema de negócio (resumo).** Atlas Comercial, MaxDistribuição e Nova Cargo consultam `GET /orders` de tempos em tempos, o que deixa a integração lenta e cara; a Atlas indicou que pode migrar para um concorrente se não houver entrega até o fim do trimestre ([09:00] Marcos). "Tempo real" significa menos de 10 segundos ([09:02] Marcos). O fluxo é só de saída ([09:02] Marcos). A narrativa completa está no [PRD](PRD.md).

**Problemas técnicos do estado atual.** O OMS não tem eventos, filas, notificação externa nem webhooks. A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), numa transação que atualiza o pedido, grava `order_status_history` e movimenta o estoque. Essa transação não pode depender de um terceiro: um cliente lento ou offline travaria outras mudanças, e dar rollback por falha do cliente não é aceitável ([09:04] Bruno).

**Sistemas conectados.** Módulos de pedidos, clientes e autenticação do próprio OMS, o MySQL via Prisma e os endpoints https dos clientes.

## 3. Proposta técnica

### 3.1 Arquitetura geral

- **Topologia:** o OMS continua um monólito modular Node.js e TypeScript. A feature adiciona um módulo `src/modules/webhooks` e um **segundo processo**, o worker (`src/worker.ts`, `npm run worker`), na mesma base de código e no mesmo banco ([09:11] Larissa, [09:11] Diego).
- **Ambiente de implantação:** o mesmo do OMS atual. A reunião não definiu cloud ou on-premises (hipótese: sem mudança de ambiente).
- **Padrões:** Outbox transacional, polling, retry com backoff exponencial, DLQ, entrega at-least-once e módulos por domínio (controller, service, repository, routes, schemas).
- **Tecnologias:** MySQL e Prisma, Express, Zod, Pino e `jsonwebtoken`, todas já presentes no projeto. Nenhuma infraestrutura nova ([09:07] Diego).

### 3.2 Componentes e responsabilidades

| Componente | Responsabilidades | Dependências |
| --- | --- | --- |
| `OrderService.changeStatus` (alterado) | Valida a transição e, na mesma transação, chama `publishWebhookEvent` | Prisma (`tx`) |
| `publishWebhookEvent` | Seleciona os webhooks interessados no novo status e insere os eventos na outbox, já renderizados | Tabelas de webhook e outbox |
| Módulo `webhooks` (API) | CRUD de configuração, rotação de secret, histórico de entregas e replay de DLQ | `authenticate`, `requireRole`, `validate`, `AppError` |
| Worker | Lê eventos pendentes, assina, envia, registra a entrega, agenda retry e move para a DLQ | Prisma próprio, logger Pino, endpoints dos clientes |
| Tabelas MySQL | `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letter`; fonte de verdade de eventos e entregas | MySQL |

### 3.3 Fluxo de requisições e de dados

1. O operador chama `PATCH /orders/:id/status`.
2. Dentro de uma transação, o `changeStatus` atualiza o pedido, grava o histórico, ajusta o estoque e insere os eventos na outbox. Se qualquer passo falhar, tudo é desfeito ([09:40] Bruno).
3. A cada 2 s o worker lê os eventos pendentes mais antigos ([09:09] Diego).
4. O worker envia `POST` assinado ao endpoint do cliente, com timeout de 10 s ([09:42] Diego).
5. Resposta 2xx marca o evento como entregue. Falha agenda retry (1 min, 5 min, 30 min, 2 h, 12 h) e, esgotadas as tentativas, o evento vai para a DLQ ([09:17] Larissa).
6. Um `ADMIN` pode recolocar um item da DLQ na fila ([09:36] Larissa).

O fluxo de dados é: mudança de status, snapshot do payload na outbox, envio HTTP, registro em `webhook_deliveries` e, em caso de falha definitiva, cópia para `webhook_dead_letter`. O detalhe de cada passo está no [FDD](FDD.md), seção 4.

### 3.4 Modelo de dados (alto nível)

- **Entidades:** configuração de webhook, evento da outbox, entrega e item da DLQ.
- **Relações:** um customer tem vários webhooks; um webhook gera vários eventos; cada evento tem várias entregas e, no máximo, um item de DLQ.
- **Fonte de verdade:** o MySQL do OMS. O payload é um snapshot gravado na inserção ([09:52] Larissa); chaves em UUID ([09:51] Larissa).
- **Retenção:** arquivar eventos entregues após cerca de 30 dias fica fora desta feature ([09:08] Diego).

### 3.5 Interfaces públicas

| Nome | Tipo | Protocolo | Exposição | SLA ou limite |
| --- | --- | --- | --- | --- |
| API de configuração de webhooks (`/webhooks`) | API | REST JSON | Externa, JWT | Sem SLA definido na reunião |
| Replay de DLQ (`/admin/webhooks/dead-letter/:id/replay`) | API | REST JSON | Externa, role `ADMIN` | Sem SLA definido na reunião |
| Webhook de saída | HTTP POST | HTTPS e JSON | Externa, rumo ao cliente | Menos de 10 s até o envio; timeout de 10 s; payload até 64 KB |
| Outbox | Fila em tabela | SQL | Interna | Polling de 2 s |

Contratos, payloads e códigos `WEBHOOK_*` estão no [FDD](FDD.md), seção 5.

### 3.6 Escalabilidade e disponibilidade

- **Escala:** worker único nesta fase. A ordenação vale por `order_id` apenas com um worker; escalar exige particionar por `order_id` ou usar lock pessimista ([09:12] Diego, [09:13] Diego).
- **Resiliência:** a API não depende do envio; reiniciar a API não derruba o worker ([09:11] Diego). Eventos pendentes ficam persistidos na outbox.
- **Meta de disponibilidade:** não definida na reunião (a confirmar com Marcos e Diego).
- **Não faz parte desta fase:** cache e rate limiting de saída ([09:39] Larissa).

### 3.7 Segurança

- **Autenticação e autorização:** JWT existente em todas as rotas; replay de DLQ restrito a `ADMIN`, com log de quem executou; demais operações para qualquer role autenticada por ora ([09:36] Sofia, [09:37] Sofia).
- **Integridade e origem:** HMAC-SHA256 do corpo, secret única por endpoint, rotação com 24 h de convivência ([ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)).
- **Transporte:** URL do webhook obrigatoriamente https ([09:23] Sofia).
- **Em aberto:** gestão e criptografia da secret em repouso, e tratamento de PII, não foram discutidos (questão 7).

### 3.8 Observabilidade

Logs estruturados com o Pino existente ([09:29] Bruno). Métricas e tracing são **proposta**, sem ferramenta nova: contadores e durações derivados dos logs e das tabelas, e correlação por `requestId`, `eventId` e `webhookId`. Detalhes no [FDD](FDD.md), seção 7.

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
7. **Armazenamento da secret em repouso e PII.** O HMAC exige a secret recuperável; a proteção dela não foi discutida. Sofia pediu dois dias úteis de revisão para HMAC e geração de secret ([09:46] Sofia).
8. **Recuperação de eventos presos em `PROCESSING`.** Se o worker cair no meio de um envio, a reunião não definiu como o evento volta à fila (por exemplo, após um limite de tempo). O at-least-once tolera o reenvio, mas o critério precisa ser fechado na revisão ([FDD](FDD.md), seção 4, fluxos alternativos).
9. **Eventos de criação de pedido.** O `create` registra `PENDING` em `order_status_history` com `fromStatus` nulo, mas a reunião só tratou o `changeStatus` ([09:40] Bruno). A proposta cobre apenas mudanças via `changeStatus`.
10. **Meta de disponibilidade** do envio e da API de webhooks: não definida na reunião.

## 6. Impacto e riscos

**Impacto**
- Código: um novo módulo, uma nova entry-point (`src/worker.ts`), novas tabelas e uma alteração pontual em `OrderService.changeStatus`.
- Operação: passam a existir dois processos (API e worker) para implantar e monitorar.
- Clientes: precisam implementar verificação de assinatura e deduplicação por `X-Event-Id`; Marcos documenta no portal ([09:26] Marcos, [09:40] Marcos).

**Riscos**

Probabilidade, impacto e contingência são **proposta** deste documento; as mitigações vêm da reunião. Os riscos de produto (prazo, vazamento de secret, deduplicação) estão no [PRD](PRD.md).

| Risco | Probabilidade | Impacto | Mitigação | Plano de contingência (proposta) |
| --- | --- | --- | --- | --- |
| Inserção na outbox degradar a transação de `changeStatus` | Média | Alto | Inserção simples e indexada; teste ponta a ponta; função `publishWebhookEvent` isolada ([09:41] Bruno) | Medir a duração da transação e revisar índices da outbox antes do deploy |
| Falha de segurança em HMAC ou na geração/armazenamento da secret | Baixa | Alto | Revisão de Sofia (2 dias úteis) antes do deploy ([09:46] Sofia) | Adiar o deploy até corrigir os achados da revisão |
| Perda de ordenação ao escalar o worker | Média | Médio | Limitação documentada; só single-worker nesta fase ([09:13] Larissa) | Manter um único worker até haver desenho de particionamento por `order_id` ou lock pessimista ([09:13] Diego) |
| Cliente fora do ar por mais de cerca de 15 h | Baixa | Médio | DLQ persistida e replay manual por `ADMIN` ([09:18] Diego) | Avisar o cliente fora do sistema e reprocessar a DLQ quando ele voltar |

## 7. Decisões relacionadas

ADRs associados (todos com status Aceito):

- [ADR-001: Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002: Retry com backoff e DLQ](adrs/ADR-002-retry-com-backoff-e-dlq.md)
- [ADR-003: HMAC-SHA256 com secret por endpoint](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)
- [ADR-004: At-least-once com `X-Event-Id`](adrs/ADR-004-entrega-at-least-once-com-x-event-id.md)
- [ADR-005: Worker separado em polling](adrs/ADR-005-worker-separado-em-polling.md)
- [ADR-006: Reuso dos padrões do projeto](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
- [ADR-007: Payload, headers, timeout e limite](adrs/ADR-007-formato-do-payload-e-headers-de-envio.md)

**Decisões pendentes:** as questões 5 a 10 da seção 5, com critério de fechamento na revisão deste RFC.

**Próximos passos técnicos**
- Sessão de revisão do desenho com Bruno e Diego antes de codar ([09:50] Larissa).
- Fechar as questões em aberto e converter o resultado em ajustes no [FDD](FDD.md) e nos ADRs.
- Implementar nesta ordem estimada: outbox e DLQ (1 sprint), worker e retry (1 sprint), CRUD, deliveries, integração no `changeStatus` e testes ponta a ponta (1 sprint) ([09:46] Larissa).
- Revisão de segurança de Sofia, com 2 dias úteis, antes do deploy ([09:46] Sofia).
