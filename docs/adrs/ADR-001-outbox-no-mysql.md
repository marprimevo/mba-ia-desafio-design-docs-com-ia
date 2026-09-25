# ADR-001: Outbox no MySQL

## Status

Aceito

## Contexto

O OMS já muda o status do pedido dentro de uma transação Prisma que atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity`. Não existe fila, broker nem cliente HTTP de notificação. Três clientes B2B pediram aviso quando o status muda, com latência percebida abaixo de 10 segundos.

Disparar HTTP dentro dessa transação acoplaria a disponibilidade do cliente à mudança de status. Se o cliente estiver lento, a transação segura lock e atrasa outros pedidos. Se estiver fora, não há rollback aceitável: o status já mudou de verdade no negócio.

A garantia necessária é outra: se a transação de status commitou, o evento existe; se deu rollback, o evento não existe.

## Decisão

Adotar o padrão outbox no MySQL já usado pela aplicação.

Na mesma transação de `OrderService.changeStatus`, depois da atualização do pedido e do histórico, o sistema insere uma linha em `webhook_outbox` para cada endpoint do cliente que assina o status de destino. A inserção recebe o client transacional (`tx`). Um worker em outro processo lê a tabela e faz o HTTP.

Se nenhum endpoint ativo do `customer_id` assina aquele status, nenhuma linha é inserida. O filtro acontece na inserção, não no envio.

Linhas entregues não são arquivadas nesta feature. A fala de arquivar depois de cerca de 30 dias ficou fora de escopo.

A tabela tem índice em status (`pendente`, `processando`, `falhou`, `entregue`) e em `created_at`. O worker lê pendentes antigos em lote pequeno.

O id da linha é UUID, no mesmo padrão de `prisma/schema.prisma`. O payload gravado é o snapshot do evento no momento da inserção.

## Alternativas Consideradas

### HTTP síncrono dentro de `changeStatus`

Bruno descreveu a transação atual como pesada. Um call HTTP no meio dela faria um cliente lento travar mudança de status de outros pedidos. Cliente fora do ar levantaria a pergunta de rollback, e a resposta da reunião foi que não se desfaz a mudança de status por falha de notificação.

Trade-off do descarte: sincrono seria menos peças móveis e latência menor no caminho feliz, em troca de acoplar o commit do pedido à rede do cliente.

### Redis Streams ou Redis Cluster

Diego tratou broker novo como overengineering para um time pequeno. Outbox no MySQL existente evita subir e operar outro sistema.

Trade-off do descarte: Redis daria consumo mais reativo e tiraria volume de polling da tabela, em troca de infra, cliente e modo de falha que o time não opera hoje.

### Trigger MySQL para acordar o worker

MySQL não tem `LISTEN/NOTIFY`. Trigger executa SQL e não avisa um processo externo sem um improviso (arquivo ou HTTP a partir do banco).

Trade-off do descarte: trigger pareceria mais “em tempo real”, em troca de um efeito colateral opaco e de continuar precisando de um processo que faz o HTTP.

## Consequências

### Positivas

- Commit do status e registro do evento são atômicos. Não há pedido atualizado sem outbox, nem outbox de uma transação revertida.
- A API deixa de depender do tempo de resposta do cliente B2B.
- A feature reutiliza MySQL, Prisma e a transação que já existe em `src/modules/orders/order.service.ts`.

### Negativas

- A notificação deixa de ser instantânea. O pior caso do polling acrescenta até 2 segundos, aceito porque continua abaixo dos 10 segundos pedidos pelo produto.
- A tabela cresce com eventos entregues até uma política de arquivamento que esta feature não implementa.
- Ordering entre eventos do mesmo `order_id` fica garantida só enquanto houver um único worker, porque a leitura segue `created_at`. Vários workers em paralelo quebram essa ordem. Particionar por `order_id` ou usar lock pessimista ficou para o futuro.
