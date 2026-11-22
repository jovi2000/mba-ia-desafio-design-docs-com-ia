# Análise de Trabalho — Sistema de Webhooks de Notificação de Pedidos

> Arquivo de trabalho (não faz parte do pacote entregável). Serve como insumo consolidado para a geração dos design docs e para o TRACKER. Fontes: `TRANSCRICAO.md` e código-fonte do repositório.

## 1. Contexto do desafio

- Feature: **Sistema de Webhooks de Notificação de Pedidos** (outbound only) sobre o OMS existente.
- Motivação de negócio: 3 clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) fazem polling no `GET /orders`; integração lenta e cara; Atlas ameaça migrar para concorrente se não houver entrega até fim do trimestre `[09:00] Marcos`.
- Definição de "tempo real" acordada: **qualquer coisa abaixo de 10 segundos** `[09:02] Marcos`.
- Direção do fluxo: apenas saída (nós → clientes) `[09:02] Sofia/Marcos`. Webhook inbound não existe.
- Prazo: Atlas quer fim de novembro; estimativa de 3 sprints incluindo revisão de segurança `[09:45] Marcos / [09:46] Larissa / [09:47]`.
- Revisão de segurança: Sofia reserva pelo menos 2 dias úteis antes do deploy (HMAC e geração de secret) `[09:46] Sofia`.

## 2. Mapa do código existente (referências para os docs)

| O quê | Onde | Observação |
| --- | --- | --- |
| Entrypoint da API | `src/server.ts` | bootstrap Express + graceful shutdown. Não existe `src/worker.ts` ainda (a feature propõe criar) `[09:11] Larissa` |
| Montagem da app / DI manual | `src/app.ts` (`buildControllers`, `buildApp`) | Padrão a seguir para plugar o módulo webhooks |
| Rotas | `src/routes/index.ts` | Tudo sob `/api/v1`; novo router `webhooks` seria montado aqui |
| Máquina de estados do pedido | `src/modules/orders/order.status.ts` | PENDING→PAID→PROCESSING→SHIPPED→DELIVERED; CANCELLED a partir de PENDING/PAID/PROCESSING; DELIVERED e CANCELLED terminais |
| Transação do changeStatus | `src/modules/orders/order.service.ts` (`changeStatus`) | Transação única: validação de transição → débito/reposição de estoque → update da order → insert no `order_status_history`. **Ponto de inserção da outbox** `[09:40] Bruno` |
| Padrão de módulo | `src/modules/<dominio>/{controller,service,repository,routes,schemas}.ts` | Webhook segue igual: `src/modules/webhooks` `[09:27] Bruno` |
| Erros | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` | `AppError` base com `statusCode`, `errorCode`, `details`; subclasses (ex: `InvalidStatusTransitionError` código `INVALID_STATUS_TRANSITION`). Novos códigos com prefixo `WEBHOOK_` `[09:28] Bruno / [09:29] Larissa` |
| Error middleware | `src/middlewares/error.middleware.ts` | Trata AppError, ZodError e Prisma (P2002/P2025) sem mudança `[09:29] Bruno` |
| Auth | `src/middlewares/auth.middleware.ts` | `authenticate` (JWT Bearer, roles `ADMIN`/`OPERATOR`) + `requireRole`. Reaproveitar para replay de DLQ exigir ADMIN `[09:36] Larissa` |
| Logger | `src/shared/logger/index.ts` | Pino, redact, base `service: order-management-api`. Reuso, nada novo `[09:29] Bruno` |
| Request ID | `src/middlewares/request-logger.middleware.ts` | Gera `X-Request-Id` |
| Validação | `src/middlewares/validate.middleware.ts` + `*.schemas.ts` (Zod) | Validação de URL https entra aqui `[09:23] Sofia` |
| Prisma client | `src/config/database.ts` | Worker usa instância própria de PrismaClient (por processo, mesmo banco/DATABASE_URL) `[09:30] Bruno` |
| Env | `src/config/env.ts` | Zod; novas variáveis (ex: intervalo de polling) seguiriam este padrão |
| Respostas | `src/shared/http/response.ts` | Helpers de paginação (`paginated`) |
| Schema do banco | `prisma/schema.prisma` | Tabelas atuais: users, customers, products, orders, order_items, order_status_history, order_number_sequence. IDs UUID `Char(36)` em todo o projeto (padrão citado em `[09:51] Larissa`) |

## 3. Decisões fechadas (as 6 principais do enunciado + secundárias)

### Principais (devem virar ADRs; cobrir no mínimo 5 das 6)

1. **Padrão Outbox no MySQL** `[09:06] Diego`, decidido `[09:08] Larissa`. Insert na `webhook_outbox` dentro da mesma transação SQL do `changeStatus`. Commit ⇒ evento registrado; rollback ⇒ evento some junto. Sem inconsistência possível.
2. **Worker em processo separado, polling de 2s** `[09:09] Diego`, `[09:10] Larissa`. Entry nova `src/worker.ts` + script `npm run worker` `[09:11] Larissa`. Processo separado para não morrer junto com restart da API `[09:11] Diego`. Mesmo banco, mesma stack, PrismaClient próprio `[09:30] Bruno/Diego`. Latência mínima no pior caso: 2s, aceita contra o alvo de <10s `[09:10] Larissa`.
3. **Retry com backoff e DLQ** `[09:15] Diego`, `[09:17] Larissa`. 5 tentativas, backoff 1m/5m/30m/2h/12h (total ~15h entre primeira falha e última tentativa). Depois: DLQ persistida em tabela separada `webhook_dead_letter` com payload, motivo da falha e timestamp `[09:18] Diego`.
4. **HMAC-SHA256 com secret por endpoint** `[09:19]-[09:22] Sofia`. Assinatura sobre o corpo do request no header `X-Signature`. Secret única por endpoint (não global). Secret rotacionável via endpoint; na rotação a antiga fica válida 24h em paralelo (grace period) `[09:21] Sofia`.
5. **At-least-once com `X-Event-Id`** `[09:24]-[09:26] Diego/Sofia/Larissa`. UUID gerado quando o evento entra na outbox; cliente deduplica do lado dele. Padrão de mercado (Stripe, GitHub). Exactly-once descartado.
6. **Reuso dos padrões existentes** `[09:30] Larissa`. AppError, Pino, error middleware, padrão de módulos, schemas Zod, padrão de códigos de erro, prefixo `WEBHOOK_`.

### Secundárias (ADR extra ou conteúdo de FDD)

- Índices da outbox: campo de status (pendente/processando/falhou/entregue) e `created_at`; worker lê só pendentes em batch pequeno `[09:08] Diego`.
- Single-worker: ordering implícito por `order_id` (ordem de `created_at` da outbox). Sem garantia de ordering global. Escala futura: particionar por order_id ou lock pessimista (não agora) `[09:12] Diego / [09:13] Larissa`.
- ID da outbox: UUID, seguindo o padrão do projeto `[09:51] Diego/Larissa`.
- Snapshot do payload renderizado na inserção (não no envio); evento reflete o estado no momento da mudança `[09:52] Larissa/Diego/Bruno`.
- Integração via função `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebendo o tx client da transação corrente; sem injetar repository inteiro no OrderService `[09:41] Bruno/Diego`.
- Timeout do HTTP call do worker: 10s; sem resposta = falha e agendamento de retry `[09:42] Diego/Sofia`.
- `customer_id` passado no body ou path, **não** extraído do JWT (JWT é do usuário operador) `[09:32] Bruno/Marcos/Larissa`.
- Filtro de eventos por status aplicado **na inserção** da outbox; se nenhum webhook do customer quer aquele status, não insere `[09:34] Bruno/Diego`.

