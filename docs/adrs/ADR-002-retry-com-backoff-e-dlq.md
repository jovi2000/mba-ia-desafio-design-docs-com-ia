# ADR-002: Retry com backoff exponencial e DLQ persistida

**Status:** Aceito
**Data:** 2026-11-22
**ADRs relacionados:** ADR-001, ADR-005

## Contexto

Com o outbox em produção (ADR-001), a falha de entrega deixa de ser problema da transação de pedidos, mas passa a ser problema do worker: o endpoint do cliente pode estar offline no momento da entrega [09:14] Larissa. Sem política de retentativa, o evento falha uma vez e se perde; sem teto de tentativas, o evento de um cliente que sumiu fica pendurado para sempre na fila [09:15] Diego.

A reunião discutiu o número de tentativas e a progressão dos intervalos. Houve precedente real de cliente indisponível por duas horas em manutenção planejada [09:16] Diego, o que descartou políticas agressivas com poucas tentativas próximas. Discutiu-se também o destino do evento que esgota as tentativas: marcar como failed na própria outbox ou mover para uma tabela separada [09:17] Larissa.

## Direcionadores da Decisão

- Cobrir indisponibilidades transitórias do cliente, incluindo janelas de manutenção planejada de horas.
- Não deixar evento pendurado para sempre quando o cliente some definitivamente.
- Manter a tabela da outbox limpa para a leitura do worker.
- Preservar evidência do que falhou, para debug e reprocessamento manual.

## Alternativas Consideradas

1. **5 tentativas com backoff exponencial (1m, 5m, 30m, 2h, 12h) e DLQ em tabela separada**: esgotadas as tentativas, o evento é movido para `webhook_dead_letter`.
2. **Retry indefinido com backoff crescente**: tentar para sempre com intervalos cada vez maiores.
3. **3 tentativas com backoff agressivo**: desistir rápido e mover para a DLQ.

## Decisão

Opção 1. Cinco tentativas com progressão de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, totalizando quase 15 horas entre a primeira falha e a última tentativa [09:17] Diego. Cinco tentativas cobrem uma janela de 12 a 24 horas de indisponibilidade do cliente [09:15] Diego. Esgotadas as tentativas, o evento é persistido em tabela separada `webhook_dead_letter`, com o payload, o motivo da falha e o timestamp [09:18] Diego, mantendo a outbox principal limpa para a leitura do worker. O reprocessamento é manual, por endpoint administrativo que recoloca o evento na outbox como pendente [09:18] Diego.

## Prós e Contras das Opções

**5 tentativas com backoff e DLQ separada**

- Prós: cobre manutenções planejadas reais, como a indisponibilidade de duas horas já observada; a DLQ separada mantém a leitura da outbox simples; a DLQ serve de evidência para debug e reprocessamento.
- Contras: cliente indisponível por mais de ~15 horas só volta a receber via replay manual; o endpoint administrativo de replay exige controle de acesso próprio (role ADMIN).

**Retry indefinido**

- Prós: nenhum evento é descartado sem intervenção humana.
- Contras: evento de cliente sumido fica pendurado para sempre [09:15] Diego; rejeitado.

**3 tentativas**

- Prós: desistência rápida libera o worker.
- Contras: as três tentativas se esgotariam em cerca de 30 minutos, matando o evento durante uma manutenção planejada de duas horas [09:16] Diego; rejeitado.

## Consequências

Positivas: a política cobre o cenário real observado de indisponibilidade programada; a DLQ separada preserva o histórico de falhas sem poluir a tabela operacional; o replay manual dá controle explícito sobre o reprocessamento.

Negativas: eventos de clientes fora do ar por mais de ~15 horas dependem de intervenção manual via replay; a tabela de dead letter cresce sem política de limpeza definida nesta fase.

O trade-off assumido: aceitar uma janela máxima de aproximadamente 15 horas de retry automático em troca de uma fila previsível e sem eventos zumbis.

## Referências

- `prisma/schema.prisma` (local de modelagem das tabelas webhook_outbox e webhook_dead_letter)
- `src/modules/orders/order.service.ts` (transação que origina os eventos entregues com retry)
- `TRANSCRICAO.md` [09:14]-[09:18] (discussão e decisão)
