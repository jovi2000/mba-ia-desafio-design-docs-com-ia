# RFC: Sistema de Webhooks de Notificação de Pedidos

**Autora:** Larissa (Tech Lead)
**Status:** Proposto
**Data:** 2026-11-22
**Revisores:** Marcos (Product Manager), Bruno (Engenheiro Pleno, time de Pedidos), Diego (Engenheiro Sênior, time de Plataforma), Sofia (Engenheira de Segurança)

**Documentos relacionados:** [ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-002](adrs/ADR-002-retry-com-backoff-e-dlq.md), [ADR-003](adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md), [ADR-004](adrs/ADR-004-garantia-at-least-once-com-x-event-id.md), [ADR-005](adrs/ADR-005-worker-separado-em-polling.md), [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md), [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md)

---

## 1. Resumo executivo (TL;DR)

Este RFC propõe um sistema de notificação de mudanças de status de pedidos via webhooks outbound para clientes B2B. A gravação do evento usa o padrão outbox no MySQL, dentro da mesma transação da mudança de status, o que garante que status alterado implica evento registrado. Um worker dedicado, em processo separado, consome a outbox por polling de 2 segundos e entrega os eventos assinados com HMAC-SHA256, com secret por endpoint. Entregas que falham passam por retry com backoff (5 tentativas) e, ao esgotar, vão para uma DLQ com reprocessamento manual. A garantia de entrega é at-least-once, com event_id para deduplicação no cliente. A configuração fica em módulo novo no projeto, reusando os padrões existentes. Estimativa: 3 sprints, incluindo revisão de segurança antes do deploy.

## 2. Contexto e problema

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram formalmente notificação de mudança de status de pedidos. Hoje eles fazem polling no endpoint de listagem de pedidos de tempos em tempos, o que deixa a integração lenta e cara para eles; a Atlas sinalizou risco de migração para o concorrente se não houver entrega até o fim do trimestre [09:00] Marcos. Para os clientes, "tempo real" é qualquer coisa abaixo de 10 segundos [09:02] Marcos, e o fluxo é apenas de saída: eles querem receber, não enviar [09:02] Sofia/Marcos.

O sistema atual não tem nenhum mecanismo de notificação externa, eventos, filas ou webhooks. A mudança de status de pedido já é uma transação SQL de peso: atualiza o pedido, grava o histórico de status e ajusta estoque [09:04] Bruno. Disparar HTTP dentro desse fluxo travaria mudanças de status de outros pedidos e criaria o dilema impossível de reverter uma mudança de status porque um cliente externo está fora do ar [09:04] Bruno, [09:06] Diego.

## 3. Proposta técnica

A proposta tem seis blocos. O detalhamento de implementação (fluxos, contratos, erros) fica no FDD; aqui interessa a forma da solução e o que ela garante.

**Registro do evento: outbox no MySQL.** Na mesma transação que muda o status do pedido, o sistema insere a linha do evento em uma tabela de outbox própria. Se a transação commita, o evento existe; se dá rollback, o evento some junto. Não há estado em que o status mudou e o evento não foi registrado [09:06] Diego. O evento é gravado já renderizado (snapshot), refletindo o estado do pedido no momento da mudança [09:52] Larissa. Detalhes em [ADR-001](adrs/ADR-001-outbox-no-mysql.md) e [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md).

**Entrega: worker dedicado em polling.** Um processo separado da API, no mesmo banco e na mesma stack, lê os eventos pendentes a cada 2 segundos e os entrega por HTTP com timeout de 10 segundos [09:09] Diego, [09:42] Diego. Separar o processo desacopla a entrega do ciclo de deploy da API [09:11] Diego. Enquanto houver um único worker, a ordem de entrega por pedido é preservada; ordenação global não é garantida e fica documentada como limitação [09:12]-[09:13] Diego/Larissa. Detalhes em [ADR-005](adrs/ADR-005-worker-separado-em-polling.md).

**Resiliência: retry com backoff e DLQ.** Falha de entrega reagenda com backoff exponencial de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, no máximo 5 tentativas [09:17] Diego. Esgotadas as tentativas, o evento é persistido em tabela de dead letter separada, com payload, motivo da falha e timestamp, e pode ser reprocessado manualmente por endpoint administrativo com role ADMIN [09:18] Diego, [09:36] Sofia/Larissa. Detalhes em [ADR-002](adrs/ADR-002-retry-com-backoff-e-dlq.md).

**Segurança: HMAC-SHA256 por endpoint.** Todo evento é assinado com HMAC-SHA256 sobre o corpo do request, com a assinatura em header próprio. Cada endpoint de webhook tem uma secret única, gerada pela plataforma e devolvida ao cliente na criação; a rotação mantém a secret antiga válida por 24 horas em paralelo [09:20]-[09:21] Sofia. A URL do webhook precisa ser HTTPS, recusada em validação caso contrário [09:23] Sofia. Detalhes em [ADR-003](adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md).

