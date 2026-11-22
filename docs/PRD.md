# PRD: Sistema de Webhooks de Notificação de Pedidos

**Versão:** 1.0
**Data:** 2026-11-22
**Responsável:** Marcos (Product Manager)
**Documentos relacionados:** [RFC](RFC.md), [FDD](FDD.md), [ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-002](adrs/ADR-002-retry-com-backoff-e-dlq.md), [ADR-003](adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md), [ADR-004](adrs/ADR-004-garantia-at-least-once-com-x-event-id.md), [ADR-005](adrs/ADR-005-worker-separado-em-polling.md), [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md), [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md)

---

## 1. Resumo e contexto da feature

O OMS passa a notificar clientes B2B em tempo real quando o status de um pedido muda, por webhooks outbound: a plataforma faz HTTP POST para endpoints HTTPS cadastrados pelos próprios clientes, no momento em que o pedido muda de status. A entrega é assíncrona ao fluxo de pedidos, com retentativas em caso de falha e fila de eventos mortos com reprocessamento manual.

A feature entra no sistema existente: notificação e entrega ficam como módulo novo do monolito, com um processo worker separado para o envio [09:11] Diego. Nenhuma alteração de contrato nas APIs atuais.

---

## 2. Problema e motivação

- Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram formalmente notificação em tempo real de mudança de status. Hoje eles consultam o `GET /orders` de tempos em tempos para saber se algo mudou, o que torna a integração lenta e cara para eles [09:00] Marcos.
- A Atlas sinalizou risco de migrar para o concorrente se a entrega não acontecer até o fim do trimestre [09:00] Marcos.
- Para esses clientes, "tempo real" é qualquer coisa abaixo de 10 segundos; o que eles não aceitam é ficar pendurado atualizando manualmente [09:02] Marcos.
- O escopo é apenas de saída: eles querem receber eventos, não enviar [09:02] Sofia/Marcos.

A motivação é reter e viabilizar a integração dos clientes B2B, com prazo pressionado pelo risco de churn da Atlas.

---

## 3. Público-alvo e cenários de uso

**Público-alvo**
- Clientes B2B do OMS: Atlas Comercial, MaxDistribuição e Nova Cargo [09:00] Marcos
- Usuários operadores e admins da plataforma, que gerenciam os cadastros de webhook pela API do OMS [09:32] Marcos, [09:37] Sofia

**Cenários de uso chave**
- Cliente B2B recebe `order.status_changed` e atualiza o próprio ERP/logística sem polling
- Cliente que só se importa com expedição e entrega cadastra um webhook que escuta apenas SHIPPED e DELIVERED [09:33] Marcos
- Cliente investiga uma entrega que não chegou: consulta o histórico das últimas entregas com sucesso/falha, payload, response e tempo de resposta [09:34] Marcos
- Admin da plataforma reprocessa um evento que esgotou as retentativas, com trilha de auditoria [09:35]-[09:36] Diego/Sofia

---

## 4. Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Notificar mudança de status em tempo real | Latência do commit da mudança de status até a entrega no endpoint do cliente | Abaixo de 10 segundos [09:02] Marcos |
| Nenhum evento perdido | Eventos entregues com sucesso ou persistidos na dead letter queue após esgotar retentativas | 100% dos eventos [09:18] Diego |
| Entregar antes do prazo de churn da Atlas | Feature em produção | Até o fim de novembro, em 3 sprints incluindo revisão de segurança [09:45]-[09:47] Marcos/Larissa/Sofia |

---

## 5. Escopo

**Incluso**
- Cadastro, edição, remoção e listagem de webhooks por cliente [09:31]-[09:33]
- Filtro de eventos por status de interesse do pedido [09:33] Marcos
- Notificação de `order.status_changed` com garantia de registro na mesma transação da mudança de status [09:06] Diego, [09:40] Bruno
- Entrega com retentativas, backoff e dead letter queue [09:17]-[09:18] Diego
- Assinatura HMAC-SHA256 por endpoint e rotação de secret self service [09:20]-[09:21] Sofia
- Histórico de entregas por webhook [09:34] Marcos
- Reprocessamento administrativo da dead letter queue [09:35] Diego

