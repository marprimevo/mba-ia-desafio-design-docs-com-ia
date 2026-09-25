# ADR-003: Retry com backoff e DLQ

## Status

Aceito

## Contexto

O cliente B2B pode estar fora do ar ou lento. Uma falha de HTTP não pode reverter o status do pedido nem segurar a outbox para sempre. Já houve cliente com manutenção planejada de cerca de duas horas. Retry curto demais desiste durante essa janela. Retry infinito deixa evento pendurado se o endpoint sumiu.

O worker trata timeout de 10 segundos como falha. Payload acima de 64 KB é erro e não deve ser reenviado à espera de encolher.

## Decisão

Backoff com cinco intervalos e, em seguida, dead letter em tabela separada.

Progressão fechada na reunião: 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas. Diego estimou quase 15 horas entre a primeira falha e a última tentativa, que é a soma desses intervalos (14h36). Larissa fechou “5 tentativas” junto com essa mesma lista.

Leitura usada na implementação, para não apagar nenhum intervalo nem a janela de ~15 horas:

1. Tentativa imediata, no próximo ciclo do worker.
2. Se falhar, retentativa após 1 minuto.
3. Se falhar, retentativa após 5 minutos.
4. Se falhar, retentativa após 30 minutos.
5. Se falhar, retentativa após 2 horas.
6. Se falhar, retentativa após 12 horas.
7. Se essa última falhar, o evento sai da outbox ativa e vai para `webhook_dead_letter`.

O rótulo “5 tentativas” do fechamento nomeia as cinco retentativas da progressão. A chamada inicial não é retentativa. Três tentativas foi rejeitado: cobriria algo como 30 minutos e mataria o evento numa manutenção de duas horas. Retry indefinido foi rejeitado pelo mesmo motivo do evento eterno.

A DLQ é a tabela `webhook_dead_letter`, com payload, motivo da falha e timestamp. Não é apenas um status `failed` na outbox. Reprocessamento é manual: `POST /api/v1/admin/webhooks/dead-letter/:id/replay`, somente role `ADMIN`, recolocando o evento como pendente e registrando quem fez o replay.

Timeout de 10 segundos, erro de rede e HTTP fora de 2xx contam como falha de tentativa. Payload acima de 64 KB não entra nessa escada: vai direto para a DLQ com código `WEBHOOK_PAYLOAD_TOO_LARGE`, porque um novo retry não muda o snapshot.

## Alternativas Consideradas

### Três tentativas

Bruno sugeriu um teto mais agressivo. Diego recusou: três tentativas em cerca de 30 minutos desistiriam no meio de uma manutenção de duas horas que o time já viu.

Trade-off do descarte: fila volta ao normal mais rápido e a DLQ enche menos, em troca de declarar falha permanente durante indisponibilidade que ainda é recuperável.

### Retry indefinido com backoff

Cobre cliente que volta depois de muitos dias, e também acumula evento de endpoint abandonado para sempre.

Trade-off do descarte: nenhuma perda por desistência precoce, em troca de crescimento sem teto da outbox e de operação sem um estado terminal claro.

### DLQ como status na própria outbox

Uma coluna `failed` evitaria outra tabela. Diego preferiu tabela separada para a outbox principal continuar legível e para a DLQ guardar evidência de debug e replay.

Trade-off do descarte: um modelo a menos, em troca de misturar tráfego vivo com falha permanente.

## Consequências

### Positivas

- Indisponibilidade de algumas horas, inclusive a manutenção de duas horas citada, cabe na janela.
- Existe estado terminal e um caminho de replay auditável.
- A outbox de trabalho não fica carregada de falha permanente.

### Negativas

- No pior caso o cliente recebe a mesma notificação várias vezes ao longo de ~15 horas. Deduplicação é do lado dele, via `X-Event-Id` ([ADR-005](ADR-005-garantia-at-least-once-com-x-event-id.md)).
- O número falado “5 tentativas” e a lista de cinco intervalos somados em ~15 horas não fecham na mesma contagem de chamadas HTTP. Esta ADR explicita a leitura (1 envio inicial + 5 retentativas). Uma revisão que queira teto literal de cinco chamadas precisa cortar um degrau; isso mudaria a janela de 15 horas ou a lista fechada.
- Replay recoloca como pendente e zera a contagem, senão o evento voltaria à DLQ no mesmo instante. A reunião disse “recoloca como pendente” e não detalhou o contador; zerar é a consequência necessária dessa frase.