**Semântica de entrega: at-least-once.** Com retry e timeout, duplicatas são inevitáveis; a plataforma garante at-least-once e o cliente deduplica pelo event_id enviado em header próprio, padrão de mercado seguido por Stripe e GitHub [09:24]-[09:25] Diego. O comportamento fica documentado de forma destacada no portal de desenvolvedores [09:26] Marcos. Detalhes em [ADR-004](adrs/ADR-004-garantia-at-least-once-com-x-event-id.md).

**Configuração e integração: módulo novo nos padrões do projeto.** O cliente gerencia seus webhooks por endpoints autenticados (criação, edição, remoção, listagem, rotação de secret, filtro por status de interesse e histórico de entregas) [09:31]-[09:34]. O módulo segue a estrutura padrão do projeto (controller, service, repository, routes, schemas), com erros na hierarquia existente usando prefixo WEBHOOK_, o mesmo logger e o mesmo middleware de erro [09:27]-[09:30] Bruno/Larissa. A extensão da mudança de status para gravar a outbox acontece por função dedicada que recebe a transação corrente, sem injetar dependências extras no serviço de pedidos [09:41] Bruno/Diego. Detalhes em [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md).

## 4. Alternativas consideradas

Alternativas reais colocadas na mesa e descartadas na reunião:

| Alternativa | Trade-off que levou ao descarte | Origem |
| --- | --- | --- |
| Disparo síncrono dentro do service de orders | Cliente lento travaria mudança de status de outros pedidos; cliente fora do ar forçaria rollback de uma mudança de status legítima | [09:03]-[09:04] Bruno, [09:06] Diego |
| Redis Streams (ou fila dedicada) | Subir e operar Redis Cluster para um time pequeno é overengineering; o MySQL existente resolve | [09:07] Larissa/Diego |
| Reatividade via trigger do banco | MySQL não tem listener nativo para processos externos (tipo NOTIFY/LISTEN); trigger só executa SQL, notificar o worker por mecanismo improvisado foi considerado esquisito | [09:09] Diego |
| Exactly-once delivery | Exigiria coordenação e ack dos dois lados, com complexidade muito maior; at-least-once com event_id resolve a quase totalidade dos casos | [09:25] Diego |
| Renderizar o payload na entrega, guardando só referências | O evento poderia refletir estado posterior do pedido em vez do estado no momento da mudança, o "caso esquisito" apontado na discussão | [09:51]-[09:52] Bruno/Larissa |

## 5. Questões em aberto

Pontos levantados na reunião e deliberadamente não decididos nesta fase:

1. **Rate limiting de envio**: se um cliente tem 50 mudanças de status num minuto, recebe 50 chamadas? Decisão: observar em produção e implementar se virar problema [09:38] Diego, [09:39] Larissa.
2. **Permissões do CRUD de configuração**: hoje qualquer role autenticada pode gerenciar webhooks; endurecer fica para depois [09:36] Marcos, [09:37] Sofia.
3. **Escala para múltiplos workers**: a ordenação por pedido só vale para worker único; particionamento por order_id ou lock pessimista são problemas do futuro [09:13] Diego.

## 6. Impacto e riscos

**Impacto.** Novo módulo no projeto e um novo processo para operar (worker). Extensão da transação de mudança de status, controlada em função dedicada. Novas tabelas no MySQL existente. Nenhuma mudança de contrato nas APIs atuais. Revisão de segurança obrigatória antes do deploy, com dois dias úteis reservados para HMAC e geração de secret [09:46] Sofia. Estimativa de 3 sprints incluindo essa revisão [09:46]-[09:47] Larissa/Sofia.

**Riscos principais.**

- Vazamento de secret de cliente: mitigado por secret única por endpoint, rotação self service e redact de campos sensíveis nos logs; houve precedente real de cliente que vazou secret em log [09:21]-[09:22] Sofia/Diego.
- Acúmulo de eventos na outbox: a tabela cresce com o volume; o arquivamento de linhas entregues após 30 dias foi discutido e deixado fora do escopo desta fase [09:08] Diego.
- Cliente não implementar a deduplicação: efeitos duplicados passam a ser problema do integrador; mitigação é documentação destacada no portal de desenvolvedores [09:25]-[09:26] Diego/Marcos.

## 7. Decisões relacionadas

| Decisão | ADR |
| --- | --- |
| Outbox no MySQL, transação atômica com a mudança de status | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Retry com backoff 1m/5m/30m/2h/12h, 5 tentativas, DLQ separada | [ADR-002](adrs/ADR-002-retry-com-backoff-e-dlq.md) |
| HMAC-SHA256, secret por endpoint, rotação com grace de 24h | [ADR-003](adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md) |
| At-least-once com X-Event-Id | [ADR-004](adrs/ADR-004-garantia-at-least-once-com-x-event-id.md) |
| Worker em processo separado, polling de 2s | [ADR-005](adrs/ADR-005-worker-separado-em-polling.md) |
| Reuso dos padrões existentes do projeto | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) |
| Snapshot do payload na inserção, UUID como id | [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md) |