**Fora de escopo**
- Notificação por email quando o webhook do cliente falha repetidamente [09:37] Larissa
- Dashboard visual para o cliente; apenas endpoints [09:40] Larissa
- Rate limiting de envio; observar em produção e decidir depois [09:39] Diego/Larissa
- Arquivamento de eventos entregues da outbox após 30 dias [09:08] Diego
- Garantia de ordenação global de eventos e escala para múltiplos workers [09:13] Diego
- Webhooks de entrada (inbound) [09:02] Sofia/Marcos

---

## 6. Requisitos funcionais

#### FR-01 Cadastro de webhook
O usuário autenticado cadastra um webhook com a URL do endpoint (obrigatoriamente https), o cliente dono e a lista de status que quer receber. A secret de assinatura é gerada pela plataforma e devolvida ao cliente apenas na criação.

**Fluxo principal**
- Usuário envia url, customer_id e lista de status
- Plataforma valida a URL https e cadastra o webhook com a secret gerada
- Resposta devolve o webhook criado com a secret, exibida uma única vez

**Fluxos alternativos e exceções**
- URL em http é rejeitada com erro de validação [09:23] Sofia
- O customer_id vai no body ou path, não no JWT [09:32] Larissa

**Erros previstos**
- `WEBHOOK_INVALID_URL` (URL não https)
- `WEBHOOK_SECRET_REQUIRED`

**Prioridade:** alta

#### FR-02 Edição de webhook
O usuário edita um webhook existente: URL, lista de status de interesse e estado ativo.

**Fluxo principal**
- Usuário envia os campos a alterar
- Plataforma valida e atualiza o cadastro

**Fluxos alternativos e exceções**
- URL em http é rejeitada como no cadastro

**Erros previstos**
- `WEBHOOK_NOT_FOUND`
- `WEBHOOK_INVALID_URL`

**Prioridade:** alta

#### FR-03 Remoção de webhook
O usuário remove um webhook; o cliente deixa de receber eventos a partir do momento da remoção.

**Fluxo principal**
- Usuário solicita a remoção pelo id
- Plataforma remove o cadastro

**Fluxos alternativos e exceções**
- Eventos já gravados na fila seguem o fluxo normal de entrega

**Erros previstos**
- `WEBHOOK_NOT_FOUND`

**Prioridade:** alta

#### FR-04 Listagem de webhooks
O usuário lista os webhooks de um cliente.

**Fluxo principal**
- Usuário consulta pelo customer_id
- Plataforma devolve os cadastros com URL, status de interesse e estado ativo

**Erros previstos**
- Cadastro inexistente não retorna erro; lista vazia

**Prioridade:** média

#### FR-05 Filtro de eventos por status
O cliente escolhe por webhook quais status de pedido quer ouvir; a plataforma filtra na hora de gravar o evento, não no envio.

**Fluxo principal**
- Status muda para um status que o webhook escuta
- Evento gravado para esse webhook

**Fluxos alternativos e exceções**
- Nenhum webhook do cliente escuta o status: nenhum evento é gravado, economizando fila [09:34] Bruno

**Erros previstos**
- Sem erro; ausência de evento é o comportamento esperado do filtro

**Prioridade:** alta

#### FR-06 Rotação de secret
O cliente pede uma nova secret de assinatura pelo endpoint próprio; a antiga continua válida por 24 horas em paralelo e depois é desativada.

**Fluxo principal**
- Cliente solicita a rotação
- Plataforma gera nova secret, devolve ao cliente e mantém a antiga válida por 24h [09:21] Sofia

**Fluxos alternativos e exceções**
- Durante o período de graça, as duas secrets validam a assinatura

**Erros previstos**
- `WEBHOOK_NOT_FOUND`

