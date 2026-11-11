# ADR-002: Retry com backoff exponencial e DLQ em tabela separada

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (decisão em [09:17] Larissa)
- **Decisores:** Larissa, Diego, Bruno, Marcos

## Contexto

Clientes podem ficar indisponíveis, inclusive por manutenção planejada de duas horas ([09:16] Diego). É preciso decidir quantas vezes tentar, com que intervalos, e o que fazer com o evento depois que as tentativas se esgotam.

## Decisão

- **Backoff exponencial** com **5 tentativas** e progressão de **1 min, 5 min, 30 min, 2 h, 12 h**. A reunião estima "quase 15 horas" entre a primeira falha e a última tentativa ([09:17] Diego).
- Esgotadas as tentativas, a falha é considerada permanente e o evento vai para a **DLQ**, em uma **tabela separada `webhook_dead_letter`**, com payload, motivo da falha e timestamp ([09:18] Diego).
- O reprocessamento é **manual**, via `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego). O endpoint exige role `ADMIN` e registra em log quem executou o replay ([09:36] Sofia, [09:36] Larissa).
- Uma chamada HTTP que não responde em **10 segundos** conta como falha e entra no fluxo de retry ([09:42] Diego).

## Alternativas Consideradas

1. **Retry indefinido com backoff.** Descartada: o evento fica pendurado para sempre se o cliente sumiu ([09:15] Diego).
2. **3 tentativas.** Descartada: cobriria só cerca de 30 minutos e mataria o evento numa indisponibilidade de 2 horas ([09:16] Diego).
3. **Marcar como `failed` na própria outbox, sem tabela separada.** Descartada em favor da tabela dedicada: leitura da outbox principal mais limpa e evidência preservada para debug e reprocessamento ([09:18] Diego).

## Consequências

**Positivas**
- Cobre indisponibilidades longas sem retry infinito.
- A outbox principal fica enxuta; a DLQ serve de evidência para suporte.
- Replay manual com auditoria (role `ADMIN` e log do autor).

**Negativas / trade-offs**
- Um cliente fora do ar por mais de cerca de 15 horas perde a entrega automática e depende de replay manual. Marcos considerou o limite aceitável ([09:17] Marcos).
- Não há aviso automático ao cliente sobre falhas: o e-mail foi explicitamente adiado ([09:37] Larissa).
- Uma entrega que espera na fila de retry pode chegar depois de eventos mais novos do mesmo pedido; ver a limitação de ordenação no [ADR-005](ADR-005-worker-separado-em-polling.md).
- **Ponto não esclarecido na reunião:** se "5 tentativas" inclui a tentativa inicial. Os cinco intervalos listados sugerem 1 tentativa inicial mais 5 retries (ver Questões em Aberto do [RFC](../RFC.md)).
