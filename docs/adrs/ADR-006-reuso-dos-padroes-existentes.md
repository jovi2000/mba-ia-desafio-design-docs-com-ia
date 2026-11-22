# ADR-006: Reuso dos padrões existentes do projeto no módulo de webhooks

**Status:** Aceito
**Data:** 2026-11-22
**ADRs relacionados:** ADR-001, ADR-005

## Contexto

A codebase tem padrões consolidados: cada domínio é um módulo em `src/modules` com controller, service, repository, routes e schemas; erros centralizados na hierarquia de AppError, cada um com código de erro próprio; error middleware que já trata AppError, Zod e Prisma; logger Pino em todo o projeto; validação de entrada com Zod [09:27] Bruno. A pergunta foi se o webhook seguiria esses padrões ou ganharia estrutura própria.

A proposta de Bruno foi seguir o padrão módulo por módulo: uma pasta `src/modules/webhooks` (a criar) com a estrutura completa, entrypoint de worker separado e a lógica de processamento dentro do módulo [09:27]-[09:28] Bruno. Larissa fechou a decisão como reuso máximo do que já existe: AppError, Pino, error middleware, padrão de módulos, schemas Zod e padrão de códigos de erro [09:30] Larissa.

## Direcionadores da Decisão

- Manter a uniformidade da codebase, sem ilhas de padrão próprio.
- Aproveitar infraestrutura compartilhada que já funciona em produção.
- Reduzir superfície de revisão de código e tempo de implementação.
- Erros do módulo tratados pelo mesmo fluxo centralizado.

## Alternativas Consideradas

1. **Reuso máximo dos padrões existentes**: módulo `src/modules/webhooks` igual aos demais, erros AppError com prefixo WEBHOOK_, Pino, error middleware e Zod.
2. **Estrutura própria para o módulo**: tratamento de erro e log próprios do módulo de webhooks.
3. **Lógica embutida no módulo orders**: notificação como parte do domínio de pedidos, sem módulo novo.

## Decisão

Opção 1. O webhook vira um módulo como os outros: `src/modules/webhooks` com controller, service, repository, routes e schemas [09:27] Bruno, e o worker fica em entrypoint separado (`src/worker.ts`, a criar) com a lógica de processamento em arquivo do próprio módulo, como `webhook.worker.ts` ou `webhook.processor.ts` [09:28] Bruno. Os erros seguem a hierarquia existente com códigos no padrão `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, todos com prefixo `WEBHOOK_` [09:28] Bruno, [09:29] Larissa. O logger é o Pino já usado no projeto, sem adição de nada novo [09:29] Bruno, e o middleware de erro centralizado trata os novos erros sem precisar de mudança, porque já cobre AppError, Zod e Prisma [09:29] Bruno.

## Prós e Contras das Opções

**Reuso máximo**

- Prós: error middleware, logger e validação já funcionam e já são cobertos por teste; revisão de código menor; quem já conhece o projeto não precisa aprender padrão novo.
- Contras: os erros do módulo ficam limitados ao que a hierarquia existente expressa; necessidades específicas de webhook (timeout, retry) exigem código próprio dentro do padrão.

**Estrutura própria**

- Prós: liberdade para desenhar fluxo de erro e log específico de entrega.
- Contras: cria ilha de padrão na codebase; duplica solução de problemas já resolvidos; rejeitada.

**Lógica no módulo orders**

- Prós: mudança concentrada num único módulo existente.
- Contras: mistura notificação com domínio de pedidos, ferindo a estrutura modular que a codebase já tem [09:27] Bruno; acopla os dois domínios; rejeitada.

## Consequências

Positivas: o módulo de webhooks nasce com a mesma forma dos demais em estrutura, erros, validação e log; o middleware centralizado e o logger já em produção absorvem o módulo sem mudança; os códigos `WEBHOOK_` deixam a origem do erro identificável na resposta da API, como já acontece com `INVALID_STATUS_TRANSITION` e `INSUFFICIENT_STOCK`.

Negativas: a extensão do `changeStatus` para gravar a outbox exige tocar o módulo de orders, controlada para não vazar lógica de webhook para dentro dele (a gravação acontece por função dedicada que recebe a transação corrente) [09:41] Bruno/Diego; decisões de erro que não couberem na hierarquia atual exigem estender o padrão em vez de criar atalho.

O trade-off assumido: disciplina de seguir o padrão estabelecido em troca de não manter duas formas de fazer a mesma coisa.

## Referências

- `src/modules/orders/` (padrão de módulo a replicar: controller, service, repository, routes, schemas)
- `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` (hierarquia a estender com o prefixo WEBHOOK_)
- `src/middlewares/error.middleware.ts` (tratamento centralizado que absorve os novos erros sem mudança)
- `src/shared/logger/index.ts` (logger Pino reutilizado, com redact já configurado)
- `src/middlewares/auth.middleware.ts` (authenticate e requireRole reaproveitados, inclusive no replay com role ADMIN)
