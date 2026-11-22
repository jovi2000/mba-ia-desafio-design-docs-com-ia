# Plano de Implementação — Design Docs do Sistema de Webhooks de Notificação de Pedidos

> Plano de produção do pacote de design docs do desafio. Acompanhamento de execução na seção final.

## Contexto

Desafio da pós-graduação (FullCycle/MBA IA): transformar `TRANSCRICAO.md` (reunião técnica de ~55min que decidiu a feature de webhooks de pedidos) + código do OMS existente em um pacote de design docs acionável. Entrega 100% documental em fork próprio do GitHub; o código (`src/`, `prisma/`, `tests/`, configs) e `TRANSCRICAO.md` são intocáveis. Regra central: todo item de documentação precisa ser rastreável à transcrição (timestamp + falante) ou ao código (path real).

**Insumos:**
- `WORK_ANALYSIS.md` (raiz do repo): mapa decisor com timestamps, referências de código, FRs, NFRs, fora de escopo, alternativas descartadas, questões em aberto, semente da matriz `WEBHOOK_*`, cuidados anti-alucinação e log de iterações
- `prompts/` do professor: `prd-generator.md`, `fdd-generator.md`, `adr-generator.md` (mermaid e c4 como referência de regras)
- `INSTRUCTIONS.MD`: seções obrigatórias e checklist de critérios de aceite

**Decisões de processo:**
- Pasta `prompts/` permanece no repo entregue
- RFC com autora Larissa (Tech Lead); revisores: Marcos, Bruno, Diego, Sofia
- Diagramas mermaid embutidos no FDD (sem arquivos extras)
- Revisão por etapa: checkpoint com o usuário após cada documento/grupo
- Adaptações de formato: PRD e FDD usam os esqueletos de saída dos prompts (a entrevista não se aplica; a fonte já existe); ADRs usam MADR adaptado de `prompts/adr-generator.md` (7 seções, sem o pipeline de git-archaeology e sem `[NEEDS INPUT]`); RFC estruturado pelo INSTRUCTIONS.MD

## Arquivos produzidos

| Arquivo | Ação |
| --- | --- |
| `docs/adrs/ADR-001-outbox-no-mysql.md` … `ADR-007-snapshot-do-payload-na-insercao.md` | Criar (7 ADRs) |
| `docs/adrs/README.md` | Atualizar (alinhado ao formato `ADR-NNN`, virar índice) |
| `docs/RFC.md` | Substituir placeholder |
| `docs/FDD.md` | Substituir placeholder |
| `docs/PRD.md` | Substituir placeholder |
| `docs/TRACKER.md` | Substituir placeholder |
| `README.md` | Substituir placeholder (doc do processo) |

Nenhum arquivo de código é alterado. Datas dos documentos: data de produção real (nov/2026).

## Etapa 1 — ADRs (7 arquivos)

Formato: header (Status: Aceito / Data / ADRs relacionados) + 7 seções MADR. Limites de concisão do prompt do professor: máx 3 opções, máx 5 referências de arquivo, sem code snippets.

| ADR | Decisão | Alternativas (reais, da transcrição) |
| --- | --- | --- |
| 001 | Outbox no MySQL, insert na mesma transação do `changeStatus` | Síncrono no service [09:04]; Redis Streams [09:07] |
| 002 | Retry 5 tentativas, backoff 1m/5m/30m/2h/12h, DLQ em tabela `webhook_dead_letter` separada | Retry indefinido [09:15]; 3 tentativas [09:16]; DLQ como status "failed" na própria outbox [09:17] |
| 003 | HMAC-SHA256 no corpo, secret por endpoint, rotação com grace 24h | Secret global da plataforma [09:21]; alternativa plausível sem assinatura |
| 004 | At-least-once com `X-Event-Id` (UUID) para dedup no cliente | Exactly-once [09:25] |
| 005 | Worker em processo separado (`src/worker.ts` a criar, `npm run worker`), polling 2s | Worker no mesmo processo da API [09:11]; trigger de banco [09:09] |
| 006 | Reuso dos padrões existentes: AppError, Pino, error middleware, módulos, prefixo `WEBHOOK_` | Abordagem própria/paralela para o módulo. Referencia código explicitamente (5 paths reais) |
| 007 | Snapshot do payload renderizado na inserção + UUID como id da outbox [09:51-09:52] | Guardar só order_id e renderizar no envio [09:51] |

