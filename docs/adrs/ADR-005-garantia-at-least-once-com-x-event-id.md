# ADR-005: Garantia at-least-once com X-Event-Id

## Status

Aceito

## Contexto

O worker pode enviar, receber 2xx e cair antes de marcar a outbox como entregue. Retry também reenvia o mesmo fato. O cliente precisa distinguir “este evento eu já processei” de “este pedido mudou de novo”.

Exactly-once entre dois sistemas exigiria coordenação dos dois lados. Diego citou Stripe e GitHub como referência de mercado para o contrário: at-least-once com id de evento.

## Decisão

A entrega é at-least-once.

Quando a linha entra em `webhook_outbox`, o sistema gera um UUID. Esse id viaja no header `X-Event-Id` e no campo `event_id` do payload. É estável para todas as tentativas e para o replay da DLQ: recolocar o evento como pendente não cria outro id.

O receptor deduplica. Marcos se comprometeu a destacar isso no portal de desenvolvedores. A plataforma não oferece exactly-once.

Headers que acompanham o POST:

- `X-Event-Id`: UUID do evento
- `X-Signature`: HMAC-SHA256 do body ([ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md))
- `X-Timestamp`: instante do envio, para o cliente detectar replay se quiser
- `X-Webhook-Id`: id do endpoint cadastrado, para quem tem mais de um
- `Content-Type: application/json`

## Alternativas Consideradas

### Exactly-once

Exigiria acordo transacional com o receptor (ou um protocolo de confirmação de dois lados). A reunião considerou isso mais complexo do que o problema pede e suficiente o id para “99% dos casos”.

Trade-off do descarte: o cliente não precisaria deduplicar, em troca de um protocolo que a plataforma não controla sozinha e que a call rejeitou.

## Consequências

### Positivas

- Retry, crash depois do 2xx e replay continuam corretos sem fingir entrega única.
- O mesmo id serve de chave de correlação em log, métrica e suporte.

### Negativas

- Receptor sem dedup processa o mesmo status duas vezes. O efeito é do lado dele; a mitigação combinada é documentação no portal, não uma garantia técnica nova.
- `X-Timestamp` permite ao cliente recusar replay antigo, mas a plataforma não valida janela de tempo na ida. Isso não foi pedido.
