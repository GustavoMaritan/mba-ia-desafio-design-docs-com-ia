# Architectural Decision Records

Decisões arquiteturais do Sistema de Webhooks de Notificação de Pedidos, uma por arquivo, no formato `ADR-NNN-titulo-em-kebab-case.md` (variante do MADR).

| ADR | Decisão | Status |
|---|---|---|
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL | Aceito |
| [ADR-002](ADR-002-worker-separado-em-polling.md) | Worker em processo separado, polling de 2s | Aceito |
| [ADR-003](ADR-003-retry-com-backoff-e-dlq.md) | Retry com backoff 1m/5m/30m/2h/12h e DLQ em tabela separada | Aceito |
| [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h | Aceito |
| [ADR-005](ADR-005-at-least-once-com-x-event-id.md) | Entrega at-least-once com deduplicação por `X-Event-Id` | Aceito |
| [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md) | Reuso dos padrões existentes do projeto | Aceito |
| [ADR-007](ADR-007-payload-snapshot-e-filtro-na-insercao.md) | Payload em snapshot e filtro de eventos na inserção | Aceito |
