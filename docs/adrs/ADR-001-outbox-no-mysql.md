# ADR-001: Padrão Outbox no MySQL para eventos de mudança de status

**Status:** Aceito
**Data:** 2026-11-22
**ADRs relacionados:** ADR-002, ADR-005, ADR-007

## Contexto

A feature de webhooks precisa notificar clientes B2B quando o status de um pedido muda. Hoje a mudança de status é uma transação SQL que atualiza o pedido, insere o registro no histórico de status e ajusta o estoque (`src/modules/orders/order.service.ts`). O disparo da notificação não pode entrar nessa transação como chamada HTTP: um cliente lento travaria a mudança de status de outros pedidos, e um cliente fora do ar criaria a escolha impossível entre reverter a mudança de status ou perder o evento [09:04] Bruno.

O problema central: registrar o evento de notificação com atomicidade em relação à mudança de status (commit da mudança implica evento registrado; rollback implica evento descartado), sem acoplar a disponibilidade do cliente à operação de pedidos e sem introduzir infraestrutura de mensageria nova. O sistema opera MySQL via Prisma e o time é pequeno; subir um broker só para isso foi considerado desproporcional [09:07] Diego.

## Direcionadores da Decisão

- Atomicidade entre mudança de status e registro do evento, na mesma transação de banco.
- A chamada HTTP de notificação não pode estar no caminho crítico da mudança de status.
- Nenhuma infraestrutura nova: MySQL e Prisma existentes resolvem.
- O evento persistido deve servir de evidência para auditoria e reprocessamento.

## Alternativas Consideradas

1. **Outbox no MySQL**: inserir a linha do evento em tabela própria, dentro da mesma transação da mudança de status; um worker separado lê a tabela e entrega.
2. **Disparo síncrono no service de orders**: chamar o endpoint do cliente dentro do fluxo de mudança de status.
3. **Redis Streams (ou fila dedicada)**: publicar o evento em um broker externo no momento da mudança de status.

## Decisão

Opção 1, outbox no MySQL. Dentro da mesma transação SQL que atualiza o pedido e o histórico de status, o sistema insere uma linha na tabela `webhook_outbox` com o evento. Um worker separado lê essa tabela e dispara as chamadas HTTP. Se a transação principal commita, o evento foi registrado; se dá rollback, o evento some junto: não há estado inconsistente possível [09:06] Diego. A tabela terá índice pelo status do evento (pendente, processando, falhou, entregue) e por `created_at`, e o worker lê apenas os pendentes, em lotes pequenos [09:08] Diego.

## Prós e Contras das Opções

**Outbox no MySQL**

- Prós: atomicidade transacional sem infraestrutura nova; o worker pode cair e reiniciar sem perder eventos; a tabela serve de trilha para auditoria e retry.
- Contras: latência mínima de entrega imposta pelo polling; a tabela cresce com o volume e precisará de arquivamento, deixado fora do escopo desta fase [09:08] Diego; ordenação garantida apenas com worker único.

**Disparo síncrono**

- Prós: latência mínima e implementação simples, sem tabelas extras.
- Contras: cliente lento trava a mudança de status de outros pedidos; cliente fora do ar força rollback da mudança ou perda do evento [09:04] Bruno; rejeitado como fora de questão [09:06] Diego.

**Redis Streams**

- Prós: entrega reativa por push; infraestrutura de mensageria madura.
- Contras: exige subir e operar mais infraestrutura (Redis Cluster) para um time pequeno, tratado como overengineering [09:07] Diego; introduz segunda fonte de verdade em potencial se a publicação no broker falhar após o commit.

## Consequências

Positivas: a garantia de consistência fica no banco que já é a fonte de verdade; falha de entrega nunca bloqueia mudança de status; o evento fica persistido para retry, auditoria e reprocessamento.

Negativas: a entrega passa a ter latência mínima de 2 segundos no pior caso (intervalo de polling, ADR-005), aceita frente ao requisito de notificação abaixo de 10 segundos [09:10] Larissa; linhas entregues acumulam e o arquivamento após 30 dias ficou fora do escopo [09:08] Diego; a ordenação de eventos só é garantida por pedido enquanto houver um único worker [09:12]-[09:13] Diego/Larissa.

O trade-off assumido: abrir mão de latência mínima e de reatividade em troca de consistência transacional e zero infraestrutura nova.

## Referências

- `src/modules/orders/order.service.ts` (transação de mudança de status que passa a gravar a outbox)
- `src/modules/orders/order.status.ts` (máquina de estados cujas transações originam os eventos)
- `prisma/schema.prisma` (modelo transacional atual, onde a outbox será modelada)
- `TRANSCRICAO.md` [09:03]-[09:08] (discussão e decisão)
