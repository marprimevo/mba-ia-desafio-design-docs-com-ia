# ADR-006: Reuso dos padrões do projeto

## Status

Aceito

## Contexto

O OMS já tem um jeito estável de organizar domínio, erro, validação, log e autenticação. A feature de webhook é o primeiro mecanismo de saída, mas a reunião pediu para ela parecer mais um módulo, não uma ilha.

Referências concretas no código atual:

- Módulos em `src/modules/<domínio>/` com controller, service, repository, routes e schemas. Pedidos estão em `src/modules/orders/`.
- Erros em `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts`. `InsufficientStockError` e `InvalidStatusTransitionError` carregam código estável (`INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`). `src/middlewares/error.middleware.ts` serializa qualquer `AppError` e também Zod e Prisma, sem um `if` por módulo.
- Validação Zod via `src/middlewares/validate.middleware.ts`.
- Logger Pino em `src/shared/logger/index.ts`.
- Papel de acesso em `requireRole` (`src/middlewares/auth.middleware.ts`), já usado em `src/modules/users/user.routes.ts` com `requireRole('ADMIN')`.
- Ids UUID `@default(uuid())` em `prisma/schema.prisma`.
- Prefixo de API `/api/v1` montado em `src/app.ts` e `src/routes/index.ts`.

## Decisão

O módulo novo é `src/modules/webhooks/`, com a mesma fatia dos outros domínios.

Códigos de erro do módulo usam o prefixo `WEBHOOK_` (`WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` e os demais da matriz do FDD). As classes estendem `AppError` no mesmo arquivo de erros HTTP existente. O error middleware não ganha ramo especial.

Não se introduz outro logger. O worker usa o Pino já criado.

O replay da DLQ usa `authenticate` + `requireRole('ADMIN')`. O CRUD de configuração fica em qualquer role autenticada nesta fase.

Ids das tabelas novas são UUID. Larissa confirmou isso depois do resumo, para seguir o restante do projeto, em vez de autoincremento.

A integração com pedidos não injeta o repository de webhook em `OrderService`. Entra uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `Prisma.TransactionClient` da transação já aberta em `changeStatus`.

O worker abre outro `PrismaClient` com `createPrismaClient()` de `src/config/database.ts`. Mesmo banco, outra instância, porque é outro processo.

## Alternativas Consideradas

### Injetar o repository de webhook no `OrderService`

Bruno colocou as duas opções e fechou na função que recebe `tx`. Diego concordou: não é preciso injetar o repository inteiro.

Trade-off do descarte: o service de pedidos enxergaria o contrato do repository e ficaria mais fácil de mockar por classe, em troca de acoplar `OrderService` ao módulo de webhook além da função de publicação.

### Autoincremento na outbox

Diego perguntou no fim da call. Larissa escolheu UUID porque o resto do schema é UUID.

Trade-off do descarte: id numérico é menor e ordena sozinho, em troca de quebrar o padrão de `String @id @default(uuid())` e de misturar a ordem de inserção (já coberta por `created_at`) com o tipo da chave.

## Consequências

### Positivas

- Quem já lê um módulo do OMS lê webhook sem um segundo estilo.
- `AppError` existente já vira a resposta `{ error: { code, message, details } }`.
- `requireRole('ADMIN')` já existe; o replay não inventa um segundo middleware de papel.

### Negativas

- O módulo herda limites do que já existe. Não há fila, não há tracer e o redact do Pino ainda não cobre `secret`. Isso entra como ajuste no logger atual, não como stack nova.
- CRUD autenticado sem vínculo obrigatório entre `req.user` e `customer_id` é o que a reunião aceitou “por enquanto”. O endurecimento ficou para depois e é questão em aberto do RFC.