## 4. Requisitos funcionais (com timestamp)

| # | Requisito | Fonte |
| --- | --- | --- |
| FR-1 | Cadastrar webhook via POST. Campos: url, lista de status que quer receber; secret é **gerada pelo sistema e devolvida na criação**; customer_id no body ou path | `[09:31] Marcos`, `[09:32] Larissa` |
| FR-2 | Editar webhook via PATCH | `[09:33] Bruno` |
| FR-3 | Remover webhook via DELETE | `[09:33] Bruno` |
| FR-4 | Listar webhooks de um customer via GET | `[09:33] Bruno` |
| FR-5 | Filtro de eventos por status por endpoint, aplicado na inserção da outbox | `[09:33] Marcos`, `[09:34] Bruno` |
| FR-6 | Histórico de entregas: `GET /webhooks/:id/deliveries` com últimos 100 envios, sucesso/falha, payload, response, tempo de resposta | `[09:34] Marcos` |
| FR-7 | Replay manual de DLQ: `POST /admin/webhooks/dead-letter/:id/replay`, recoloca na outbox como pendente, exige role ADMIN, log de auditoria de quem fez | `[09:18] Diego`, `[09:36] Sofia/Larissa` |
| FR-8 | Rotação de secret via endpoint; antiga válida por 24h em paralelo | `[09:21] Sofia` |
| FR-9 | Envio HTTP assinado com headers `X-Event-Id`, `X-Signature`, `X-Timestamp` (envio; anti replay attack), `X-Webhook-Id`, `Content-Type: application/json` | `[09:44] Diego`, `[09:44] Sofia` |
| FR-10 | Payload enxuto: event_id, event_type `"order.status_changed"`, timestamp ISO 8601, order_id, order_number, from_status, to_status, customer_id, total_cents. Sem items; detalhe via `GET /orders/:id` | `[09:43] Diego` |

