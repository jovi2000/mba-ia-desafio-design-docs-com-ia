# Architectural Decision Records

Este diretório armazena os ADRs (Architecture Decision Records) da feature Sistema de Webhooks de Notificação de Pedidos. Cada decisão é registrada em arquivo individual, nomeado no formato `ADR-NNN-titulo-em-kebab-case.md`.

Formato: MADR, com header (Status, Data, ADRs relacionados) e 7 seções: Contexto e Enunciado do Problema, Direcionadores da Decisão, Opções Consideradas, Resultado da Decisão, Prós e Contras das Opções, Consequências e Referências. Toda decisão é rastreável à transcrição da reunião (`TRANSCRICAO.md`, citada com timestamp) ou ao código-fonte existente.

## Índice

| ADR | Decisão | Status |
| --- | --- | --- |
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL para eventos de mudança de status | Aceito |
| [ADR-002](ADR-002-retry-com-backoff-e-dlq.md) | Retry com backoff exponencial (1m/5m/30m/2h/12h, 5 tentativas) e DLQ persistida em tabela separada | Aceito |
| [ADR-003](ADR-003-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h | Aceito |
| [ADR-004](ADR-004-garantia-at-least-once-com-x-event-id.md) | Entrega at-least-once com X-Event-Id para deduplicação no cliente | Aceito |
| [ADR-005](ADR-005-worker-separado-em-polling.md) | Worker em processo separado consumindo a outbox por polling de 2s | Aceito |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes.md) | Reuso dos padrões existentes do projeto no módulo de webhooks | Aceito |
| [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md) | Snapshot do payload do evento na inserção da outbox | Aceito |