**Prioridade:** alta

#### FR-07 Notificação de mudança de status
Toda mudança de status de pedido gera evento para os webhooks inscritos, gravado na mesma transação da mudança de status.

**Fluxo principal**
- Status do pedido muda
- Evento gravado junto com a mudança, de forma atômica [09:06] Diego, [09:40] Bruno
- Evento entregue ao endpoint do cliente com assinatura e headers de deduplicação

**Fluxos alternativos e exceções**
- Transação da mudança de status sofre rollback: o evento some junto, sem inconsistência [09:06] Diego
- O payload reflete o estado do pedido no momento da mudança, não um estado posterior [09:52] Larissa

**Erros previstos**
- Falha no registro do evento reverte a mudança de status

**Prioridade:** alta

#### FR-08 Entrega com retentativa e dead letter queue
Entrega que falha por timeout ou resposta de erro é reagendada com backoff de 1m, 5m, 30m, 2h e 12h, no máximo 5 tentativas; esgotadas, o evento vai para a dead letter queue e nunca é descartado.

**Fluxo principal**
- Entrega falha
- Nova tentativa no intervalo do backoff [09:17] Diego
- Sucesso em qualquer tentativa encerra o ciclo

**Fluxos alternativos e exceções**
- 5 falhas consecutivas: evento persistido na dead letter queue com payload, motivo e timestamp [09:18] Diego
- Timeout de 10 segundos por tentativa conta como falha [09:42] Diego

**Erros previstos**
- `WEBHOOK_DELIVERY_FAILED`
- `WEBHOOK_PAYLOAD_TOO_LARGE` (payload acima de 64KB é rejeitado, sem truncamento) [09:23]-[09:24] Sofia/Diego

**Prioridade:** alta

#### FR-09 Reprocessamento administrativo da dead letter queue
Usuário com role ADMIN reprocessa um evento da dead letter queue, que volta para a fila como pendente; a ação fica registrada para auditoria.

**Fluxo principal**
- Admin solicita o replay pelo id do evento
- Evento volta para a fila como pendente [09:18] Diego
- Ação registrada com o autor, para auditoria [09:36] Sofia

**Fluxos alternativos e exceções**
- Operador (não ADMIN) recebe erro de permissão [09:36] Sofia/Larissa

**Erros previstos**
- `WEBHOOK_DEAD_LETTER_NOT_FOUND`
- `FORBIDDEN` (role sem permissão)

**Prioridade:** média

#### FR-10 Histórico de entregas
O cliente consulta as últimas 100 entregas de um webhook, com sucesso/falha, payload, response e tempo de resposta.

**Fluxo principal**
- Cliente consulta pelo id do webhook
- Plataforma devolve as últimas 100 entregas [09:34] Marcos

**Erros previstos**
- `WEBHOOK_NOT_FOUND`

**Prioridade:** média

---

## 7. Requisitos não funcionais

**Performance**
- Entrega do evento abaixo de 10 segundos contadas do commit da mudança de status [09:02] Marcos

**Segurança e autorização**
- Assinatura HMAC-SHA256 de todo evento, com secret por endpoint, para o cliente validar origem e integridade [09:20] Sofia
- URL do webhook obrigatoriamente https, rejeitada em validação caso contrário [09:23] Sofia
- Rotação de secret self service com período de graça de 24 horas [09:21] Sofia
- Reprocessamento de dead letter restrito à role ADMIN, com trilha de auditoria [09:36] Sofia/Larissa

**Confiabilidade e integridade de dados**
- Registro do evento atômico com a mudança de status: não existe estado com status alterado e evento ausente [09:06] Diego, [09:40]-[09:41] Bruno/Diego
- Entrega at-least-once: duplicatas são possíveis e o cliente deduplica pelo `X-Event-Id` [09:24]-[09:25] Diego
- Timeout de 10 segundos por tentativa de entrega [09:42] Diego
- Payload limitado a 64KB; acima disso o evento é rejeitado com erro, sem truncamento [09:23]-[09:24] Sofia/Diego

