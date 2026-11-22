# ADR-005: Worker em processo separado consumindo a outbox por polling

**Status:** Aceito
**Data:** 2026-11-22
**ADRs relacionados:** ADR-001, ADR-002, ADR-006

## Contexto

Definido o outbox no MySQL (ADR-001), faltava decidir onde roda o código de entrega e como ele toma ciência dos eventos pendentes. Rodar dentro da instância da API acopla a entrega ao ciclo de vida da API: se a API reinicia, o worker some junto [09:11] Diego. Sobre o modo de leitura, o MySQL não tem mecanismo nativo de notificação a processos externos, como o NOTIFY/LISTEN do PostgreSQL; triggers executam apenas SQL e não notificam ninguém [09:09] Diego.

O requisito de latência dá folga: os clientes consideram tempo real qualquer coisa abaixo de 10 segundos [09:02] Marcos, e um intervalo de polling de 2 segundos atende com margem [09:10] Larissa. A ordenação também entrou na discussão: com um único worker, os eventos saem na ordem de criação na outbox; com múltiplos workers em paralelo, a garantia se perde [09:12] Diego.

## Direcionadores da Decisão

- A entrega sobrevive a restarts e deploys da API.
- Mesmo banco e mesma stack, sem infraestrutura nova.
- Latência de entrega abaixo de 10 segundos, com folga.
- Ordenação previsível dos eventos por pedido, por ora.

## Alternativas Consideradas

1. **Processo separado com polling de 2 segundos**: entrypoint próprio no projeto, lendo os pendentes mais antigos a cada 2 segundos.
2. **Worker embutido no processo da API**: timer dentro da própria instância da API.
3. **Reatividade via trigger do banco**: a trigger do banco notificando o worker por mecanismo externo (arquivo, endpoint).

## Decisão

Opção 1. O worker roda como processo separado, não dentro da instância da API [09:11] Diego. O projeto ganha um entrypoint novo, no padrão do `src/server.ts` existente: um `src/worker.ts` (a criar) e um script `npm run worker` [09:11] Larissa. Conecta no mesmo banco, na mesma stack, com instância própria de PrismaClient, porque o client é por processo [09:30] Bruno. A leitura é por polling: a cada 2 segundos, busca os eventos pendentes mais antigos, entrega e marca [09:09] Diego. Enquanto houver um único worker, a ordenação por pedido fica preservada pela ordem de criação na outbox; a garantia é por order_id, não global [09:12]-[09:13] Diego/Larissa. Escalar para múltiplos workers, se um dia for necessário, passa por particionamento por order_id ou lock pessimista, tratado como problema do futuro [09:13] Diego.

## Prós e Contras das Opções

**Processo separado com polling de 2s**

- Prós: ciclo de vida independente da API; implementação simples com leitura indexada dos pendentes; latência mínima de 2 segundos no pior caso, aceita frente ao alvo de 10 segundos [09:10] Larissa.
- Contras: mais um processo para operar e monitorar; custo de polling constante no banco mesmo sem eventos; ordenação global não garantida se escalar workers.

**Worker embutido na API**

- Prós: um único processo para operar.
- Contras: restart da API perde o worker [09:11] Diego; acopla a entrega de webhook ao ciclo de deploy da API; rejeitado.

**Trigger do banco**

- Prós: reatividade imediata ao insert.
- Contras: MySQL não notifica processos externos; improvisar notificação via arquivo ou endpoint foi considerado esquisito [09:09] Diego; rejeitado.

## Consequências

Positivas: deploy e restart da API não afetam a entrega; a leitura indexada dos pendentes mantém o custo do polling baixo; a ordenação por pedido é preservada sem coordenação extra enquanto houver um worker.

Negativas: a latência mínima de entrega passa a ser o intervalo de polling (2 segundos no pior caso), aceita explicitamente [09:10] Larissa; o time passa a operar dois processos (API e worker), o que exige rotina de deploy e monitoramento próprios; a limitação de ordenação apenas por order_id e apenas com worker único fica documentada como restrição conhecida [09:13] Larissa.

O trade-off assumido: conviver com latência mínima de 2 segundos e com a operação de um segundo processo em troca de entrega desacoplada da API e implementação simples.

## Referências

- `src/server.ts` (padrão de entrypoint que o novo worker segue)
- `src/config/database.ts` (criação do PrismaClient por processo)
- `prisma/schema.prisma` (índices de status e created_at a criar na outbox)
- `TRANSCRICAO.md` [09:08]-[09:13], [09:30] (discussão e decisão)
