# PRD: Sistema de Webhooks de Notificação de Pedidos

## Resumo e contexto da feature

O OMS em produção gerencia clientes, produtos e pedidos. O ciclo de vida do pedido tem máquina de estados, estoque transacional e histórico de status. Não há notificação para fora.

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — pediram aviso quando o status dos pedidos deles muda. A decisão de produto e de arquitetura já foi tomada numa call técnica (Larissa, Marcos, Bruno, Diego e Sofia) e está só na transcrição. Esta feature é o sistema de webhooks de saída que materializa essa decisão.

O fluxo é outbound: a plataforma chama o endpoint do cliente. O cliente não chama a plataforma por webhook.

## Problema e motivação

Hoje o integrador descobre mudança de status consultando `GET /orders` de tempos em tempos. A integração fica lenta e cara para ele, e o dado chega tarde.

A Atlas sinalizou que, sem isso até o fim do trimestre, pode ir para o concorrente. Na call, o prazo concreto pedido foi fim de novembro. Para esses clientes, qualquer atraso abaixo de 10 segundos já conta como tempo real. O que não pode acontecer é a informação ficar parada até alguém atualizar a consulta na mão.

## Público-alvo e cenários de uso

**Público**

- Integradores B2B dos clientes citados, que operam um endpoint próprio.
- Usuários do OMS (roles `ADMIN` e `OPERATOR` já existentes) que cadastram essa configuração pela API, em nome de um `customer_id`.
- `ADMIN`, único papel que pode reprocessar falha permanente.
- Marcos, no portal de desenvolvedores, para explicar deduplicação e verificação de assinatura. O portal não é software deste repositório.

**Cenários**

1. A Atlas cadastra um endpoint `https` pedindo só `SHIPPED` e `DELIVERED`. Uma mudança para `PROCESSING` não gera evento. Uma mudança para `SHIPPED` gera.
2. O operador muda o status. Em menos de 10 segundos no caminho feliz, o endpoint da Atlas recebe um POST assinado, com id estável do evento.
3. O endpoint está em manutenção. A plataforma tenta de novo na progressão combinada e, esgotadas as tentativas, guarda o evento para um admin recolocar na fila.
4. A secret apareceu em log do cliente. Ele pede outra pela API e tem 24 horas com as duas válidas.

## Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta |
| --- | --- | --- | --- |
| PRD-OBJ-01 | O integrador deixa de precisar adivinhar o status por polling | Tempo entre o commit da mudança de status e o início da primeira tentativa HTTP, no caminho sem retry | Menos de 10 segundos |
| PRD-OBJ-02 | A Atlas recebe a capacidade no prazo que Marcos levou para a call | Entrega da feature, incluindo a revisão de segurança | Fim de novembro, em três sprints |

O polling de 2 segundos foi aceito como compatível com a meta de 10 segundos. Os 10 segundos de timeout do HTTP do worker são outra coisa: é quanto se espera a resposta do cliente, não o orçamento de ponta a ponta.

## Escopo

### Incluso

- Cadastro, edição, remoção e listagem de endpoints de webhook por cliente.
- Secret gerada pela plataforma, rotação com graça de 24 horas, histórico das últimas 100 entregas.
- Filtro por status, aplicado antes de gravar o evento.
- Publicação do evento na mesma transação de `changeStatus`.
- Worker separado, polling de 2 segundos, retry, DLQ e replay admin.
- Assinatura HMAC-SHA256, HTTPS obrigatório, teto de 64 KB, timeout de 10 segundos, entrega at-least-once com `X-Event-Id`.

### Fora de escopo

- **E-mail quando o webhook falha.** Marcos pediu aviso ao cliente depois de falhas seguidas. Larissa tirou desta fase. Pode voltar depois que houver medida de impacto.
- **Painel visual.** O cliente não ganha tela para ver os webhooks. Só a API. Painel é projeto do time de frontend.
- **Arquivar linhas entregues** depois de cerca de 30 dias. Diego mencionou e marcou como fora desta feature.
- **Webhook de entrada.** O cliente não envia evento para o OMS.
- **Redis, HTTP síncrono na troca de status, trigger de banco, exactly-once e vários workers.** Foram descartados ou adiados. O motivo está no RFC e nos ADRs.
- **Rate limit de saída.** Não entra agora; fica em observação (questão em aberto do RFC).
- **Notificação na criação do pedido.** O `PENDING` inicial nasce em `OrderService.create`, e o gancho combinado é só `changeStatus`. A máquina em `src/modules/orders/order.status.ts` também não transiciona para `PENDING`.

