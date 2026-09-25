# ADR-007: Snapshot do payload na inserção da outbox

## Status

Aceito

## Contexto

O worker pode enviar o evento minutos ou horas depois da mudança de status, por causa do polling e do backoff. Nesse intervalo o pedido continua no banco e pode mudar de novo. Se o payload fosse montado na hora do HTTP, o corpo poderia descrever um estado posterior ao fato que o evento anuncia (`from_status` / `to_status`).

Bruno fez essa pergunta no fim da call, depois do resumo. Larissa e Diego fecharam snapshot na inserção.

## Decisão

`publishWebhookEvent` grava o JSON já renderizado na linha da outbox, no momento da transação de `changeStatus`.

O corpo segue o que Diego listou: `event_id`, `event_type` = `order.status_changed`, `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents`. Não inclui `items`. Quem precisa do pedido completo chama o `GET /api/v1/orders/:id` que já existe.

O worker envia esse JSON. Não relê a order para remontar o evento. Replay da DLQ reenvia o mesmo snapshot e o mesmo `event_id`.

## Alternativas Consideradas

### Guardar só `order_id` e renderizar no envio

O payload acompanharia o estado mais novo do pedido, inclusive mudanças posteriores de campos como `notes` ou um status que já avançou de novo.

Trade-off do descarte: menos bytes na outbox e um único lugar de formatação na hora do envio, em troca do caso que Larissa chamou de esquisito: o evento de `PAID` → `PROCESSING` sair com dados de um momento em que o pedido já está `SHIPPED`.

## Consequências

### Positivas

- O fato notificado é o da transação que commitou, mesmo que o retry saia 12 horas depois.
- O histórico de deliveries mostra o payload que foi de fato tentado.

### Negativas

- A outbox armazena o JSON completo de cada evento, não só uma chave. O teto de 64 KB se aplica a esse snapshot; estourar é erro, não truncamento.
- Um bug no formato do payload fica congelado nas linhas já inseridas. Corrigir o renderer não reescreve evento antigo.
