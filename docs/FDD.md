# FDD: Sistema de Webhooks de Notificação de Pedidos

## Contexto e motivação técnica

`OrderService.changeStatus` (`src/modules/orders/order.service.ts`) abre `prisma.$transaction`, valida a transição com `canTransition` (`src/modules/orders/order.status.ts`), debita ou devolve estoque, atualiza `orders.status` e insere `order_status_history`. Não há escrita de evento nem cliente HTTP.

A call pediu que essa transação também grave o fato “o status mudou”, e que outro processo faça o POST. Se a gravação do fato falhar, o status não pode ficar mudado. Se o POST falhar, o status não pode voltar atrás.

Este documento é o contrato de implementação. O porquê está no [RFC](RFC.md) e nos [ADRs](adrs/README.md).

## Objetivos técnicos

- Publicar o snapshot na mesma transação de `changeStatus`, via `publishWebhookEvent(tx, order, fromStatus, toStatus)`.
- Entregar em processo separado, polling de 2 segundos, uma tentativa inicial em menos de 10 segundos no caminho feliz.
- Repetir falha transitória na progressão 1 min / 5 min / 30 min / 2 h / 12 h e só então mover para `webhook_dead_letter`.
- Assinar o body com HMAC-SHA256 da secret do endpoint e manter o mesmo `X-Event-Id` em todas as tentativas e no replay.
- Expor CRUD, histórico, rotação e replay no prefixo `/api/v1`, com os mesmos mecanismos de auth, Zod e `AppError`.

## Escopo e exclusões

Entra o módulo `src/modules/webhooks/`, as tabelas abaixo, o entry `src/worker.ts` e a chamada nova dentro de `changeStatus`.

Não entra:

- e-mail de falha, painel, rate limit de saída, arquivamento de entregues (~30 dias), webhook de entrada, Redis, vários workers;
- evento na criação do pedido (`OrderService.create` grava `PENDING` sem passar por `changeStatus`);
- items no payload;
- exactly-once.

## Modelo de dados

Nomes fechados na call: `webhook_outbox` e `webhook_dead_letter`. A tabela de cadastro não ganhou nome; este FDD usa `webhook_endpoints`. O histórico de tentativas, pedido pelo `GET .../deliveries` e também sem nome na call, usa `webhook_deliveries` para não guardar corpo de resposta dentro da outbox.

Ids UUID, `@default(uuid())`, como o restante de `prisma/schema.prisma`.

### `webhook_endpoints`

| Coluna | Papel |
| --- | --- |
| `id` | UUID. Vai no `X-Webhook-Id`. |
| `customer_id` | FK para `customers.id`. |
| `url` | HTTPS. |
| `secret` | Secret vigente. Nunca volta em GET/lista. |
| `previous_secret` | Secret anterior, nula se não houve rotação. |
| `previous_secret_expires_at` | Agora + 24 h no momento da rotação. |
| `active` | Inativo não recebe evento novo. |
| `subscribed_statuses` | Lista de `OrderStatus`. |
| `created_at`, `updated_at` | Auditoria mínima, no padrão dos outros models. |

### `webhook_outbox`

| Coluna | Papel |
| --- | --- |
| `id` | UUID. É o `event_id` e o `X-Event-Id`. |
| `webhook_endpoint_id` | Endpoint de destino. |
| `order_id` | Pedido. Não é FK obrigatória para releitura: o fato está no payload. |
| `status` | `PENDING`, `PROCESSING`, `FAILED`, `DELIVERED`. Na call: pendente, processando, falhou, entregue. |
| `payload` | JSON snapshot. |
| `attempt_count` | Chamadas HTTP já feitas. |
| `next_attempt_at` | Quando o worker pode pegar de novo. |
| `last_error` | Texto curto da última falha. |
| `created_at` | Ordem de entrega. Índice junto com `status`. |

Índice em `(status, created_at)`, como Diego pediu (status e `created_at`).

### `webhook_deliveries`