## Requisitos funcionais

| ID | Requisito |
| --- | --- |
| PRD-FR-01 | O cliente da API cadastra um webhook com `POST`, informando URL e a lista de status que quer receber. |
| PRD-FR-02 | A secret é gerada pela plataforma e devolvida na criação. O integrador não escolhe a secret no cadastro. |
| PRD-FR-03 | O `customer_id` vai no body ou no path. Não sai do JWT. O JWT continua sendo o do usuário operador. |
| PRD-FR-04 | Dá para editar o webhook com `PATCH` e remover com `DELETE`. |
| PRD-FR-05 | Dá para listar os webhooks de um customer com `GET`. |
| PRD-FR-06 | Cada endpoint escolhe os status que escuta. Se nenhum endpoint daquele customer quer o status novo, o sistema não insere na outbox. |
| PRD-FR-07 | `GET /webhooks/:id/deliveries` devolve as últimas 100 entregas, com sucesso ou falha, payload, response e tempo de resposta. |
| PRD-FR-08 | `POST /admin/webhooks/dead-letter/:id/replay` recoloca o evento da DLQ como pendente. |
| PRD-FR-09 | O replay exige role `ADMIN` e registra quem executou, para auditoria. |
| PRD-FR-10 | O CRUD de configuração, nesta fase, aceita qualquer role autenticada. |
| PRD-FR-11 | Existe endpoint para girar a secret. A anterior fica válida por 24 horas. |
| PRD-FR-12 | URL que não seja `https` é recusada na validação. |
| PRD-FR-13 | Cada POST ao cliente leva HMAC-SHA256 do corpo, com a secret daquele endpoint, no header `X-Signature`. |
| PRD-FR-14 | A entrega é at-least-once. O header `X-Event-Id` é um UUID estável, criado quando o evento entra na outbox, para o cliente deduplicar. |
| PRD-FR-15 | O JSON leva `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents`. Não leva items. |
| PRD-FR-16 | Os headers de saída são `X-Event-Id`, `X-Signature`, `X-Timestamp` (instante do envio), `X-Webhook-Id` e `Content-Type: application/json`. |
| PRD-FR-17 | A publicação ocorre dentro da transação de `changeStatus`. Se a outbox não gravar, o status não muda. |

## Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| PRD-NFR-01 | No caminho feliz, a primeira tentativa sai em menos de 10 segundos após o commit. |
| PRD-NFR-02 | O worker é outro processo e consulta a outbox a cada 2 segundos. A espera máxima do polling, 2 segundos, foi aceita. |
| PRD-NFR-03 | A chamada HTTP ao cliente estoura em 10 segundos e esse estouro é falha, com retry. |
| PRD-NFR-04 | Payload acima de 64 KB não é enviado nem truncado. É erro. |
| PRD-NFR-05 | Há no máximo um worker. A ordem dos eventos de um mesmo pedido segue `created_at`. Não há garantia de ordem global entre workers, porque não haverá vários. |
| PRD-NFR-06 | Retry: espera de 1 min, 5 min, 30 min, 2 h e 12 h depois das falhas, e então DLQ. A janela citada na call é de quase 15 horas entre a primeira falha e a última tentativa. Três tentativas e retry infinito foram recusados. |
| PRD-NFR-07 | A feature reutiliza módulo por domínio, `AppError`, códigos com prefixo, Pino, Zod, error middleware e `requireRole`. Ids são UUID. |
| PRD-NFR-08 | O payload gravado é o snapshot da hora da mudança de status, não uma releitura do pedido na hora do envio. |

## Decisões e trade-offs principais

