# Architectural Decision Records

Decisões fechadas na call do sistema de webhooks de notificação de pedidos. O formato é MADR (status, contexto, decisão, alternativas, consequências).

| ADR | Decisão |
| --- | --- |
| [ADR-001](ADR-001-outbox-no-mysql.md) | Outbox no MySQL, na mesma transação da mudança de status |
| [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) | Worker em processo separado, polling de 2 segundos |
| [ADR-003](ADR-003-retry-com-backoff-e-dlq.md) | Backoff 1 min / 5 min / 30 min / 2 h / 12 h e DLQ em tabela separada |
| [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256, secret por endpoint, rotação com graça de 24 h |
| [ADR-005](ADR-005-garantia-at-least-once-com-x-event-id.md) | At-least-once com `X-Event-Id` |
| [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md) | Reuso de módulo, `AppError`, Pino, Zod, `requireRole` e UUID |
| [ADR-007](ADR-007-snapshot-de-payload-na-insercao.md) | Payload renderizado na inserção da outbox |

A proposta que amarra essas decisões está em [../RFC.md](../RFC.md). O como implementar está em [../FDD.md](../FDD.md).
