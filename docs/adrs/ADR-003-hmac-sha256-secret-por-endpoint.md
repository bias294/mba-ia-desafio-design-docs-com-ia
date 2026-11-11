# ADR-003: Autenticação por HMAC-SHA256 com secret por endpoint e rotação

**Status:** Aceito
**Data:** 11-11-2026
**ADRs relacionados:** [ADR-004](ADR-004-entrega-at-least-once-com-x-event-id.md), [ADR-007](ADR-007-formato-do-payload-e-headers-de-envio.md)

## Contexto e Problema

Os eventos levam dados de pedidos para um endpoint fora da infraestrutura da empresa. O cliente precisa validar que a requisição veio da plataforma e que o conteúdo não foi adulterado no caminho ([09:19] Sofia). Já houve um cliente que vazou uma secret no log da própria aplicação ([09:22] Diego), então o desenho precisa limitar o estrago de um vazamento e permitir a troca da secret.

## Direcionadores da Decisão

- O cliente precisa provar a origem e a integridade de cada chamada ([09:19] Sofia).
- Usar um padrão de mercado, com bibliotecas disponíveis em qualquer stack ([09:20] Sofia).
- Isolar o impacto de um vazamento a um único endpoint ([09:21] Sofia).
- Permitir a troca de secret sem derrubar a integração do cliente ([09:21] Sofia).
- Permitir ao cliente detectar reenvio malicioso de uma chamada antiga ([09:44] Diego).

## Alternativas Consideradas

1. HMAC-SHA256 do corpo, com secret única por endpoint e rotação com período de convivência.
2. Uma secret global da plataforma.
3. Confiar apenas em TLS, sem assinatura.

## Decisão

Opção escolhida: **HMAC-SHA256 sobre o corpo da requisição, com secret única por endpoint de webhook e rotação com período de convivência de 24 horas** ([09:22] Sofia).

A assinatura vai em um header da requisição. A secret é gerada pela plataforma e devolvida ao cliente na criação do cadastro ([09:31] Marcos). O cliente pode pedir uma nova secret pela API; a anterior continua válida por 24 horas em paralelo e depois deixa de valer ([09:21] Sofia). O momento do envio também vai em um header, para quem quiser detectar replay ([09:44] Diego).

A exigência de URL https no cadastro, validada no schema de entrada, faz parte da proteção em trânsito, mas não é uma decisão arquitetural separada ([09:23] Sofia).

[PRECISA DE INFORMAÇÃO: durante as 24 horas de convivência, o envio assina com uma secret ou com as duas? A reunião não definiu. Confirmar com Sofia.]

[PRECISA DE INFORMAÇÃO: como a secret é protegida em repouso, já que o HMAC exige que a plataforma a recupere em claro? Não foi discutido; entra na revisão de segurança de Sofia ([09:46] Sofia).]

## Prós e Contras das Alternativas

**HMAC-SHA256 com secret por endpoint e rotação**
- Bom: um vazamento compromete apenas um endpoint.
- Bom: padrão de mercado, simples de verificar do lado do cliente.
- Bom: a rotação com convivência evita downtime na migração.
- Ruim: duas secrets válidas durante a rotação aumentam a complexidade de envio e verificação.

**Secret global**
- Bom: um único segredo para gerir.
- Ruim: vazar uma secret vaza tudo ([09:21] Sofia).

**Apenas TLS**
- Bom: sem gestão de segredos.
- Ruim: o cliente não consegue provar a origem nem a integridade do conteúdo recebido ([09:19] Sofia).

## Consequências

Cada cliente passa a poder verificar origem e integridade de forma independente, e um vazamento é contido e remediável pela rotação. Em troca, a plataforma precisa guardar e proteger as secrets, gerar secrets com qualidade criptográfica e manter a lógica de convivência de duas secrets por 24 horas.

Sofia reservou dois dias úteis de revisão do código de HMAC e de geração de secret antes do deploy ([09:46] Sofia).

## Referências

- `src/middlewares/auth.middleware.ts:49` (autorização por role, usada nas rotas de configuração e de administração)
- `src/middlewares/validate.middleware.ts:11` (validação de entrada com schemas, onde a regra de https é aplicada)