- **Outbox no MySQL em vez de HTTP na transação.** Ganha consistência e isola o cliente lento. Perde o “na hora” e aceita até 2 segundos de polling.
- **MySQL em vez de Redis.** Ganha zero infra nova. Perde um canal feito para fila.
- **At-least-once em vez de exactly-once.** Ganha um desenho operável por um lado só. O cliente precisa deduplicar.
- **Cinco degraus de backoff em vez de três.** Cobre manutenção de horas. O evento demora mais para chegar na DLQ.
- **DLQ em tabela separada.** A outbox viva fica legível. Há mais uma tabela.
- **Secret por endpoint e graça de 24 horas.** Um vazamento não abre todos os clientes, e a troca não corta a integração na hora. A secret antiga ainda assina durante um dia.
- **Snapshot na inserção.** O evento continua verdadeiro horas depois. A outbox guarda o JSON, não só o id do pedido.

## Dependências

- MySQL e Prisma já usados pela API (`prisma/schema.prisma`, `src/config/database.ts`).
- JWT, `authenticate` e `requireRole` já existentes.
- Máquina de status e `changeStatus` como único gancho de publicação.
- Revisão de segurança da Sofia, mínimo de dois dias úteis, antes do deploy, com olhar específico para HMAC e geração de secret.
- Os três clientes precisam implementar o receptor (HTTPS, HMAC, dedup). Isso é dependência externa, não código deste repositório.
- Documentação no portal, responsabilidade do Marcos, para o contrato de dedup.

## Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| PRD-RISK-01 | Secret vaza em log do cliente ou da própria API. Já aconteceu com cliente. | Média | Alto | Secret por endpoint, rotação com graça de 24 h, e redact de `secret` no Pino. Durante as 24 h a chave antiga ainda é válida: é o custo da migração. |
| PRD-RISK-02 | Endpoint do cliente fica indisponível por horas (já houve manutenção de duas horas) ou responde devagar. | Média | Médio | Timeout de 10 s, backoff até a janela de quase 15 h, depois DLQ e replay admin. Três tentativas foi rejeitado justamente por ser curto demais. |
| PRD-RISK-03 | O mesmo evento chega duas vezes e o cliente aplica o efeito de novo. | Média | Médio | `X-Event-Id` estável e destaque no portal. A plataforma não promete exactly-once. |
| PRD-RISK-04 | Qualquer usuário autenticado cadastra webhook para qualquer `customer_id`, porque o papel do CRUD ficou frouxo de propósito. | Alta | Alto | Replay já é `ADMIN` e auditado. Endurecer o CRUD é questão em aberto; até lá o risco fica explícito e não vira regra silenciosa. |

## Critérios de aceitação

1. Mudança de status confirmada gera linha na outbox para cada endpoint ativo daquele customer que assina o status de destino, dentro da mesma transação. Rollback não deixa linha.
2. Status que ninguém assina não gera linha.
3. Criar pedido (`PENDING` inicial) não gera webhook.
4. No caminho feliz, a primeira tentativa HTTP começa em menos de 10 segundos após o commit.
5. URL `http` é recusada. URL `https` válida é aceita. A resposta de criação inclui a secret gerada.
6. O POST ao cliente contém o JSON combinado, sem items, e os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`.
7. Falha de rede, timeout de 10 s ou HTTP não-2xx reagenda na progressão. Esgotada a progressão, o evento está na DLQ com payload, motivo e timestamp.
8. Replay sem role `ADMIN` é negado. Replay com `ADMIN` volta o evento para pendente e deixa rastro de quem fez.
9. Girar a secret devolve a nova e mantém a anterior utilizável por 24 horas.
10. Payload acima de 64 KB não é enviado.

## Estratégia de testes e validação

A call reservou testes de ponta a ponta na estimativa de três sprints e uma revisão humana de segurança no fim.

- Testes automatizados no Vitest já usado pelo projeto (`tests/orders.test.ts` é o padrão de HTTP com a API de pedidos): contrato dos endpoints, rejeição de URL não-HTTPS, papel `ADMIN` no replay, e a transação de `changeStatus` com e sem rollback da outbox.
- Testes do processor com servidor HTTP falso: timeout, status não-2xx, agenda de backoff, id estável entre tentativas, DLQ, e recusa de payload acima de 64 KB.
- Checklist manual da Sofia sobre geração de secret, HMAC e o que é redactado no log, nos dois dias úteis anteriores ao deploy.
- Validação com os três clientes fica fora do ambiente deste repositório: depende do endpoint deles e do texto do portal.
