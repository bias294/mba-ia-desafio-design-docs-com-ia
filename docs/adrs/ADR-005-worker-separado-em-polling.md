# ADR-005: Worker em processo separado com polling de 2 segundos

**Status:** Aceito
**Data:** 11-11-2026
**ADRs relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-retry-com-backoff-e-dlq.md)

## Contexto e Problema

Para os clientes, "tempo real" significa qualquer coisa abaixo de 10 segundos ([09:02] Marcos). Os eventos gravados na outbox ([ADR-001](ADR-001-outbox-no-mysql.md)) precisam ser lidos e enviados por algum processo. É preciso decidir como esse processo descobre que há eventos novos e onde ele executa.

## Direcionadores da Decisão

- Atender o limite de 10 segundos de ponta a ponta ([09:02] Marcos).
- Reiniciar a API não pode perder o processamento de envios ([09:11] Diego).
- Não introduzir infraestrutura nova ([09:07] Diego).
- Reaproveitar a mesma stack e o mesmo banco ([09:11] Diego).
- Manter a ordem dos eventos de um mesmo pedido ([09:12] Larissa).

## Alternativas Consideradas

1. Worker em processo separado, em polling a cada 2 segundos.
2. Gatilho no banco de dados para avisar o worker.
3. Worker dentro do mesmo processo da API.

## Decisão

Opção escolhida: **worker em processo separado, em polling a cada 2 segundos**, porque atende o limite de 10 segundos "tranquilo" e mantém a arquitetura simples ([09:09] Diego, [09:10] Larissa).

A cada ciclo, o worker busca em lote pequeno os eventos pendentes mais antigos, envia e marca o resultado ([09:08] Diego, [09:09] Diego). Ele roda como processo próprio, com uma nova entrada ao lado da entrada atual do servidor e um script para iniciá-lo ([09:11] Larissa), e a lógica de processamento fica dentro do módulo de webhooks ([09:28] Bruno). Usa o mesmo banco e a mesma string de conexão, mas uma instância própria do cliente de banco, já que o cliente é por processo ([09:30] Bruno).

A execução é com **um único worker**. A ordenação decorre da ordem de criação na outbox e vale por pedido, apenas enquanto houver um worker; não há garantia de ordenação global. A reunião registrou isso como limitação conhecida ([09:12] Diego, [09:13] Larissa), e os clientes não pediram ordenação global ([09:14] Marcos).

## Prós e Contras das Alternativas

**Processo separado com polling de 2 segundos**
- Bom: isolamento total entre API e envio.
- Bom: latência máxima de 2 segundos na leitura, dentro do limite de 10.
- Bom: reaproveita a stack e o banco existentes.
- Ruim: consulta ao banco a cada 2 segundos, mesmo sem eventos.

**Gatilho no banco**
- Bom: seria mais reativo, em teoria.
- Ruim: o MySQL não tem mecanismo nativo de escuta como o NOTIFY/LISTEN do Postgres, e o gatilho só executa SQL, sem avisar processo externo ([09:09] Diego).

**Worker dentro da API**
- Bom: um processo a menos para implantar.
- Ruim: reiniciar a API perde o worker ([09:11] Diego).

## Consequências

A API fica independente do envio e a latência fica previsível. Em troca, a plataforma passa a ter dois processos para implantar e monitorar, e o worker é um ponto único de processamento: escalar para vários workers fica fora desta fase porque quebraria a ordenação por pedido ([09:13] Diego).

## Referências

- `src/server.ts:6` (entrada atual da API, modelo para a entrada do worker)
- `src/config/database.ts:4` (criação do cliente de banco, reutilizada pelo worker)
- `package.json:10` (scripts de execução, onde entra o script do worker)