Cobertura: 6/6 decisões principais + 1 secundária (enunciado pede 5-8 ADRs, ≥5 das 6 principais).

**Checkpoint 1**: revisão do usuário.

## Etapa 2 — RFC

`docs/RFC.md`, conciso (2-4 páginas), status "Proposto". Seções: Metadados (autora Larissa; revisores Marcos, Bruno, Diego, Sofia; data), Resumo executivo (TL;DR), Contexto e problema, Proposta técnica (visão geral: outbox + worker polling + retry/DLQ + HMAC + at-least-once + CRUD de configuração), Alternativas consideradas (≥2 com trade-off: síncrono, Redis Streams, trigger de banco, exactly-once, render-no-envio), Questões em aberto (≥2: rate limiting [09:39], permissões do CRUD [09:37], escala multi-worker [09:13]), Impacto e riscos, Decisões relacionadas (links para os 7 ADRs).

Regra de altura: sem detalhe de implementação do FDD (sem payloads de exemplo, sem matriz de erros).

**Checkpoint 2**: revisão do usuário.

## Etapa 3 — FDD

`docs/FDD.md` no esqueleto do `prompts/fdd-generator.md` (10 seções numeradas) + seção adicional obrigatória "Integração com o sistema existente".

- **Seção 4 (Fluxos)**: criação do evento na outbox dentro do `changeStatus` (via `publishWebhookEvent(tx, order, fromStatus, toStatus)` [09:41]), processamento pelo worker (polling 2s, batch de pendentes), retry com backoff, DLQ, replay admin. Diagramas mermaid embutidos (regras de `prompts/mermaid-diagram-generator.md` aplicadas inline): sequência do fluxo principal, fluxograma do worker com retry/DLQ, ER da outbox/deliveries/dead_letter.
- **Seção 5 (Contratos públicos)**: 7 endpoints com request/response JSON e status codes: `POST /api/v1/webhooks`, `GET /api/v1/webhooks`, `PATCH /api/v1/webhooks/:id`, `DELETE /api/v1/webhooks/:id`, rotação de secret (`POST /api/v1/webhooks/:id/rotate-secret`), `GET /api/v1/webhooks/:id/deliveries`, `POST /api/v1/admin/webhooks/dead-letter/:id/replay`. Contrato do envio: payload (`event_id`, `event_type: "order.status_changed"`, `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`) + headers (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json`) [09:43-09:45].
- **Seção 6 (Erros/exceções/fallback)**: matriz com códigos `WEBHOOK_*` (citados: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` [09:28] + derivados no mesmo padrão referenciados nos requisitos: URL não https [09:23], payload > 64KB [09:23-09:24], falha de entrega, DLQ inexistente no replay). Resiliência: timeout 10s [09:42], retry 5x com backoff, DLQ, replay. Invariantes.
- **Seção "Integração com o sistema existente"**: ≥4 paths reais com descrição de integração: `src/modules/orders/order.service.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/shared/logger/index.ts`, `src/routes/index.ts`/`src/app.ts`, `src/config/database.ts`, `prisma/schema.prisma`.
- **Seção 7 (Observabilidade)**: métricas (sucesso/falha de entrega, latência, profundidade da fila), logs estruturados Pino (event_id, webhook_id, tentativa, status), tracing (correlação request → outbox → delivery).

**Checkpoint 3**: revisão do usuário.

## Etapa 4 — PRD