**Observabilidade**
- Histórico das últimas 100 entregas por webhook com status, payload, response e tempo de resposta [09:34] Marcos
- Comportamento at-least-once e deduplicação documentados de forma destacada no portal de desenvolvedores [09:26] Marcos

**Compatibilidade**
- Nenhuma mudança de contrato nas APIs existentes do OMS [09:40] Bruno
- Worker como processo separado, com deploy e restart independentes da API [09:11] Diego

---

## 8. Decisões e trade-offs principais

#### Decisão: outbox no MySQL em vez de fila dedicada
- **Justificativa:** o registro do evento acontece na mesma transação da mudança de status, sem estado inconsistente; o time é pequeno e subir Redis Cluster para isso seria overengineering [09:06]-[09:07] Diego
- **Trade-off:** a entrega depende de polling de 2 segundos, e não de push imediato; aceito porque atende à meta de 10 segundos [09:09] Diego
- Detalhe em [ADR-001](adrs/ADR-001-outbox-no-mysql.md) e [ADR-005](adrs/ADR-005-worker-separado-em-polling.md)

#### Decisão: at-least-once em vez de exactly-once
- **Justificativa:** exactly-once exigiria coordenação dos dois lados com complexidade muito maior; at-least-once com `X-Event-Id` resolve a quase totalidade dos casos e é o padrão de mercado seguido por Stripe e GitHub [09:25] Diego
- **Trade-off:** o cliente precisa implementar deduplicação pelo event_id; mitigado com documentação destacada no portal de desenvolvedores [09:26] Marcos
- Detalhe em [ADR-004](adrs/ADR-004-garantia-at-least-once-com-x-event-id.md)

#### Decisão: secret por endpoint com rotação, em vez de secret global
- **Justificativa:** conter o raio de um vazamento; houve precedente real de cliente que vazou secret em log [09:21]-[09:22] Sofia/Diego
- **Trade-off:** a plataforma passa a gerenciar segredos por cliente e o período de graça mantém duas secrets válidas por 24 horas
- Detalhe em [ADR-003](adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md)

#### Decisão: feature no monolito existente, com worker em processo separado
- **Justificativa:** reuso máximo dos padrões do projeto (módulos, erros, logger, middlewares) e velocidade de entrega dentro do prazo [09:27]-[09:30] Bruno/Larissa; worker separado desacopla a entrega do ciclo de restart da API [09:11] Diego
- **Trade-off:** um processo a mais para operar; a ordenação global de eventos não é garantida com worker único [09:12]-[09:13] Diego
- Detalhe em [ADR-005](adrs/ADR-005-worker-separado-em-polling.md) e [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)

---

## 9. Dependências

#### Organizacional: revisão de segurança antes do deploy
A Sofia (Engenheira de Segurança) revisa o código de segurança, com dois dias úteis reservados, focando HMAC e geração de secret; o deploy depende dessa revisão [09:46] Sofia.

#### Organizacional: documentação no portal de desenvolvedores
O Marcos documenta a integração via API no portal de desenvolvedores, com destaque para o comportamento at-least-once e a deduplicação por event_id [09:26] Marcos.

#### Técnica: estabilidade do fluxo de mudança de status
A integração acontece dentro da transação de mudança de status do serviço de pedidos; alterações concorrentes nesse fluxo precisam ser coordenadas [09:40] Bruno.

---

## 10. Riscos e mitigação

#### Cliente não implementa a deduplicação e sofre efeitos duplicados
- **Probabilidade:** média
- **Impacto:** cliente processa o mesmo evento duas vezes (ex.: baixa duplicada no ERP dele)
- **Mitigação:**
  - Documentar o comportamento at-least-once de forma destacada no portal de desenvolvedores [09:26] Marcos
  - Enviar `X-Event-Id` único por evento para viabilizar a deduplicação [09:25] Diego
