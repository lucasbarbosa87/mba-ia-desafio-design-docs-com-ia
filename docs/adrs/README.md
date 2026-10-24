# Architectural Decision Records

Este diretório guarda os ADRs (Architectural Decision Records) da feature **Sistema de Webhooks de Notificação de Pedidos**. Cada decisão fica em um arquivo próprio, nomeado `ADR-NNN-titulo-em-kebab-case.md`, no formato MADR: Status, Contexto, Decisão, Alternativas Consideradas e Consequências.

| ADR | Decisão | Status |
| --- | --- | --- |
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL existente | Aceito |
| [ADR-002](ADR-002-worker-separado-em-polling.md) | Worker em processo separado com polling de 2 s | Aceito |
| [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) | Retry com backoff exponencial e DLQ em tabela separada | Aceito |
| [ADR-004](ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação de 24h | Aceito |
| [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md) | Entrega at-least-once com `X-Event-Id` | Aceito |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) | Reuso dos padrões existentes do projeto | Aceito |
| [ADR-007](ADR-007-payload-snapshot-enxuto-na-insercao.md) | Payload enxuto como snapshot na inserção, com filtro na inserção | Aceito |