`docs/PRD.md` no esqueleto do `prompts/prd-generator.md`. Conteúdo: resumo/contexto (3 clientes B2B, polling no `GET /orders`, risco de churn Atlas) [09:00-09:02]; objetivos com métrica/meta (notificação < 10s do commit à entrega [09:02] [09:10]); escopo incluso x fora (≥2; tenho 6); 10 FRs com fluxo principal, exceções, erros e prioridade; NFRs (latência, TLS/https [09:23], payload ≤ 64KB [09:24], timeout 10s [09:42], HMAC, autorização); decisões e trade-offs em altura de produto; dependências (revisão de segurança da Sofia [09:46], portal do desenvolvedor [09:26]); riscos ≥2 com probabilidade/impacto/mitigação/contingência; critérios de aceitação; estratégia de testes.

**Checkpoint 4**: revisão do usuário.

## Etapa 5 — TRACKER

`docs/TRACKER.md`, tabela no formato obrigatório (`ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização`), construída por varredura dos documentos prontos. Convenção de IDs: `PRD-FR-01`, `PRD-NFR-01`, `RFC-ALT-01`, `RFC-OPEN-01`, `FDD-CONTRATO-01`, `FDD-ERRO-01`, `FDD-FLUXO-01`, `ADR-001`...

Metas: ≥80% dos itens identificáveis cobertos; ≥70% das linhas com Fonte = `TRANSCRICAO` com timestamp válido `[hh:mm] Nome`; ≥5 linhas com Fonte = `CODIGO` com path real.

**Checkpoint 5**: revisão do usuário.

## Etapa 6 — README do processo

Substituir o placeholder por: Sobre o desafio (1-2 parágrafos), Ferramentas de IA utilizadas, Workflow adotado, Prompts customizados (≥2 em blocos de código), Iterações e ajustes (≥2 correções concretas, do log de iterações), Como navegar a entrega. Insumos coletados na seção 13 de `WORK_ANALYSIS.md`.

**Checkpoint 6 (final)**: revisão do usuário.

## Etapa 7 — Revisão final e verificação

1. Varredura item por item da checklist de critérios de aceite do INSTRUCTIONS.MD.
2. Verificações manuais via bash: paths citados existem; timestamps do TRACKER existem em `TRANSCRICAO.md`; contagens (5-8 ADRs, ≥4 endpoints, ≥4 paths de integração, ≥8 FRs, ≥2 fora de escopo, ≥2 alternativas, ≥2 questões em aberto); prefixo `WEBHOOK_`; percentuais do TRACKER; nenhum item descartado (email, dashboard, rate limiting, arquivamento, multi-worker, inbound) como requisito.
3. Nada é commitado ou pushado sem pedido explícito.

## Regras transversais

- Anti-invenção: item sem fonte (timestamp ou path) não entra; se não preenche a coluna Localização do TRACKER, remove ou ajusta.
- Altura por documento (tabela do INSTRUCTIONS): sem duplicação entre PRD/RFC/FDD/ADRs.
- PT-BR; termos técnicos, códigos de erro, headers e nomes de tecnologia em inglês; sem travessão "—" como conectivo.
- Código-fonte, `TRANSCRICAO.md` e `INSTRUCTIONS.MD` permanecem intocados.

## Status de execução

| Etapa | Status |
| --- | --- |
| 1 — ADRs (7 arquivos) + índice | Concluída (2026-11-22), checkpoint 1 apresentado |
| 2 — RFC | Concluída (2026-11-22), checkpoint 2 apresentado |
| 3 — FDD | Concluída (2026-11-22), checkpoint 3 apresentado |
| 4 — PRD | Concluída (2026-11-22), checkpoint 4 apresentado |
| 5 — TRACKER | Concluída (2026-11-22), checkpoint 5 apresentado |
| 6 — README do processo | Concluída (2026-11-22), checkpoint 6 apresentado |
| 7 — Revisão final e verificação | Concluída (2026-11-22) |