## 5. Requisitos não funcionais

- Latência de notificação: below 10s conta como "tempo real" (alvo); latência mínima pior caso 2s pelo polling `[09:02] Marcos`, `[09:10] Larissa`.
- TLS obrigatório: URL do webhook deve ser `https`; `http` recusado com erro de validação (Zod) `[09:23] Sofia`.
- Limite de payload: 64KB; acima disso, erro (não trunca) `[09:23] Sofia`, `[09:24] Diego/Larissa`.
- Timeout HTTP do worker: 10s `[09:42] Diego`.
- Segurança: HMAC-SHA256, secret por endpoint, rotação com grace 24h `[09:22] Sofia`. Revisão de segurança (2 dias úteis) antes do deploy `[09:46] Sofia`.
- Autorização: CRUD de configuração com qualquer role autenticada (por enquanto) `[09:37] Sofia`; replay DLQ só ADMIN `[09:36] Sofia`.

## 6. Fora de escopo (descartado ou adiado explicitamente)

| Item | Status | Fonte |
| --- | --- | --- |
| Email de alerta ao cliente por falhas (ex: 3 seguidas) | Fora desta fase; talvez próxima, depois de medir impacto | `[09:37] Larissa` |
| Dashboard/painel visual para clientes | Fora; projeto separado do time de frontend | `[09:39] Marcos`, `[09:40] Larissa` |
| Rate limiting de envio para clientes | Ponto em aberto: observar e decidir depois | `[09:38] Diego`, `[09:39] Larissa` |
| Arquivamento de linhas entregues da outbox (30 dias) | Fora do escopo da feature | `[09:08] Diego` |
| Escala multi-worker / ordering global | Limitação conhecida; problema do futuro | `[09:13] Diego/Larissa` |
| Webhook inbound (cliente → nós) | Descartado; só outbound | `[09:02] Sofia/Marcos` |

## 7. Alternativas descartadas (para o RFC, com trade-off)

| Alternativa | Trade-off que levou ao descarte | Fonte |
| --- | --- | --- |
| Disparo síncrono no service de orders | Transação de status já pesada (orders, history, estoque); HTTP call travaria mudança de status de outros pedidos; cliente fora do ar forçaria rollback impossível | `[09:04] Bruno`, `[09:06] Diego` |
| Redis Streams (ou fila dedicada) | Infra adicional para time pequeno; overengineering; outbox no MySQL existente resolve | `[09:07] Larissa/Diego` |
| Trigger de banco para reatividade | MySQL não tem listener nativo (NOTIFY/LISTEN); trigger só executa SQL, não notifica processo externo | `[09:09] Diego` |
| Retry indefinido com backoff | Evento fica pendurado para sempre se o cliente sumiu | `[09:15] Diego` |
| 3 tentativas de retry | Insuficiente: indisponibilidades reais de 2h em manutenção planejada; 3 retries morreriam em ~30 min | `[09:16] Bruno/Diego` |
| DLQ como status "failed" na própria outbox | Tabela separada deixa a outbox principal limpa e serve de evidence para debug/reprocessamento | `[09:17] Larissa`, `[09:18] Diego` |
| Exactly-once delivery | Exigiria coordenação dos dois lados; complexidade muito maior | `[09:25] Diego` |
| Guardar só order_id e renderizar payload no envio | Snapshot na inserção garante que o evento reflete o estado no momento da mudança | `[09:51] Bruno`, `[09:52] Larissa` |

## 8. Questões em aberto (para o RFC)

- Rate limiting de saída: se um cliente tem 50 mudanças de status por minuto, bombardeamos com 50 chamadas? Decisão: observar e implementar se virar problema `[09:38] Diego`, `[09:39] Larissa`.
- Endurecer permissões do CRUD de configuração (hoje qualquer role autenticada) `[09:36] Marcos`, `[09:37] Sofia`.
- Escalonamento do worker (multi-worker, particionamento por order_id, lock pessimista) `[09:13] Bruno/Diego`.

## 9. Erros citados na transcrição (semente da matriz `WEBHOOK_*`)

