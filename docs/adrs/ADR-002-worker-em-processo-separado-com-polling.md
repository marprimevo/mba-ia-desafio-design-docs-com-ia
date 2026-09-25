# ADR-002: Worker em processo separado com polling

## Status

Aceito

## Contexto

A outbox só produz efeito se algum processo ler `webhook_outbox` e chamar o cliente. Esse processo precisa sobreviver a um restart da API e usar o mesmo banco.

O requisito de produto é entrega percebida abaixo de 10 segundos. Não há necessidade, nesta fase, de push do banco para a aplicação.

## Decisão

O worker roda em processo Node separado da API.

- Entry point novo: `src/worker.ts`, no mesmo espírito de `src/server.ts`.
- Script: `npm run worker`.
- A lógica de leitura e envio fica em `src/modules/webhooks/webhook.processor.ts`.
- Conexão: novo `PrismaClient` via `createPrismaClient()` em `src/config/database.ts`, mesma `DATABASE_URL`. PrismaClient é por processo; não se reaproveita a instância da API.
- Laço de polling a cada 2 segundos. A cada ciclo, busca os eventos pendentes mais antigos, processa o lote e marca o resultado.
- Um único worker. A ordem de entrega de um mesmo pedido segue `created_at` da outbox.

A latência extra no pior caso é de 2 segundos e foi aceita.

## Alternativas Consideradas

### Worker dentro do processo da API

Reiniciar ou derrubar a API derrubaria também o disparo. Deploy e crash de HTTP passariam a significar atraso de notificação.

Trade-off do descarte: um processo só simplifica o deploy local, em troca de acoplar a vida do worker à vida do servidor HTTP.

### Trigger ou listener nativo do MySQL

Descartada no [ADR-001](ADR-001-outbox-no-mysql.md). MySQL não notifica processo externo. Polling de 2 segundos cobre o orçamento de 10 segundos sem esse mecanismo.

Trade-off do descarte: menos atraso ocioso de até 2 segundos, em troca de infra ou de um sinal improvisado a partir do banco.

## Consequências

### Positivas

- Restart da API não perde o worker.
- O ciclo de 2 segundos é simples de operar e de testar.
- Mesma stack (Node, TypeScript, Prisma, MySQL) e outro processo.

### Negativas

- Passa a existir um segundo processo para subir, observar e desligar (SIGINT/SIGTERM, no padrão de `src/server.ts`).
- Com um único worker, o throughput é o de um laço. Rate limit de saída e escala horizontal não foram decididos.
- Evento deixado em `processando` se o processo morrer no meio do HTTP não tem varredura de recuperação definida na reunião. A entrega at-least-once e o `X-Event-Id` cobrem o reenvio; a retomada da linha presa é risco operacional, não uma segunda política de retry.
