# ADR-004: HMAC-SHA256 com secret por endpoint

## Status

Aceito

## Contexto

O webhook sai da infra do OMS e chega a um HTTPS controlado pelo cliente, com dados de pedido. O receptor precisa saber que o corpo veio da plataforma e que não foi alterado no caminho. Uma secret única da plataforma inteira vazaria todos os clientes junto. Já houve secret de cliente vazada em log da aplicação dele.

A URL é cadastrada pelo integrador. HTTP claro não é aceitável.

## Decisão

Cada endpoint de webhook tem secret própria, gerada pela plataforma na criação e devolvida nessa resposta. O cliente não envia a secret no cadastro.

O worker assina o corpo cru do POST com HMAC-SHA256 e envia a assinatura no header `X-Signature`. Durante a rotação, a assinatura usa a secret vigente. A secret anterior permanece válida por 24 horas para o receptor conferir a assinatura antiga ou a nova enquanto migra. Depois disso, a anterior deixa de valer.

Rotação é endpoint da API. Não se edita a secret no PATCH comum.

TLS é obrigatório: URL sem `https` é recusada na validação. Sofia classificou isso como validação de schema, não como decisão de arquitetura separada. O código público desse erro é `WEBHOOK_INVALID_URL`, no padrão de códigos que Bruno pediu.

O formato textual da assinatura (hexadecimal ou base64, com ou sem prefixo) não foi escolhido na reunião. A revisão de segurança da Sofia, de pelo menos dois dias úteis antes do deploy, é o momento de cravar a string. Até lá o contrato fixa algoritmo, o que entra no HMAC (o body) e o nome do header.

## Alternativas Consideradas

### Secret global da plataforma

Um vazamento único comprometeria todos os endpoints. A reunião rejeitou isso de forma direta.

Trade-off do descarte: operação mais simples (uma chave para girar), em troca de raio de impacto total.

### Sem rotação, ou rotação que invalida na hora

Giro imediato quebra o receptor que ainda não trocou a chave. A reunião já tratou vazamento em log como caso real e por isso pediu janela de 24 horas com as duas secrets válidas.

Trade-off do descarte: a secret comprometida morre no ato, em troca de indisponibilidade da integração durante a migração.

## Consequências

### Positivas

- O cliente verifica origem e integridade sem conta mútua de callback.
- Vazamento de uma secret não entrega as outras.
- A janela de 24 horas dá tempo de migrar sem cortar a verificação.

### Negativas

- Durante 24 horas a secret anterior ainda autentica. É o trade-off explícito contra cortar na hora.
- A secret aparece na resposta de criação e de rotação. Logar esse campo repete o incidente que motivou a rotação. O logger Pino em `src/shared/logger/index.ts` precisa passar a redactar `secret`.
- Hex versus base64 fica em aberto até a revisão de segurança. Implementar um dos dois antes disso é escolha provisória, não decisão da call.