- Códigos citados explicitamente: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, "etc." `[09:28] Bruno`. O "etc." autoriza derivar códigos adicionais no mesmo padrão para as falhas previstas (validação de URL não-https, payload acima de 64KB, falha de entrega, replay inexistente etc.), desde que cada um esteja referenciado no requisito correspondente.
- Padrão existente a seguir: `INVALID_STATUS_TRANSITION`, `INSUFFICIENT_STOCK` (código uppercase, underscore) `src/shared/errors/http-errors.ts`.

## 10. Cuidados de geração (anti-alucinação)

1. **Não inventar**: todo item de PRD/RFC/FDD/ADR precisa de linha no TRACKER com Fonte = TRANSCRICAO (timestamp) ou CODIGO (path). Se não há fonte, remover.
2. **Não transformar descartado em requisito**: email, dashboard, rate limiting, arquivamento, multi-worker e inbound são fora de escopo/ponto em aberto. Se aparecerem como requisito, está errado.
3. **Completude legítima**: caminhos exatos do CRUD (ex: `POST /api/v1/webhooks`), nomes de colunas da outbox e códigos de erro além dos citados são detalhamento de implementação que o FDD precisa concretizar, mas devem permanecer dentro do que foi decidido (sem introduzir requisitos novos).
4. **Altura de cada documento**: PRD = produto/negócio; RFC = proposta de arquitetura + alternativas + abertos (conciso, 2-4 páginas); ADR = decisão isolada; FDD = implementação detalhada (fluxos, contratos, erros, resiliência, observabilidade, integração com ≥4 arquivos reais).
5. **Critérios objetivos do enunciado**: PRD ≥8 FRs, ≥1 objetivo com métrica/meta, ≥2 fora de escopo, ≥2 riscos com probabilidade/impacto/mitigação; RFC ≥2 alternativas descartadas com trade-off, ≥2 questões em aberto, links para ≥2 ADRs; FDD ≥4 endpoints com payload request/response, matriz `WEBHOOK_*`, seção "Integração com o sistema existente" com ≥4 paths reais, observabilidade com métricas+logs+tracing; 5-8 ADRs cobrindo ≥5 das 6 decisões principais, ≥1 ADR citando código; TRACKER com ≥80% de cobertura, ≥70% das linhas TRANSCRICAO com timestamp válido, ≥5 linhas CODIGO com path real.
6. **Formato ADR**: MADR (7 seções) conforme prompt do professor, adaptado: sem `[NEEDS INPUT]` (fontes existem), arquivos `docs/adrs/ADR-NNN-titulo-kebab.md`, referências de código como caminhos de arquivo.
7. **Ordem de produção sugerida pelo enunciado**: ADRs → RFC → FDD → PRD → TRACKER → README (processo) → revisão final contra a checklist.

## 11. Formato/estrutura dos docs (origem dos formatos)

- **PRD**: usar o esqueleto de saída do `prompts/prd-generator.md` (entrevista não se aplica; fonte = transcrição).
- **ADR**: usar `prompts/adr-generator.md` como base (MADR 7 seções, concisão, sem code snippets), adaptado conforme item 10.6.
- **FDD**: seções obrigatórias do INSTRUCTIONS.MD (+ conteúdo do `prompts/fdd-generator.md` quando disponível).
- **RFC**: seções obrigatórias do INSTRUCTIONS.MD (não há prompt específico).
- **Autor do RFC**: Larissa (Tech Lead), que abriu o doc de design na transcrição `[09:50] Larissa`. Revisores: Marcos, Bruno, Diego, Sofia.

## 12. Log de iterações (insumo para a seção "Iterações e ajustes" do README)

Registro das correções reais feitas durante o processo de geração e revisão crítica. Atualizar a cada etapa.

1. **Etapa ADRs (2026-11-22)**: na revisão crítica do pacote de ADRs, a frase "o mesmo modelo de provedores consolidados como Stripe e GitHub" do ADR-003 estava citada com o timestamp `[09:25] Diego`, que na transcrição se refere ao padrão at-least-once com event_id, e não à assinatura HMAC. Corrigido para a citação correta da Sofia sobre HMAC-SHA256 como padrão de mercado `[09:20]`. Motivo do erro: generalização de "padrão de mercado" sem conferir a fonte exata.

## 13. Insumos para o README do processo

Coleta incremental do que a versão final do `README.md` vai precisar. Atualizar a cada etapa.

### Ferramentas de IA utilizadas

- **Claude Code (CLI)** como ferramenta principal: agente com acesso direto ao repositório (leitura do código-fonte para mapear referências, edição dos arquivos Markdown e execução de verificações via bash).
- Prompts do professor (`prompts/`) usados como base, adaptados por etapa (detalhado abaixo).

### Uso dos prompts por etapa (o que foi usado e o que foi adaptado)

