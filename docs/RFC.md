# RFC: Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| Autor | marprimevo |
| Status | Em revisão |
| Data | 2026-09-25 |
| Feature | Webhooks de notificação de mudança de status de pedido |
| Revisores | Larissa (Tech Lead), Marcos (PM), Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança) |

A call que originou esta proposta está em `TRANSCRICAO.md` (quinta-feira, 09:00–09:53). Larissa encerrou marcando uma revisão do desenho com Bruno e Diego antes de codar. Sofia reserva pelo menos dois dias úteis de revisão de segurança antes do deploy.

## Resumo executivo

Notificar clientes B2B quando o status de um pedido muda, sem chamar HTTP dentro da transação de `changeStatus`.

A proposta é outbox no MySQL, na mesma transação que já grava pedido, histórico e estoque. Um processo separado faz polling a cada 2 segundos, assina o body com HMAC-SHA256 de uma secret por endpoint e entrega at-least-once com `X-Event-Id`. Falha segue backoff de 1 min, 5 min, 30 min, 2 h e 12 h e depois uma DLQ com replay manual de `ADMIN`.

O módulo novo segue o desenho de `src/modules/`. O detalhe de endpoint, erro e sequência de implementação está no [FDD](FDD.md). As decisões fechadas estão nos ADRs linkados abaixo.

## Contexto e problema

Atlas Comercial, MaxDistribuição e Nova Cargo consultam `GET /orders` em loop para descobrir mudança de status. A integração fica lenta e cara. Para eles, abaixo de 10 segundos já é tempo real. A Atlas condicionou a permanência a uma entrega até o fim do trimestre; na call o prazo pedido foi fim de novembro.

O fluxo é só de saída. Eles recebem; não enviam webhook para o OMS.

O código hoje não tem evento, fila nem HTTP de saída. A mudança de status em `src/modules/orders/order.service.ts` é transacional e não pode passar a depender do receptor.

## Proposta técnica

Visão de arquitetura, sem o contrato campo a campo (isso é o FDD):

1. **Registro.** Usuário autenticado cadastra endpoint por `customer_id` (body ou path, não o subject do JWT), URL `https`, lista de status e flag ativo. A secret nasce na API e volta na criação. Dá para editar, remover, listar, girar a secret (graça de 24 h) e ler as últimas 100 entregas.
2. **Fato.** `changeStatus` publica, na mesma transação, um snapshot `order.status_changed` para cada endpoint ativo daquele cliente que assina o status de destino. Sem assinante, não há linha. Criação do pedido não publica: o gancho decidido é o método `changeStatus`.
3. **Saída.** `src/worker.ts` faz polling de 2 s, um processo, um `PrismaClient` próprio. Ordenação por `created_at` enquanto houver um único worker.
4. **Confiabilidade.** At-least-once, timeout de HTTP de 10 s, backoff da progressão fechada na call, DLQ em tabela própria, replay admin auditado.
5. **Proteção.** HMAC-SHA256 do body, secret por endpoint, TLS obrigatório, payload máximo de 64 KB (erro, sem truncar).
6. **Encaixe.** Módulo `src/modules/webhooks/`, `AppError` com prefixo `WEBHOOK_`, Pino, Zod, `requireRole('ADMIN')` só no replay.

Prazo estimado na call: três sprints, com a revisão da Sofia no fim.

## Alternativas consideradas

### HTTP síncrono no service de pedidos

Colocar o POST ao cliente dentro da transação de status. Trade-off que levou ao descarte: qualquer receptor lento segura a mudança de status dos outros pedidos, e receptor fora do ar não pode desfazer um status que o negócio já considera mudado. Detalhe no [ADR-001](adrs/ADR-001-outbox-no-mysql.md).

### Redis Streams ou Redis Cluster

Publicar o evento num broker em vez de numa tabela MySQL. Trade-off que levou ao descarte: o time é pequeno e não quer operar Redis só para esta feature; a outbox no MySQL que já existe preserva a atomicidade com o status sem infra nova. Detalhe no [ADR-001](adrs/ADR-001-outbox-no-mysql.md).

### Trigger no MySQL para acordar o worker

Trocar o polling de 2 s por um sinal do banco. Trade-off que levou ao descarte: MySQL não tem listener no estilo `NOTIFY/LISTEN`; a trigger só roda SQL, e avisar um processo externo vira improviso. Polling de 2 s cabe nos 10 s de produto. Detalhe no [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md).

Outras rejeições que não mudam a forma da solução, e por isso ficam nos ADRs em vez de aqui: retry de três tentativas, retry infinito, DLQ como flag na mesma tabela, secret global, exactly-once, payload montado só na hora do envio, id autoincremental.

## Questões em aberto

1. **Rate limit de saída.** Se um cliente tiver dezenas de pedidos mudando no mesmo minuto, o worker hoje dispararia uma chamada por evento. Diego levantou o ponto; Larissa registrou como observar e decidir depois. Não há cota nem fila por host nesta proposta.
2. **Papel do CRUD de configuração.** Replay exige `ADMIN`. Criar, editar, listar e apagar webhook pode ser qualquer usuário autenticado “por enquanto”. Sofia deixou o endurecimento para uma fase seguinte. Até lá, o `customer_id` do body não é amarrado ao usuário do JWT.
3. **Escala do worker.** Um processo só. Vários workers perdem a ordem por `created_at`. Partição por `order_id` ou lock pessimista foi citada como problema futuro, sem escolha. Não bloqueia esta entrega; é limitação conhecida, não desenho pendente para o primeiro release.

O encoding da string HMAC (hex ou base64) também não foi falado. Não é questão aberta da call; é lacuna para a revisão de segurança já combinada com a Sofia, e o FDD não inventa um dos dois.

## Impacto e riscos

- **Pedidos.** `changeStatus` ganha uma escrita a mais na mesma transação. Falha ao inserir na outbox faz rollback do status e do estoque. É o comportamento pedido.
- **Operação.** Segundo processo (`npm run worker`) e tabelas novas no MySQL atual. Sem Redis e sem serviço novo.
- **Clientes.** Precisam de endpoint `https`, verificação HMAC e dedup por `X-Event-Id`. Marcos documenta isso no portal. Não há painel nesta entrega.
- **Segurança.** Secret por endpoint e rotação de 24 h reduzem o raio de um vazamento e, durante a graça, a secret antiga continua válida de propósito.
- **Prazo.** Três sprints até fim de novembro, com dois dias úteis da Sofia antes do deploy. Atraso aqui é o risco comercial que a Atlas colocou na mesa.

## Decisões relacionadas

- [ADR-001: Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002: Worker em processo separado com polling](adrs/ADR-002-worker-em-processo-separado-com-polling.md)
- [ADR-003: Retry com backoff e DLQ](adrs/ADR-003-retry-com-backoff-e-dlq.md)
- [ADR-004: HMAC-SHA256 com secret por endpoint](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005: Garantia at-least-once com X-Event-Id](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)
- [ADR-006: Reuso dos padrões do projeto](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
- [ADR-007: Snapshot do payload na inserção](adrs/ADR-007-snapshot-de-payload-na-insercao.md)
