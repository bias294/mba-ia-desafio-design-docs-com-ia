# Da Reunião ao Documento: Design Docs do Sistema de Webhooks

> O enunciado original do desafio está no repositório base: https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia

## Sobre o desafio

A tarefa era transformar a transcrição de uma reunião técnica de cerca de 55 minutos (`TRANSCRICAO.md`) em um pacote de design docs para uma feature de um Order Management System: um **Sistema de Webhooks de Notificação de Pedidos**. A reunião tomou várias decisões, descartou algumas ideias e adiou outras, e nada foi registrado além da gravação.

O resultado é documental: PRD, RFC, FDD, 7 ADRs e um Tracker que liga cada item à sua origem na transcrição ou no código. O código da aplicação (`src/`, `prisma/`, `tests/`) não foi alterado; serviu só de contexto.

## Ferramentas de IA utilizadas

- **Claude Code (extensão do VS Code, modelo Claude Sonnet 5.5):** leu a transcrição e o código, extraiu decisões e requisitos, redigiu os documentos e rodou a verificação final de consistência.
- **`gh` (GitHub CLI):** fork do repositório base e clone local.

## Workflow adotado

1. **Fork e setup:** fork do repositório base e clone local.
2. **Contextualização:** leitura integral do README, da `TRANSCRICAO.md` e dos arquivos do código que a reunião menciona (`order.service.ts`, `order.status.ts`, `shared/errors`, `auth.middleware.ts`, `error.middleware.ts`, logger, rotas, `prisma/schema.prisma`).
3. **Separação do que entra e do que não entra:** antes de escrever, listei decisões fechadas, requisitos, itens descartados (e-mail, dashboard, exactly-once, Redis), adiados (rate limiting, arquivamento, multi-worker) e pontos ambíguos.
4. **Ordem de produção, como sugerida no enunciado:** ADRs, RFC, FDD, PRD, Tracker e, por último, este README.
5. **Convenção de rastreabilidade:** toda afirmação nos documentos cita `[hh:mm] Nome` ou um arquivo. O que não tem fala literal fica marcado como **(derivado)** no FDD, para a revisão decidir.
6. **Verificação:** um script conferiu se cada timestamp e falante citados existem na transcrição, se cada caminho de arquivo existe e se os links entre os documentos funcionam.

## Prompts customizados

**Prompt 1: extração dirigida, em vez de "gere um PRD":**

```text
Leia TRANSCRICAO.md inteira. Produza quatro listas, cada item com [hh:mm] Nome:
1. Decisões FECHADAS (alguém disse "decidido", "anotado" ou houve consenso);
2. Itens DESCARTADOS ou ADIADOS (e a razão dada);
3. Requisitos funcionais e não funcionais explícitos;
4. Pontos AMBÍGUOS ou sem decisão.
Não inclua nada que não tenha fala correspondente. Se uma regra for necessária
mas ninguém a disse, coloque na lista 4, não nas outras.
```

**Prompt 2: ancoragem no código para a seção de integração do FDD:**

```text
Para cada ponto de integração citado na reunião (changeStatus, classes de erro,
requireRole, error middleware, logger Pino, PrismaClient, src/server.ts), abra o
arquivo real e diga: caminho, símbolo exato, o que muda e o que NÃO muda.
Só use caminhos que existam no repositório; caminhos que a feature vai criar
devem ser marcados como "novo".
```

**Prompt 3: verificação anti-alucinação:**

```text
Para cada linha do TRACKER.md, confira se o timestamp e o nome existem em
TRANSCRICAO.md e se o caminho de código existe. Liste também todo trecho dos
documentos que cite fala ou arquivo inexistente.
```

## Iterações e ajustes

Foram **2 ciclos principais**: geração dos documentos e uma revisão com verificação. Pontos concretos:

- **Contagem de tentativas ambígua.** A reunião diz "5 tentativas" com cinco atrasos (1 min a 12 h, soma de cerca de 14 h 36 min). Isso sugere uma tentativa inicial mais 5 retries, e a fala sobre "12 ou 24 horas" não bate com isso. Em vez de escolher em silêncio, registrei a interpretação como derivada no FDD e abri a questão no RFC.
- **Requisitos sem fala de origem.** O primeiro rascunho do PRD citava "time de suporte" como público; ninguém falou isso na reunião. Removi. Pelo mesmo motivo `WEBHOOK_SECRET_REQUIRED` ficou fora da matriz de erros: foi citado como exemplo, mas sem dizer quando ocorre.
- **Observabilidade sem base na reunião.** A transcrição só fixa o Pino existente. Métricas e tracing do FDD foram escritos como derivados desse princípio (logs estruturados e correlação por `requestId`/`eventId`), sem inventar ferramenta nova.
- **Lacunas do próprio desenho.** Ao escrever o FDD, apareceu o caso de um evento preso em `PROCESSING` se o worker cair, e o comportamento do envio durante o grace period de 24 h (quantas assinaturas enviar). Ambos foram para as Questões em Aberto do RFC.
- **PRD refeito no modelo de entrevista.** O primeiro PRD tinha as seções do enunciado, mas não seguia o esqueleto do prompt de PRD do curso. Reescrevi no formato exato do modelo (requisitos com fluxo, erros e prioridade; riscos com contingência; decisões com trade-off). O que a reunião não definiu, como prioridades e uptime, ficou marcado como proposta ou hipótese. Depois padronizei os demais documentos com as mesmas regras de estilo (sem travessão, itens sem fonte sinalizados).
- **Caminhos de arquivo que não existem ainda.** A verificação acusou `src/worker.ts` e `src/modules/webhooks/` como inexistentes. Eles são criados pela feature; acrescentei a indicação explícita nos documentos para que não pareçam referências inválidas.

## Como navegar a entrega

Ordem sugerida de leitura:

1. [docs/PRD.md](docs/PRD.md): por que e o quê.
2. [docs/RFC.md](docs/RFC.md): proposta técnica, alternativas e questões em aberto.
3. [docs/adrs/](docs/adrs/README.md): decisões individuais (ADR-001 a ADR-007).
4. [docs/FDD.md](docs/FDD.md): como construir, incluindo contratos, erros e integração com o código.
5. [docs/TRACKER.md](docs/TRACKER.md): origem de cada item.

```
.
├── README.md
├── TRANSCRICAO.md          (não alterado)
├── docs/
│   ├── PRD.md
│   ├── RFC.md
│   ├── FDD.md
│   ├── TRACKER.md
│   └── adrs/
│       ├── README.md
│       ├── ADR-001-outbox-no-mysql.md
│       ├── ADR-002-retry-com-backoff-e-dlq.md
│       ├── ADR-003-hmac-sha256-secret-por-endpoint.md
│       ├── ADR-004-entrega-at-least-once-com-x-event-id.md
│       ├── ADR-005-worker-separado-em-polling.md
│       ├── ADR-006-reuso-dos-padroes-do-projeto.md
│       └── ADR-007-formato-do-payload-e-headers-de-envio.md
└── src/, prisma/, tests/   (não alterados)
```
