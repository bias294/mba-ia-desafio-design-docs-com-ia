# ADR-002: Retry com backoff exponencial e DLQ em tabela separada

**Status:** Aceito
**Data:** 11-11-2026
**ADRs relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-004](ADR-004-entrega-at-least-once-com-x-event-id.md), [ADR-005](ADR-005-worker-separado-em-polling.md)

## Contexto e Problema

Clientes podem ficar indisponíveis, inclusive por manutenção planejada de duas horas ([09:16] Diego). A plataforma precisa decidir quantas vezes tentar reenviar um evento, com que intervalos e o que fazer quando as tentativas se esgotam, sem deixar eventos pendurados para sempre e sem perder evidência para suporte.

## Direcionadores da Decisão

- Cobrir indisponibilidades longas do cliente, de horas, não de minutos ([09:16] Diego).
- Não manter evento pendente indefinidamente quando o cliente sumiu ([09:15] Diego).
- Manter a leitura da fila principal limpa ([09:18] Diego).
- Preservar payload, motivo e horário da falha para debug e reprocessamento ([09:18] Diego).
- Exigir auditoria para qualquer reprocessamento manual ([09:36] Sofia).

## Alternativas Consideradas

1. Cinco tentativas com backoff exponencial e DLQ em tabela separada.
2. Retry indefinido com backoff.
3. Três tentativas, marcando a falha na própria outbox.

## Decisão

Opção escolhida: **cinco tentativas com backoff exponencial de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas**, porque cobre uma janela de quase 15 horas sem manter evento vivo para sempre ([09:17] Larissa, [09:17] Diego).

Esgotadas as tentativas, a falha é considerada permanente e o evento vai para uma **DLQ em tabela separada**, com o payload, o motivo da falha e o horário ([09:18] Diego). O reprocessamento é manual, por um endpoint administrativo que recoloca o evento na fila como pendente; ele exige a role de administrador e registra em log quem o executou ([09:36] Sofia, [09:36] Larissa).

Uma chamada que não responde em 10 segundos conta como falha e entra nesse fluxo ([09:42] Diego).

[PRECISA DE INFORMAÇÃO: "5 tentativas" inclui a tentativa inicial? Os cinco intervalos sugerem uma tentativa inicial mais cinco retries, e a reunião também cita uma janela de "12 ou 24 horas" ([09:15] Diego). Confirmar com Diego.]

## Prós e Contras das Alternativas

**Cinco tentativas com backoff e DLQ separada**
- Bom: cobre manutenções de duas horas e quedas mais longas.
- Bom: a fila principal fica enxuta e a DLQ serve de evidência.
- Bom: reprocessamento controlado e auditado.
- Ruim: acima de cerca de 15 horas de indisponibilidade, a recuperação depende de intervenção manual.

**Retry indefinido**
- Bom: nenhuma falha vira definitiva automaticamente.
- Ruim: evento fica pendurado para sempre se o cliente desapareceu ([09:15] Diego).

**Três tentativas, falha marcada na própria outbox**
- Bom: menos estados e menos tabelas.
- Ruim: cobriria só cerca de 30 minutos e perderia o evento numa manutenção de duas horas ([09:16] Diego).
- Ruim: mistura eventos falhos com a fila de trabalho.

## Consequências

O sistema tolera quedas longas de cliente sem retry infinito, e todo evento termina entregue ou na DLQ. O custo é que clientes fora do ar por mais de cerca de 15 horas dependem do reprocessamento manual; Marcos considerou o limite aceitável ([09:17] Marcos).

Não há aviso automático ao cliente: o e-mail foi explicitamente adiado para uma fase futura ([09:37] Larissa). Um evento em espera de retry pode chegar depois de um evento mais novo do mesmo pedido, o que se soma à limitação de ordenação do [ADR-005](ADR-005-worker-separado-em-polling.md).

## Referências

- `src/middlewares/auth.middleware.ts:49` (verificação de role usada no endpoint administrativo)
- `src/shared/logger/index.ts:13` (logger usado no registro de auditoria)