- **Plano de contingência:** suporte direto com o cliente para implementar a deduplicação pelo event_id

#### Vazamento de secret de cliente
- **Probabilidade:** baixa (houve precedente real de cliente que vazou secret em log [09:22] Diego)
- **Impacto:** forja de eventos contra o endpoint do cliente afetado
- **Mitigação:**
  - Secret única por endpoint, nunca global [09:21] Sofia
  - Rotação self service com período de graça de 24 horas [09:21] Sofia
  - Revisão de segurança dedicada antes do deploy [09:46] Sofia
- **Plano de contingência:** cliente rotaciona a secret pelo endpoint próprio e invalida a antiga em até 24 horas

#### Prazo estourado e churn da Atlas
- **Probabilidade:** média
- **Impacto:** Atlas migra para o concorrente, como sinalizou formalmente [09:00] Marcos
- **Mitigação:**
  - Escopo deliberadamente enxuto: sem email, sem dashboard, sem rate limiting nesta fase [09:37]-[09:40]
  - Estimativa de 3 sprints incluindo a revisão de segurança, com confirmação de prazo com a Atlas [09:45]-[09:47] Marcos/Larissa
- **Plano de contingência:** escalonar prioridade do módulo de entrega e comunicar a Atlas antes do fim do prazo

#### Acúmulo de eventos entregues na fila
- **Probabilidade:** média
- **Impacto:** a fila cresce com o volume de pedidos e a leitura do worker fica lenta [09:07] Bruno
- **Mitigação:**
  - Monitorar a profundidade da fila como métrica primária
  - Trabalhador lê apenas eventos pendentes em lote pequeno [09:08] Diego
- **Plano de contingência:** arquivamento de eventos entregues após 30 dias, já discutido e adiado para depois desta fase [09:08] Diego

---

## 11. Critérios de aceitação

- Cliente cadastra um webhook e passa a receber eventos de mudança de status abaixo de 10 segundos após o commit [09:02] Marcos
- Evento só é gravado se a transação da mudança de status commitar; rollback remove o evento junto [09:06] Diego
- Toda entrega chega assinada com HMAC-SHA256 e o cliente consegue validar a assinatura com a secret do cadastro [09:20] Sofia
- Após rotação, a secret antiga valida por 24 horas e para de validar depois [09:21] Sofia
- Cliente que cadastrou filtro por status não recebe evento de status que não cadastrou [09:33] Marcos
- Evento com falha de entrega é retentado 5 vezes com os intervalos de 1m/5m/30m/2h/12h e, esgotado, está na dead letter queue com payload, motivo e timestamp [09:17]-[09:18] Diego
- Admin reprocessa um evento da dead letter queue e o evento volta como pendente; a ação fica com o autor registrado [09:18] Diego, [09:36] Sofia
- Cliente consulta as últimas 100 entregas do seu webhook com sucesso/falha, payload, response e tempo de resposta [09:34] Marcos
- Cadastro com URL http é rejeitado com erro de validação [09:23] Sofia
- Nenhuma API existente do OMS muda de contrato [09:40] Bruno

---

## 12. Estratégia de testes e validação

**Tipos de teste obrigatórios**
- Unitários: progressão do backoff, filtro de status na gravação, geração e verificação de assinatura HMAC
- Integração: atomicidade do evento com a mudança de status, rotação de secret com período de graça, percurso da dead letter queue
- Ponta a ponta: mudança de status até a entrega no endpoint de destino, incluindo headers e payload
- Segurança: permissão ADMIN no reprocessamento, rejeição de URL não https, auditoria do replay [09:36] Sofia

**Estratégia de validação**
- Revisão de segurança da Sofia antes do deploy, com dois dias úteis reservados para HMAC e geração de secret [09:46] Sofia
- Validação com clientes B2B reais após o deploy, começando pelos três que formalizaram o pedido [09:00] Marcos
- Observação em produção do volume de envios por cliente para decidir rate limiting no futuro [09:39] Diego/Larissa
