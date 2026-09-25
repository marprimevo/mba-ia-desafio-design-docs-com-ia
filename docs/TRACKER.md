# Tracker de rastreabilidade

Cada linha liga um item dos design docs à fala da call ou a um arquivo do OMS. `TRANSCRICAO.md` não foi alterada. Timestamp no formato da gravação.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes B2B pediram aviso de status: Atlas, MaxDistribuição e Nova Cargo | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-01 | docs/PRD.md | Problema | Hoje eles consultam GET /orders em loop; a integração fica lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-02 | docs/PRD.md | Problema | Abaixo de 10 segundos já é tempo real para esses clientes | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Primeira tentativa HTTP em menos de 10 segundos após o commit, no caminho feliz | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Entrega até fim de novembro, estimada em três sprints | TRANSCRICAO | [09:45] Marcos |
| PRD-PUB-01 | docs/PRD.md | Público | Fluxo só de saída; o cliente não envia webhook para o OMS | TRANSCRICAO | [09:02] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | POST de cadastro com URL e lista de status | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | customer_id vai no body ou no path, não no JWT do operador | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | PATCH para editar e DELETE para remover | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | GET lista os webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Filtro de status na inserção da outbox; sem assinante, não insere | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | GET /webhooks/:id/deliveries com as últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | POST /admin/webhooks/dead-letter/:id/replay recoloca como pendente | TRANSCRICAO | [09:18] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Replay só com role ADMIN e com registro de quem executou | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | CRUD de configuração aceita qualquer role autenticada nesta fase | TRANSCRICAO | [09:37] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Rotação de secret com graça de 24 horas | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | URL sem https é recusada na validação | TRANSCRICAO | [09:23] Sofia |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | HMAC-SHA256 do corpo no header X-Signature, secret por endpoint | TRANSCRICAO | [09:20] Sofia |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | At-least-once com X-Event-Id UUID estável para dedup do cliente | TRANSCRICAO | [09:25] Diego |
| PRD-FR-15 | docs/PRD.md | Requisito Funcional | Payload JSON sem items, com os campos listados por Diego | TRANSCRICAO | [09:43] Diego |
| PRD-FR-16 | docs/PRD.md | Requisito Funcional | Headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id e Content-Type | TRANSCRICAO | [09:44] Diego |
| PRD-FR-17 | docs/PRD.md | Requisito Funcional | Publicação dentro da transação de changeStatus; falha na outbox faz rollback | TRANSCRICAO | [09:40] Bruno |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Worker em outro processo, polling a cada 2 segundos | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Timeout HTTP de 10 segundos conta como falha | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Acima de 64 KB é erro, sem truncar | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Um worker; ordem por created_at; sem ordem global | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Backoff 1 min, 5 min, 30 min, 2 h e 12 h, depois DLQ | TRANSCRICAO | [09:17] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Reuso de AppError, Pino, Zod, módulo e prefixo WEBHOOK_ | TRANSCRICAO | [09:30] Larissa |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Snapshot do payload na inserção, não na hora do envio | TRANSCRICAO | [09:52] Larissa |
| PRD-OUT-01 | docs/PRD.md | Fora de escopo | E-mail de falha fica para uma fase futura | TRANSCRICAO | [09:37] Larissa |
| PRD-OUT-02 | docs/PRD.md | Fora de escopo | Dashboard visual fora; só endpoints, painel é do frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-OUT-03 | docs/PRD.md | Fora de escopo | Arquivar entregues depois de cerca de 30 dias está fora desta feature | TRANSCRICAO | [09:08] Diego |
| PRD-OUT-04 | docs/PRD.md | Fora de escopo | Webhook de entrada não faz parte do pedido | TRANSCRICAO | [09:02] Marcos |
| PRD-RISK-01 | docs/PRD.md | Risco | Vazamento de secret já ocorreu em log de cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-02 | docs/PRD.md | Risco | Manutenção de cerca de duas horas já foi vista; três retries seriam curtos | TRANSCRICAO | [09:16] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Entrega duplicada é aceita; o cliente deduplica pelo event id | TRANSCRICAO | [09:26] Larissa |
| PRD-DEP-01 | docs/PRD.md | Dependência | Sofia revisa HMAC e geração de secret por pelo menos dois dias úteis | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Portal do desenvolvedor explica dedup; não é código deste serviço | TRANSCRICAO | [09:26] Marcos |
| RFC-ALT-01 | docs/RFC.md | Trade-off | HTTP síncrono na transação foi descartado: cliente lento trava status e não há rollback | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Redis foi descartado como overengineering; outbox fica no MySQL | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Trigger MySQL não notifica processo externo; polling de 2 s permanece | TRANSCRICAO | [09:09] Diego |
| RFC-OPEN-01 | docs/RFC.md | Questão em aberto | Rate limit de saída: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| RFC-OPEN-02 | docs/RFC.md | Questão em aberto | Endurecer o papel do CRUD ficou para mais adiante | TRANSCRICAO | [09:37] Sofia |
| RFC-OPEN-03 | docs/RFC.md | Limitação | Vários workers exigiriam partição por order_id ou lock; é problema futuro | TRANSCRICAO | [09:13] Diego |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox na mesma transação SQL de orders e order_status_history | TRANSCRICAO | [09:06] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Entry src/worker.ts e script npm run worker, processo separado da API | TRANSCRICAO | [09:11] Larissa |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | DLQ em webhook_dead_letter, não como flag na outbox | TRANSCRICAO | [09:18] Diego |
| ADR-003-ALT | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Trade-off | Três tentativas descartadas; cinco degraus cobrem a janela longa | TRANSCRICAO | [09:16] Diego |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Secret única por endpoint, rotação com as duas válidas por 24 h | TRANSCRICAO | [09:22] Sofia |
| ADR-005 | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Decisão | Exactly-once foi rejeitado; o padrão de mercado citado é at-least-once | TRANSCRICAO | [09:25] Diego |
| ADR-005-HDR | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Decisão | X-Webhook-Id identifica qual cadastro disparou, quando há mais de um | TRANSCRICAO | [09:44] Sofia |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Módulo src/modules/webhooks no mesmo formato dos domínios atuais | TRANSCRICAO | [09:27] Bruno |
| ADR-006-ERR | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Códigos WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| ADR-006-FN | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | publishWebhookEvent(tx, order, fromStatus, toStatus), sem injetar repository | TRANSCRICAO | [09:41] Bruno |
| ADR-006-UUID | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Id da outbox é UUID, como o resto do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-007 | docs/adrs/ADR-007-snapshot-de-payload-na-insercao.md | Decisão | Payload renderizado na inserção para não refletir estado posterior | TRANSCRICAO | [09:52] Diego |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Worker marca pendente, processando, falhou e entregue | TRANSCRICAO | [09:08] Diego |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Índice de outbox em status e created_at; lote pequeno dos mais antigos | TRANSCRICAO | [09:08] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Latência extra de até 2 segundos no polling foi aceita | TRANSCRICAO | [09:10] Larissa |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /api/v1/webhooks cria endpoint e devolve a secret | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET lista por customer, PATCH edita, DELETE remove | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | GET de deliveries devolve payload, response e tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | Replay admin no caminho nomeado na call | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | Rotação devolve secret nova e mantém a anterior por 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-ERR-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND para id de endpoint inexistente | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL para scheme que não seja https | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-03 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE aos 64 KB, sem retry | TRANSCRICAO | [09:24] Larissa |
| FDD-ERR-04 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT aos 10 segundos de espera do cliente | TRANSCRICAO | [09:42] Diego |
| FDD-INT-01 | docs/FDD.md | Integração | O gancho é o método changeStatus, não um call HTTP no meio da transação | TRANSCRICAO | [09:40] Bruno |
| FDD-INT-02 | docs/FDD.md | Integração | PrismaClient novo no worker, mesma DATABASE_URL, outro processo | TRANSCRICAO | [09:30] Bruno |
| FDD-INT-03 | docs/FDD.md | Integração | requireRole já existente é o mecanismo do replay ADMIN | TRANSCRICAO | [09:36] Larissa |
| FDD-TEST-01 | docs/PRD.md | Teste | Testes de ponta a ponta entram na estimativa de três sprints | TRANSCRICAO | [09:46] Larissa |
| COD-ORD-01 | docs/FDD.md | Integração | changeStatus já atualiza order, history e estoque na mesma transação Prisma | CODIGO | src/modules/orders/order.service.ts |
| COD-STA-01 | docs/FDD.md | Restrição | Status assináveis são o enum e a máquina já existentes; nada transiciona para PENDING | CODIGO | src/modules/orders/order.status.ts |
| COD-ERR-01 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | AppError e classes como InsufficientStockError são o molde dos códigos WEBHOOK_ | CODIGO | src/shared/errors/http-errors.ts |
| COD-MID-01 | docs/FDD.md | Integração | error middleware já serializa AppError, Zod e Prisma; webhook não adiciona ramo | CODIGO | src/middlewares/error.middleware.ts |
| COD-AUTH-01 | docs/FDD.md | Integração | requireRole('ADMIN') já protege rota em user.routes e será reusado no replay | CODIGO | src/middlewares/auth.middleware.ts |
| COD-DB-01 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | createPrismaClient() é a fábrica que o worker chama no processo novo | CODIGO | src/config/database.ts |
| COD-SRV-01 | docs/FDD.md | Integração | src/worker.ts espelha o bootstrap e o shutdown de src/server.ts | CODIGO | src/server.ts |
| COD-APP-01 | docs/FDD.md | Integração | Rotas novas entram em /api/v1 via buildApp e buildApiRouter | CODIGO | src/app.ts |
| COD-LOG-01 | docs/FDD.md | Restrição | Logger é Pino com redact; secret de webhook ainda não está na lista e precisa entrar | CODIGO | src/shared/logger/index.ts |
| COD-PRISMA-01 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Ids novos seguem @default(uuid()) e o datasource MySQL do schema atual | CODIGO | prisma/schema.prisma |
| COD-HTTP-01 | docs/FDD.md | Contrato | Listagem usa paginated() de response.ts; delete de pedido já responde 204 | CODIGO | src/shared/http/response.ts |
| COD-TEST-01 | docs/PRD.md | Teste | Suíte atual é Vitest; tests/orders.test.ts é a referência de teste HTTP | CODIGO | tests/orders.test.ts |
