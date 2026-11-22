# Tracker de Rastreabilidade

Matriz de referência cruzada entre os documentos produzidos e suas origens na transcrição da reunião (`TRANSCRICAO.md`) ou no código existente. Cada requisito, decisão, contrato, erro, fluxo e risco citado nos documentos tem uma linha com a âncora verificável de onde veio.

Convenções:
- Um item que aparece em mais de um documento (por exemplo, a decisão de outbox no PRD e no ADR-001) tem uma linha por documento, porque cada documento é verificável de forma independente.
- Seções de síntese (resumo, TL;DR, critérios de aceitação, estratégia de testes) consolidam itens que já têm linha própria abaixo; não geram linhas duplicadas.
- Na coluna Localização, `TRANSCRICAO` traz o timestamp e o falante originais; `CODIGO` traz o caminho real do arquivo no repositório.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Entrega do evento abaixo de 10 segundos do commit à entrega | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | 100% dos eventos entregues ou persistidos na dead letter queue | TRANSCRICAO | [09:18] Diego |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Produção até o fim de novembro, em 3 sprints | TRANSCRICAO | [09:46] Larissa |
| PRD-OUT-01 | docs/PRD.md | Exclusão de escopo | Notificação por email em falha repetida | TRANSCRICAO | [09:37] Larissa |
| PRD-OUT-02 | docs/PRD.md | Exclusão de escopo | Dashboard visual para o cliente | TRANSCRICAO | [09:40] Larissa |
| PRD-OUT-03 | docs/PRD.md | Exclusão de escopo | Rate limiting de envio | TRANSCRICAO | [09:39] Diego |
| PRD-OUT-04 | docs/PRD.md | Exclusão de escopo | Arquivamento de eventos entregues após 30 dias | TRANSCRICAO | [09:08] Diego |
| PRD-OUT-05 | docs/PRD.md | Exclusão de escopo | Múltiplos workers e ordenação global | TRANSCRICAO | [09:13] Diego |
| PRD-OUT-06 | docs/PRD.md | Exclusão de escopo | Webhooks de entrada (inbound) | TRANSCRICAO | [09:02] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook com secret gerada na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Edição de webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remoção de webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listagem dos webhooks de um cliente | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status aplicado na inserção | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Rotação de secret com graça de 24 horas | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Notificação de mudança de status registrada na mesma transação | TRANSCRICAO | [09:06] Diego |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Entrega com retry 5x, backoff e dead letter queue | TRANSCRICAO | [09:17] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Replay administrativo da DLQ com auditoria | TRANSCRICAO | [09:35] Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência de entrega abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Assinatura HMAC-SHA256 por endpoint | TRANSCRICAO | [09:20] Sofia |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | URL do webhook obrigatoriamente https | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Rotação de secret self service | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Replay restrito à role ADMIN com trilha de auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Atomicidade do registro do evento com a mudança de status | TRANSCRICAO | [09:41] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | At-least-once com deduplicação por X-Event-Id | TRANSCRICAO | [09:25] Diego |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 segundos por tentativa | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Payload limitado a 64KB, sem truncamento | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Comportamento at-least-once documentado no portal de desenvolvedores | TRANSCRICAO | [09:26] Marcos |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Sem mudança de contrato nas APIs existentes | TRANSCRICAO | [09:40] Bruno |
| PRD-NFR-12 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado com deploy independente | TRANSCRICAO | [09:11] Diego |
| PRD-DEC-01 | docs/PRD.md | Decisão | Outbox no MySQL em vez de fila dedicada | TRANSCRICAO | [09:07] Diego |
| PRD-DEC-02 | docs/PRD.md | Decisão | At-least-once em vez de exactly-once | TRANSCRICAO | [09:25] Diego |
| PRD-DEC-03 | docs/PRD.md | Decisão | Secret por endpoint em vez de secret global | TRANSCRICAO | [09:21] Sofia |
| PRD-DEC-04 | docs/PRD.md | Decisão | Feature como módulo do monolito com worker separado | TRANSCRICAO | [09:27] Bruno |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança com 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Documentação da integração no portal de desenvolvedores | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-03 | docs/PRD.md | Dependência | Estabilidade da transação de mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-RISK-01 | docs/PRD.md | Risco | Cliente não implementar a deduplicação | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-02 | docs/PRD.md | Risco | Vazamento de secret de cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Churn da Atlas se o prazo estourar | TRANSCRICAO | [09:00] Marcos |
| PRD-RISK-04 | docs/PRD.md | Risco | Crescimento da fila de eventos com o volume | TRANSCRICAO | [09:07] Bruno |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Disparo síncrono no service de orders | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams como fila dedicada | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger de banco para reatividade | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Exactly-once delivery | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Renderizar o payload na hora do envio | TRANSCRICAO | [09:51] Bruno |
| RFC-OPEN-01 | docs/RFC.md | Questão em aberto | Rate limiting de envio | TRANSCRICAO | [09:39] Diego |
| RFC-OPEN-02 | docs/RFC.md | Questão em aberto | Permissões do CRUD de configuração | TRANSCRICAO | [09:37] Sofia |
| RFC-OPEN-03 | docs/RFC.md | Questão em aberto | Escala para múltiplos workers | TRANSCRICAO | [09:13] Diego |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Registro do evento na mesma transação e entrega pelo worker | TRANSCRICAO | [09:06] Diego |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Falha de entrega agenda retry com backoff | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Esgotadas as tentativas, evento movido para a dead letter | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | Replay recoloca o evento como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-05 | docs/FDD.md | Fluxo | Rotação mantém a secret antiga válida por 24 horas | TRANSCRICAO | [09:21] Sofia |
| FDD-FLUXO-06 | docs/FDD.md | Fluxo | Worker fora do ar retoma dos eventos mais antigos | TRANSCRICAO | [09:11] Diego |
| FDD-FUNC-01 | docs/FDD.md | Contrato | publishWebhookEvent(tx, order, fromStatus, toStatus) | TRANSCRICAO | [09:41] Bruno |
| FDD-ENDPOINT-01 | docs/FDD.md | Contrato | POST /api/v1/webhooks | TRANSCRICAO | [09:31] Marcos |
| FDD-ENDPOINT-02 | docs/FDD.md | Contrato | GET /api/v1/webhooks | TRANSCRICAO | [09:33] Bruno |
| FDD-ENDPOINT-03 | docs/FDD.md | Contrato | PATCH /api/v1/webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-ENDPOINT-04 | docs/FDD.md | Contrato | DELETE /api/v1/webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-ENDPOINT-05 | docs/FDD.md | Contrato | POST /api/v1/webhooks/:id/rotate-secret | TRANSCRICAO | [09:21] Sofia |
| FDD-ENDPOINT-06 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-ENDPOINT-07 | docs/FDD.md | Contrato | POST /api/v1/admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | Payload do evento com snapshot do pedido, sem items | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | Headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44] Diego |
| FDD-ERRO-01 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL (URL não https rejeitada) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-03 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-04 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE (acima de 64KB, sem truncamento) | TRANSCRICAO | [09:24] Diego |
| FDD-ERRO-05 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND (replay de id inexistente) | TRANSCRICAO | [09:35] Diego |
| FDD-ERRO-06 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_FAILED (não-2xx ou timeout) | TRANSCRICAO | [09:42] Diego |
| FDD-DADO-01 | docs/FDD.md | Modelo de dados | Tabela webhook_outbox com o evento, lida pelo worker | TRANSCRICAO | [09:06] Diego |
| FDD-DADO-02 | docs/FDD.md | Modelo de dados | Tabela webhook_dead_letter com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-DADO-03 | docs/FDD.md | Modelo de dados | Configuração com url, secret, customer_id e estado ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-DADO-04 | docs/FDD.md | Modelo de dados | Histórico de entregas com response e tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| FDD-DADO-05 | docs/FDD.md | Modelo de dados | event_id como UUID na outbox | TRANSCRICAO | [09:51] Larissa |
| FDD-INTEGRACAO-01 | docs/FDD.md | Integração | Transação de changeStatus com update, history e estoque | CODIGO | src/modules/orders/order.service.ts |
| FDD-INTEGRACAO-02 | docs/FDD.md | Integração | Hierarquia AppError com statusCode e errorCode | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INTEGRACAO-03 | docs/FDD.md | Integração | Middleware de erro centralizado trata AppError, Zod e Prisma | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INTEGRACAO-04 | docs/FDD.md | Integração | authenticate e requireRole por role (ADMIN/OPERATOR) | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INTEGRACAO-05 | docs/FDD.md | Integração | Logger Pino com redact configurado | CODIGO | src/shared/logger/index.ts |
| FDD-INTEGRACAO-06 | docs/FDD.md | Integração | Routers por módulo montados em /api/v1 | CODIGO | src/routes/index.ts |
| FDD-INTEGRACAO-07 | docs/FDD.md | Integração | PrismaClient próprio por processo do worker | CODIGO | src/config/database.ts |
| FDD-INTEGRACAO-08 | docs/FDD.md | Integração | Valores do filtro de eventos vêm do enum OrderStatus | CODIGO | prisma/schema.prisma |
| FDD-INTEGRACAO-09 | docs/FDD.md | Integração | Envelope paginado { data, pagination } nas listagens | CODIGO | src/shared/http/response.ts |
| FDD-INTEGRACAO-10 | docs/FDD.md | Integração | DELETE sem corpo com 204, padrão do projeto | CODIGO | src/modules/orders/order.controller.ts |
| FDD-INTEGRACAO-11 | docs/FDD.md | Integração | Formato ORD-NNNNNN do order_number no payload | CODIGO | src/modules/orders/order.service.ts |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs Pino no worker, sem biblioteca nova | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Secret nunca aparece em log (redact) | TRANSCRICAO | [09:22] Diego |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Profundidade da fila e eventos na DLQ como métricas primárias | TRANSCRICAO | [09:07] Bruno |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | Correlação request → outbox → entrega via X-Request-Id e event_id | CODIGO | src/middlewares/request-logger.middleware.ts |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox no MySQL, insert na mesma transação do changeStatus | TRANSCRICAO | [09:06] Diego |
| ADR-002 | docs/adrs/ADR-002-retry-com-backoff-e-dlq.md | Decisão | Retry 5 tentativas com backoff 1m/5m/30m/2h/12h e DLQ separada | TRANSCRICAO | [09:17] Diego |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 com secret por endpoint e rotação com graça de 24h | TRANSCRICAO | [09:20] Sofia |
| ADR-004 | docs/adrs/ADR-004-garantia-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id para deduplicação no cliente | TRANSCRICAO | [09:25] Diego |
| ADR-005 | docs/adrs/ADR-005-worker-separado-em-polling.md | Decisão | Worker em processo separado consumindo a outbox por polling de 2s | TRANSCRICAO | [09:09] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Reuso dos padrões existentes do projeto no módulo webhooks | TRANSCRICAO | [09:30] Larissa |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisão | Snapshot do payload renderizado na inserção da outbox | TRANSCRICAO | [09:52] Larissa |

## Resumo de cobertura

| Medida | Valor | Meta |
| --- | --- | --- |
| Linhas totais | 99 |  |
| Linhas com Fonte = TRANSCRICAO (timestamp e falante) | 87 (87,9%) | ≥70% |
| Linhas com Fonte = CODIGO (caminho real de arquivo) | 12 | ≥5 |
| Cobertura dos itens identificáveis dos documentos | FRs, NFRs, objetivos, escopo excluído, decisões, dependências, riscos, alternativas, questões em aberto, fluxos, contratos, erros, modelo de dados, integrações, observabilidade e ADRs, todos com linha | ≥80% |

Itens sem linha própria, por consolidarem itens que já têm linha: resumos/TL;DR dos documentos, critérios de aceitação e estratégia de testes do PRD e do FDD (consolidação de FRs, NFRs e contratos já rastreados) e blocos da proposta técnica do RFC (cada bloco aponta para o ADR correspondente, que tem linha).
