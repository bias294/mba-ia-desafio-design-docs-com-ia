# ADR-006: Reuso máximo dos padrões existentes do projeto

**Status:** Aceito
**Data:** 11-11-2026
**ADRs relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-005](ADR-005-worker-separado-em-polling.md)

## Contexto e Problema

O projeto segue uma estrutura consistente: cada domínio é um módulo com controller, service, repository, rotas e schemas ([09:27] Bruno). Há uma hierarquia de erros com códigos padronizados, um middleware de erro centralizado, um logger único e uma verificação de role já em uso. A feature de webhooks pode seguir esses padrões ou criar os seus próprios.

## Direcionadores da Decisão

- Time pequeno, com pouca margem para novas ferramentas ([09:07] Diego).
- Manter erros e logs consistentes com o restante da API ([09:29] Bruno).
- Reduzir a curva de aprendizado e o esforço de revisão.
- Aproveitar o tratamento de erros já centralizado, sem alterá-lo ([09:29] Bruno).
- Respeitar as limitações atuais de identidade: o token é de usuário operador, não de cliente ([09:32] Bruno).

## Alternativas Consideradas

1. Módulo de webhooks nos mesmos padrões do projeto.
2. Serviço de notificações independente, com padrões próprios.
3. Colocar a lógica de webhooks dentro do módulo de pedidos.

## Decisão

Opção escolhida: **um módulo de webhooks igual aos demais, com reuso máximo do que já existe** ([09:30] Larissa).

Na prática, o módulo segue a estrutura por domínio do projeto ([09:27] Bruno); os erros novos derivam da hierarquia de erros existente, com códigos de erro prefixados por `WEBHOOK_` ([09:28] Bruno, [09:29] Larissa); a validação usa os schemas de entrada já adotados; o middleware de erro centralizado trata os novos erros sem mudança ([09:29] Bruno); o logger é o já existente ([09:29] Bruno); e o endpoint administrativo reaproveita a verificação de role de administrador ([09:36] Larissa). O worker usa o mesmo banco com uma instância própria do cliente ([09:30] Bruno).

A integração com o módulo de pedidos fica limitada a uma função que recebe o cliente da transação, e não a um acoplamento do serviço de pedidos com o módulo de webhooks ([09:41] Diego).

## Prós e Contras das Alternativas

**Módulo nos padrões do projeto**
- Bom: baixa curva de aprendizado e revisão simples.
- Bom: erros e logs consistentes com o resto da API.
- Bom: nenhuma mudança no tratamento de erros existente.
- Ruim: herda as limitações atuais, como ter apenas as roles de administrador e operador.

**Serviço independente com padrões próprios**
- Bom: liberdade de tecnologia e de ciclo de deploy.
- Ruim: duplica infraestrutura e convenções para um time pequeno.
- Ruim: incompatível com a decisão de reuso máximo.

**Lógica dentro do módulo de pedidos**
- Bom: menos arquivos novos.
- Ruim: mistura dois domínios e engorda um serviço que já faz muito.

## Consequências

A feature entra no sistema como mais um módulo, com custo de adoção baixo e comportamento previsível para quem já conhece o projeto. Como consequência, o controle de acesso do cadastro de webhooks fica no nível atual: qualquer role autenticada pode configurar, e o endurecimento ficou para depois ([09:37] Sofia). Como o token identifica um usuário operador e não um cliente, o identificador do cliente é informado no corpo ou no caminho da requisição ([09:32] Larissa).

## Referências

- `src/modules/orders/order.service.ts:26` (estrutura de serviço do módulo de pedidos, modelo para o novo módulo)
- `src/shared/errors/http-errors.ts:55` (exemplo de erro de domínio derivado da hierarquia existente)
- `src/shared/errors/app-error.ts:3` (classe base de erros com código)
- `src/middlewares/error.middleware.ts:14` (tratamento centralizado de erros)
- `src/shared/logger/index.ts:13` (logger único do projeto)