- **ADRs (Etapa 1)**: `prompts/adr-generator.md` usado como base. Mantido: formato MADR de 7 seções, header (Status/Data/ADRs relacionados), limites de concisão (máx 3 opções, máx 5 referências de arquivo, sem code snippets), tradução pt-BR. Adaptado fora: pipeline de "potential ADRs" mineradas do histórico git, marcadores `[NEEDS INPUT]`, classificação em tiers e saída em `docs/adrs/generated/{MODULE}/` (aqui a fonte é a transcrição, não o git, e o INSTRUCTIONS.MD exige `docs/adrs/ADR-NNN-titulo.md`).
- **RFC (Etapa 2)**: sem prompt do professor para RFC; documento estruturado pelas seções obrigatórias do INSTRUCTIONS.MD.
- **FDD (Etapa 3)**: `prompts/fdd-generator.md` como base. Mantido: esqueleto de saída com 10 seções. Adaptado: a entrevista estruturada não se aplica (a fonte, transcrição + código, já existe); adicionada a seção obrigatória do desafio "Integração com o sistema existente". Regras de diagramas de `prompts/mermaid-diagram-generator.md` aplicadas inline (mermaid embutido no FDD, sem arquivo separado).
- **PRD (Etapa 4)**: `prompts/prd-generator.md` como base. Mantido: esqueleto de saída. Adaptado: mesma razão do FDD, sem entrevista.

### Workflow adotado (fatos para o README)

1. Análise prévia: leitura completa de INSTRUCTIONS.MD, da transcrição e do código; mapa decisor consolidado em `WORK_ANALYSIS.md` (decisões com timestamps, referências de código, fora de escopo, alternativas descartadas, questões em aberto).
2. Ordem de produção: ADRs → RFC → FDD → PRD → TRACKER → README (ordem sugerida pelo enunciado).
3. Revisão por etapa com checkpoint do autor (papel de maestro), não geração de uma vez.
4. Verificação manual via bash ao final de cada etapa: timestamps citados existem na TRANSCRICAO.md, paths de código existem, contagens, ausência de travessão.
5. Log de iterações na seção 12 deste arquivo; o README final cita as correções concretas.

### Iterações até agora

- Ver seção 12 (log de iterações). Contagem até o momento: 5 correções concretas (ADR-003 citação errada; RFC citação malformada; FDD ajustes de mermaid e nomenclatura; PRD prazo do objetivo impreciso; TRACKER referência da linha de dados e bug do script de verificação).

2. **Etapa RFC (2026-11-22)**: revisão crítica do RFC encontrou uma citação malformada no bloco de configuração ("[09:31]-09:34]", sem o colchete de abertura do segundo timestamp), corrigida para "[09:31]-[09:34]". Erro de digitação capturado na passada de revisão antes do checkpoint; a verificação manual (grep de timestamps contra a TRANSCRICAO.md) confirmou o restante.

3. **Etapa FDD (2026-11-22)**: revisão crítica dos diagramas mermaid encontrou três desvios das regras de `prompts/mermaid-diagram-generator.md`: participantes com espaço no nome sem aspas (`API Express`, `Endpoint cliente`), labels de nó acima do limite de 3 palavras ("Envia HTTP timeout 10s", "Tentativa menor que 5?") e entidades do ER em MAIÚSCULAS quando os ADRs e a transcrição citam as tabelas em minúsculas (`webhook_outbox`, `webhook_dead_letter`). Todos corrigidos antes do checkpoint; verificação manual de timestamps e paths rodou limpa.

4. **Etapa PRD (2026-11-22)**: revisão crítica pegou imprecisão de fonte na tabela de objetivos: a meta de prazo dizia "fim do trimestre" citando [09:45]-[09:47], mas nesse trecho da transcrição o prazo concreto é "fim de novembro" (Marcos); "fim do trimestre" é a formulação de [09:00]. A meta foi corrigida para "até o fim de novembro", deixando "fim do trimestre" apenas na seção de problema, onde a citação é [09:00].

5. **Etapa TRACKER (2026-11-22)**: dois ajustes. Primeiro, a linha FDD-DADO-01 misturava dois fatos com referências diferentes: o nome da tabela `webhook_outbox` vem de [09:06] Diego e o índice por status/created_at vem de [09:08] Diego; o conteúdo da linha foi restrito ao que a referência de fato sustenta. Segundo, o próprio script de verificação tinha um bug (a variável extraída já continha os colchetes do timestamp e virava classe de caracteres na regex, gerando falso negativo em todos os 87 pares); o check foi reescrito com prefixo fixo via awk e passou. Lição registrada para o README: a verificação manual também precisa ser verificada quando falha de forma suspeita.