Uma linha por tentativa HTTP: `id`, `outbox_id`, `webhook_endpoint_id`, `attempt_number`, `request_payload`, `response_status` nulo em timeout ou erro de rede, `response_body`, `duration_ms`, `success`, `error_code`, `created_at`.

O que se guarda da tentativa é o que Marcos pediu no histórico: sucesso ou falha, payload, response e tempo de resposta. O teto de 64 KB continua valendo só para o payload de saída, que não se trunca.

### `webhook_dead_letter`

| Coluna | Papel |
| --- | --- |
| `id` | UUID do registro de DLQ, usado no replay. |
| `outbox_id` | Mesmo UUID do evento (`X-Event-Id`). |
| `webhook_endpoint_id`, `order_id` | Contexto. |
| `payload` | Cópia do snapshot. |
| `failure_reason` | Motivo. |
| `failed_at` | Timestamp. |
| `replayed_at`, `replayed_by_user_id` | Preenchidos no replay. O user id é o `req.user.id` do admin. |

## Fluxos detalhados

### Criação do evento na outbox

Dentro do callback já existente de `this.prisma.$transaction` em `changeStatus`, depois de `order.update` e `orderStatusHistory.create`:

1. Carregar endpoints `active = true` com `customer_id` do pedido e `subscribed_statuses` contendo `toStatus`. A leitura usa o mesmo `tx`.
2. Se a lista vier vazia, não inserir e seguir. A transação de status commita.
3. Para cada endpoint, inserir uma linha `PENDING`, `attempt_count = 0`, `next_attempt_at = now`, `id` UUID novo, `payload` já renderizado.

Snapshot:

```json
{
  "event_id": "6f1c2a40-9c2e-4b1a-8f0a-1d2e3f4a5b6c",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-18T13:04:01.000Z",
  "order_id": "8b2d1c10-1111-4222-8333-444455556666",
  "order_number": "ORD-000123",
  "from_status": "PAID",
  "to_status": "PROCESSING",
  "customer_id": "aa11bb22-cc33-4d44-8e55-ff6677889900",
  "total_cents": 15990
}
```

`timestamp` é o instante da mudança, não o do envio. `order_number` segue o formato já produzido por `reserveOrderNumber` (`ORD-` + seis dígitos). `total_cents` é o valor do pedido nessa transação. Sem `items`.

Se qualquer insert lançar, a transação inteira reverte, inclusive estoque e status. É o requisito “não pode ter status mudado e evento ausente”.

`publishWebhookEvent` recebe `Prisma.TransactionClient` e vive no módulo `src/modules/webhooks/`, ao lado da lógica de envio. A call não bateu o nome do arquivo dessa função; o arquivo de processamento que Bruno ofereceu é `webhook.processor.ts` (a alternativa citada foi `webhook.worker.ts`). `OrderService` não recebe o repository de webhook no construtor.

`src/worker.ts` e a pasta `src/modules/webhooks/` não existem hoje. São os caminhos que a call pediu para criar. O restante desta seção de integração cita só arquivo que já está no repositório.

### Processamento pelo worker

`src/worker.ts` chama `createPrismaClient()`, loga com o Pino existente e a cada 2 segundos chama o processor.

Por ciclo, um lote pequeno de linhas com `status` em `PENDING` ou `FAILED` e `next_attempt_at <= now()`, `ORDER BY created_at ASC`. O tamanho do lote não foi numerado na call (“batch pequeno”). Não se fixa um número mágico aqui: o lote tem de caber no ciclo de 2 segundos para a meta de 10 segundos continuar verdadeira no caminho feliz.

Para cada linha:

1. Atualizar para `PROCESSING` se ainda estiver pegável. Com um worker só, isso evita reentrada no mesmo ciclo.
2. Medir o UTF-8 do `payload`. Se passar de 64 KB, não fazer HTTP. Ir para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`.
3. Assinar o body com HMAC-SHA256 da `secret` vigente. Enviar o POST com timeout de 10 segundos e os headers da seção de contrato de saída.
4. Gravar `webhook_deliveries`.
5. HTTP 2xx: outbox vai para `DELIVERED`.
6. Timeout, erro de rede ou status fora de 2xx: aplicar retry ou DLQ.

O worker trata SIGINT/SIGTERM como `src/server.ts`: para de aceitar ciclo novo, espera a tentativa corrente, `prisma.$disconnect()`.

### Retry

`attempt_count` incrementa a cada HTTP (ou a cada falha de timeout/rede). A primeira tentativa é imediata (`next_attempt_at` na inserção).

Depois de cada falha, se ainda houver degrau, `status = FAILED` e `next_attempt_at = now + espera`:

| Falha da tentativa | Espera até a próxima |
| --- | --- |
| 1 (envio inicial) | 1 minuto |
| 2 | 5 minutos |
| 3 | 30 minutos |
| 4 | 2 horas |
| 5 | 12 horas |
| 6 | DLQ |

São uma chamada inicial e cinco retentativas. A soma das esperas é 14h36, a “quase 15 horas” de Diego. O rótulo “5 tentativas” da tech lead nomeia essas retentativas. Três tentativas e retry infinito não se implementam.

### DLQ

Na falha da sexta chamada, ou no payload acima de 64 KB:

1. Inserir em `webhook_dead_letter` com payload, motivo e `failed_at`.
2. Remover a linha da outbox ativa, ou marcá-la de um jeito que o polling não a releia. Preferência: mover, para a outbox não acumular falha permanente. A call pediu a tabela separada justamente para a leitura da outbox ficar limpa.

Replay `POST /api/v1/admin/webhooks/dead-letter/:id/replay`:

1. Exigir `ADMIN`.
2. Se o id não existe, `WEBHOOK_DLQ_NOT_FOUND`.
3. Inserir de novo na outbox com o **mesmo** `id`/`event_id`, `status = PENDING`, `attempt_count = 0`, `next_attempt_at = now`, mesmo payload.
4. Preencher `replayed_at` e `replayed_by_user_id`.
5. Logar `userId`, `deadLetterId`, `eventId`. Sem a secret.

Zerar a contagem é o que torna “recolocar como pendente” executável. Sem isso, o próximo ciclo jogaria o evento de volta na DLQ.

## Contratos públicos

Base `/api/v1`, igual a `src/app.ts`. Autenticação: `Authorization: Bearer <jwt>`, middleware `authenticate`. Corpo de erro, igual ao error middleware:

```json
{
  "error": {
    "code": "WEBHOOK_NOT_FOUND",
    "message": "Webhook not found"
  }
}
```

Papel: CRUD e rotação e deliveries exigem usuário autenticado de qualquer role. Replay exige `requireRole('ADMIN')`.

### POST /api/v1/webhooks

Cria endpoint. A secret é gerada pela API e devolvida só nesta resposta. Tamanho e codificação do segredo não foram fixados na call; a revisão da Sofia fecha isso antes do deploy. O exemplo abaixo usa um placeholder, não um formato de prefixo.

Request:

```http
POST /api/v1/webhooks
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "customerId": "aa11bb22-cc33-4d44-8e55-ff6677889900",
  "url": "https://hooks.atlas.example/orders",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"],
  "active": true
}
```

Response `201`:

```json
{
  "id": "9ab2c3d4-e5f6-4789-a012-b3456789cdef",
  "customerId": "aa11bb22-cc33-4d44-8e55-ff6677889900",
  "url": "https://hooks.atlas.example/orders",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "<secret gerada pela API>",
  "createdAt": "2026-11-18T13:00:00.000Z"
}
```

`400` `WEBHOOK_INVALID_URL` se o scheme não for `https`. `404` `NOT_FOUND` se o customer não existe (`NotFoundError`, código já usado em `OrderService`, não um código `WEBHOOK_` paralelo). `401` se não houver Bearer.

### GET /api/v1/webhooks?customerId={uuid}

Lista os endpoints daquele customer. A secret não volta.

Response `200`:

```json
{
  "data": [
    {
      "id": "9ab2c3d4-e5f6-4789-a012-b3456789cdef",
      "customerId": "aa11bb22-cc33-4d44-8e55-ff6677889900",
      "url": "https://hooks.atlas.example/orders",
      "subscribedStatuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-11-18T13:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

O envelope é o de `paginated()` em `src/shared/http/response.ts`. `customerId` é obrigatório: a call pediu a lista de um customer, não um dump global.

### PATCH /api/v1/webhooks/:id

Edita URL, lista de status e `active`. Não edita secret.

Request:

```json
{
  "url": "https://hooks.atlas.example/v2/orders",
  "subscribedStatuses": ["DELIVERED"],
  "active": false
}
```

Response `200`: o mesmo formato do item da lista, sem secret. `404` `WEBHOOK_NOT_FOUND`. `400` `WEBHOOK_INVALID_URL` se a URL nova não for `https`.

### DELETE /api/v1/webhooks/:id

Response `204` sem corpo, no mesmo estilo de `OrderController.delete`. `404` `WEBHOOK_NOT_FOUND`.

### GET /api/v1/webhooks/:id/deliveries

Últimas 100 tentativas, da mais recente para a mais antiga. Sem paginação livre: o número 100 foi o pedido.

Response `200`:

```json
{
  "data": [
    {
      "id": "d1d1d1d1-2222-4333-8444-555566667777",
      "eventId": "6f1c2a40-9c2e-4b1a-8f0a-1d2e3f4a5b6c",
      "attemptNumber": 1,
      "success": false,
      "responseStatus": 503,
      "responseBody": "unavailable",
      "durationMs": 842,
      "errorCode": null,
      "payload": {
        "event_id": "6f1c2a40-9c2e-4b1a-8f0a-1d2e3f4a5b6c",
        "event_type": "order.status_changed",
        "timestamp": "2026-11-18T13:04:01.000Z",
        "order_id": "8b2d1c10-1111-4222-8333-444455556666",
        "order_number": "ORD-000123",
        "from_status": "PAID",
        "to_status": "PROCESSING",
        "customer_id": "aa11bb22-cc33-4d44-8e55-ff6677889900",
        "total_cents": 15990
      },
      "createdAt": "2026-11-18T13:04:03.000Z"
    }
  ]
}
```

`404` `WEBHOOK_NOT_FOUND` se o endpoint não existe.

### POST /api/v1/webhooks/:id/rotate-secret

Sem body. Gera secret nova, copia a vigente para `previous_secret`, seta `previous_secret_expires_at` para agora + 24 h.

Response `200`:

```json
{
  "id": "9ab2c3d4-e5f6-4789-a012-b3456789cdef",
  "secret": "<nova secret gerada pela API>",
  "previousSecretExpiresAt": "2026-11-19T13:10:00.000Z"
}
```

A secret anterior não volta no JSON. `404` `WEBHOOK_NOT_FOUND`.

O worker assina com a secret vigente. Quem recebe aceita assinatura da vigente ou da anterior até `previousSecretExpiresAt`. A call não pediu dois headers de assinatura.

### POST /api/v1/admin/webhooks/dead-letter/:id/replay

Sem body. `requireRole('ADMIN')`.

Response `200`:

```json
{
  "eventId": "6f1c2a40-9c2e-4b1a-8f0a-1d2e3f4a5b6c",
  "status": "PENDING",
  "attemptCount": 0
}
```

`403` `FORBIDDEN` se a role não for `ADMIN` (`ForbiddenError` já existente). `404` `WEBHOOK_DLQ_NOT_FOUND`. `401` sem token.

### POST de saída (plataforma → cliente)

Não é rota do OMS. É o contrato que o worker cumpre.

```http
POST /orders HTTP/1.1
Host: hooks.atlas.example
Content-Type: application/json
X-Event-Id: 6f1c2a40-9c2e-4b1a-8f0a-1d2e3f4a5b6c
X-Webhook-Id: 9ab2c3d4-e5f6-4789-a012-b3456789cdef
X-Timestamp: 2026-11-18T13:04:03.000Z
X-Signature: <HMAC-SHA256 do body cru>

{"event_id":"6f1c2a40-9c2e-4b1a-8f0a-1d2e3f4a5b6c","event_type":"order.status_changed","timestamp":"2026-11-18T13:04:01.000Z","order_id":"8b2d1c10-1111-4222-8333-444455556666","order_number":"ORD-000123","from_status":"PAID","to_status":"PROCESSING","customer_id":"aa11bb22-cc33-4d44-8e55-ff6677889900","total_cents":15990}
```

`X-Timestamp` é o relógio do envio. `timestamp` no JSON é o da mudança de status. A string exata de `X-Signature` (hex ou base64) fica para a revisão da Sofia; o HMAC é do body UTF-8, chave = secret do endpoint.

Resposta do cliente: qualquer 2xx encerra. O corpo da resposta só vai para o histórico.

## Matriz de erros

| Código | HTTP | Quando |
| --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | `id` de endpoint inexistente em PATCH, DELETE, deliveries ou rotação. |
| `WEBHOOK_INVALID_URL` | 400 | URL ausente de scheme `https`, na criação ou no PATCH. A checagem mora no schema Zod; o código lançado é este, o exemplo que Bruno deu, e não só `VALIDATION_ERROR`. |
| `WEBHOOK_SECRET_REQUIRED` | 422 | Tentativa de envio, ou de manter o endpoint ativo, sem secret vigente utilizável. A criação sempre gera secret; este código defende o invariante que Bruno nomeou. |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | — (worker) | Snapshot acima de 64 KB. Não envia, não trunca, não entra no backoff. Vai para a DLQ com esse motivo. |
| `WEBHOOK_DLQ_NOT_FOUND` | 404 | Replay com id que não está em `webhook_dead_letter`. |
| `WEBHOOK_DELIVERY_TIMEOUT` | — (worker) | O cliente não respondeu em 10 segundos. Conta como falha de tentativa e entra no backoff. Também pode aparecer em `webhook_deliveries.error_code`. |

Erros que continuam os códigos já existentes, de propósito:

| Código | HTTP | Quando |
| --- | --- | --- |
| `NOT_FOUND` | 404 | `customerId` inexistente na criação. Mesma classe `NotFoundError` de `OrderService.create`. |
| `VALIDATION_ERROR` | 400 | Zod: UUID inválido, `subscribedStatuses` vazia ou status fora do enum `OrderStatus`. |
| `UNAUTHORIZED` | 401 | Sem Bearer ou JWT inválido. |
| `FORBIDDEN` | 403 | Replay com role diferente de `ADMIN`. |
| `INTERNAL_SERVER_ERROR` | 500 | Qualquer outra exceção, já tratada pelo error middleware. |

## Estratégias de resiliência

| Mecanismo | Valor | Efeito |
| --- | --- | --- |
| Timeout | 10 segundos | Estouro vira `WEBHOOK_DELIVERY_TIMEOUT` e retry. Não segura a transação do pedido, porque o HTTP está no worker. |
| Retry | 1 min, 5 min, 30 min, 2 h, 12 h | Cinco retentativas depois do envio inicial. |
| Backoff | A tabela da seção Retry | Cobre a manutenção de duas horas citada na call e a janela de quase 15 horas. |
| Fallback | DLQ + replay admin | Não há e-mail, não há segundo canal. Esgotou a progressão, um `ADMIN` recoloca. Payload grande demais cai direto nesse fallback. |
| Isolamento | Processo separado | Restart da API não mata o laço. |
| Atomicidade | Mesma transação do status | Falha de insert na outbox desfaz status, histórico e estoque. |
| Duplicata | `X-Event-Id` | Crash depois do 2xx e antes do `DELIVERED` pode reenviar. O id não muda. |

Não há circuit breaker nem rate limit. Os dois não foram decididos. Rate limit está em aberto no RFC.

## Observabilidade

Não há biblioteca de métricas nem de tracing em `package.json`. A call proibiu stack nova de log e recusou infra nova. Observabilidade desta fase usa o Pino que já existe, com campos estáveis que dão para contar e para correlacionar.

**Métricas** (contadores e tempos emitidos como log estruturado, um evento por fato, para o agregador que o time já tiver em cima do stdout):

- `webhook_delivery_latency_ms`: do `created_at` da outbox até o início da primeira tentativa. Serve à meta de 10 segundos.
- `webhook_delivery_attempts_total`, com resultado `success`, `timeout`, `http_error`, `network_error`, `payload_too_large`.
- `webhook_outbox_pending`: quantidade lida no ciclo, não um gauge de produto novo.
- `webhook_dlq_total`: incrementa quando uma linha entra na DLQ.

**Logs** (Pino, `src/shared/logger/index.ts`):

- Worker: `eventId`, `webhookId`, `orderId`, `attempt`, `statusCode`, `durationMs`, `errorCode`.
- Replay: `userId`, `deadLetterId`, `eventId`, nível info.
- Erro não tratado do worker: o mesmo `logger.fatal` / `logger.error` de `src/server.ts`.
- Redact obrigatório, estendendo a lista que já existe: `*.secret`, `*.previousSecret`, `*.previous_secret`. Não logar `X-Signature`.

**Tracing:** correlação por `eventId` + `orderId` + `webhookId` em todo log do ciclo de vida do evento, do insert ao replay. No HTTP de entrada, o error middleware já guarda `req.id`; o log de replay inclui esse request id. Não se adiciona OpenTelemetry nem outro coletor: a reunião não escolheu um, e o ADR-006 manda não abrir infra paralela.

## Dependências e compatibilidade

- Node `>=20`, TypeScript, Express, Prisma, MySQL, Zod, Pino, JWT: já estão no projeto. Nenhuma dependência nova é exigida pela call. HMAC sai de `node:crypto`.
- `changeStatus` passa a escrever na outbox. Clientes atuais da API de pedidos não mudam o JSON de resposta.
- `GET /api/v1/orders/:id` continua sendo o jeito de buscar items e o resto do pedido.
- Script novo `worker` em `package.json`, ao lado de `dev` e `start`. Isso é configuração de execução, feita na implementação, não neste desafio.
- Variável nova nenhuma é obrigatória além de `DATABASE_URL`, que o worker reutiliza.

## Critérios de aceite técnicos

1. Teste de integração: `changeStatus` com endpoint assinante commita order, history, estoque e outbox juntos. Forçar falha no insert da outbox reverte os três primeiros.
2. Customer sem endpoint, ou endpoint que não lista o `toStatus`, commita o status e não cria outbox.
3. `OrderService.create` não cria outbox.
4. Worker em outro processo: subir só a API não drena a outbox.
5. Primeira tentativa no caminho feliz começa em menos de 10 segundos (polling de 2 segundos incluso).
6. Sequência de falhas respeita 1 min, 5 min, 30 min, 2 h, 12 h e na sexta falha o id está na DLQ e não volta no polling.
7. Payload > 64 KB vai para a DLQ sem HTTP.
8. `X-Event-Id` da tentativa 1 é o da tentativa 6 e o do replay.
9. Contrato HTTP da seção anterior, inclusive `204` no delete, `403` no replay de operador e secret só em `201` de criação e `200` de rotação.
10. Log de replay contém o user id e não contém a secret.

## Riscos e mitigação

| Risco | Mitigação |
| --- | --- |
| Linha presa em `PROCESSING` se o worker morre durante o HTTP. | Não foi definida varredura na call. O processo separado reduz restart junto com a API. Reenvio, quando a linha voltar a ser elegível por operação manual ou por um ciclo futuro, mantém o mesmo `X-Event-Id`. Não inventar timeout de reclaim nesta entrega. |
| CRUD sem vínculo usuário–customer. | Explícito no RFC como questão em aberto. O FDD não filtra `customerId` pelo JWT. |
| Secret em log. | Redact no Pino antes do primeiro deploy, item da revisão da Sofia. |
| Contagem “5 tentativas” lida como cinco HTTP no total. | A tabela de retry deste FDD é a normativa. Ela preserva os cinco intervalos e a janela de ~15 horas. |

## Integração com o sistema existente

Os caminhos abaixo existem no repositório. `src/worker.ts` e `src/modules/webhooks/` são criação pedida na call e ainda não estão na árvore.

### `src/modules/orders/order.service.ts`

`changeStatus` já concentra update, histórico e estoque num único `prisma.$transaction`. O ponto de extensão é o final desse callback, depois de `tx.orderStatusHistory.create` e antes do `findUnique` de retorno (ou logo após, ainda dentro do `tx`). Chamar `publishWebhookEvent(tx, order, from, to)`.

Não chamar em `create`, `delete` nem `list`. `create` escreve histórico `null → PENDING` e não foi incluído no gancho.

A função não usa `this.orders` (o repository). Usa o `tx`, no mesmo estilo de `debitStock` e `reserveOrderNumber`, que já recebem `TxClient`.

### `src/modules/orders/order.status.ts`

A lista de status assináveis é o enum `OrderStatus` que essa máquina já usa: `PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`. O Zod do webhook importa o mesmo enum, como `updateOrderStatusSchema` faz em `src/modules/orders/order.schemas.ts`.

Transição ilegal continua só com `InvalidStatusTransitionError`. Webhook não cria status novo e não publica se `canTransition` falhar, porque a publicação fica depois dessa checagem. `PENDING` como status assinado não dispara na criação, porque criação não passa aqui, e nenhuma transição leva a `PENDING`.

### `src/shared/errors/http-errors.ts`

Novas classes no mesmo arquivo, estendendo `AppError` / `NotFoundError` / `BadRequestError` / `UnprocessableEntityError`, no molde de `InsufficientStockError`. Exemplos: `WebhookNotFoundError` com código `WEBHOOK_NOT_FOUND`, `WebhookInvalidUrlError` com `WEBHOOK_INVALID_URL`. Export em `src/shared/errors/index.ts`.

`src/middlewares/error.middleware.ts` não muda: o ramo `err instanceof AppError` já devolve `error.code`. Zod continua `VALIDATION_ERROR`, salvo a URL, que o service relança como `WEBHOOK_INVALID_URL` para cumprir o código nomeado na call.

### `src/app.ts` e `src/routes/index.ts`

`buildControllers` instancia repository, service e controller de webhook como já faz com `OrderService` (`new OrderService(orderRepository, prisma)`). `Controllers` ganha o campo `webhooks`. `buildApiRouter` monta `router.use('/webhooks', ...)` e `router.use('/admin/webhooks', ...)` ao lado de `/orders`. O prefixo `/api/v1` permanece o de `buildApp`.

O replay pode viver no mesmo router com `requireRole('ADMIN')` só na rota de dead-letter, copiando `src/modules/users/user.routes.ts`, que já faz `authenticate` + `requireRole('ADMIN')`.

### `src/server.ts` e `src/config/database.ts`

`src/worker.ts` é um segundo bootstrap: não chama `buildApp` nem `listen`. Usa `createPrismaClient()` — a função já exportada — em vez do singleton `prisma` da API. Mesma `DATABASE_URL`, outra instância, como Bruno pediu (“PrismaClient é por processo”). Shutdown espelha o de `server.ts` (`SIGINT` / `SIGTERM`, `prisma.$disconnect()`, `logger`).

### `src/middlewares/auth.middleware.ts` e `src/shared/logger/index.ts`

Replay: `requireRole('ADMIN')`, já definido neste middleware. Não há role nova no enum `UserRole` de `prisma/schema.prisma`.

Logger: acrescentar paths de redact para secret. O array `redactPaths` hoje cobre authorization, cookie, password e token, e não cobre secret de webhook. Sem esse acréscimo, o incidente que Diego contou (secret em log) se repete do nosso lado.

`src/middlewares/validate.middleware.ts` valida body, query e params dos schemas novos sem alteração de comportamento. `src/shared/http/response.ts` formata a listagem.
