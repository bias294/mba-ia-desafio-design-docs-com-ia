# ADR-005: Worker em processo separado com polling de 2 segundos

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (decisão em [09:10] Larissa)
- **Decisores:** Diego, Larissa, Bruno, Marcos

## Contexto

O requisito de "tempo real" dos clientes é qualquer coisa abaixo de 10 segundos ([09:02] Marcos). O worker precisa ler a outbox ([ADR-001](ADR-001-outbox-no-mysql.md)) e disparar as chamadas HTTP.

## Decisão

- Rodar o worker em **polling em loop a cada 2 segundos**, buscando os eventos pendentes mais antigos em batch pequeno ([09:08] Diego, [09:09] Diego). A latência mínima no pior caso é de 2 segundos, aceita pelo time ([09:10] Larissa).
- Rodar como **processo separado da API**, para que reiniciar a API não derrube o worker ([09:11] Diego).
- Criar (arquivos novos, ainda inexistentes) uma entry-point **`src/worker.ts`**, ao lado de `src/server.ts`, com o script **`npm run worker`** ([09:11] Larissa). A lógica de processamento fica no módulo, em `src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts` ([09:28] Bruno).
- Mesmo banco e mesma `DATABASE_URL`, mas **uma instância própria de `PrismaClient`**, porque o client é por processo ([09:30] Bruno).
- **Single-worker** por enquanto: a ordenação é implícita por `created_at` da outbox, valendo **por `order_id` e apenas enquanto houver um único worker**. Isso é uma limitação conhecida, não garantia de ordenação global ([09:12] Diego, [09:13] Larissa).

## Alternativas Consideradas

1. **Trigger no banco para acionar o worker.** Descartada: o MySQL não tem listener nativo como o `NOTIFY/LISTEN` do Postgres e a trigger só executa SQL, sem notificar processo externo ([09:09] Diego).
2. **Worker dentro do processo da API.** Descartada: reiniciar a API perderia o worker ([09:11] Diego).
3. **Múltiplos workers em paralelo.** Adiada: perderia a ordenação; no futuro seria possível particionar por `order_id` ou usar lock pessimista ([09:12] Diego, [09:13] Diego).

## Consequências

**Positivas**
- Simples de operar e de testar; reaproveita a stack existente.
- Isolamento: falhas ou lentidão de clientes não afetam a API.
- A latência de 2 s mais o tempo de envio cabe com folga no limite de 10 s.

**Negativas / trade-offs**
- Uma consulta ao banco a cada 2 segundos, mesmo sem eventos.
- O worker é ponto único de processamento; escalar horizontalmente exige redesenho para preservar a ordenação.
- Nova unidade de deploy e de monitoramento (a API e o worker passam a ser dois processos).
- Os clientes não pediram ordenação global, só saber se cada pedido mudou ([09:14] Marcos).
