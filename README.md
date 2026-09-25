# Da reunião ao documento: webhooks de pedido

## Sobre o desafio

O OMS deste repositório já opera pedidos, estoque e histórico de status, e não avisa ninguém quando o status muda. Numa call de cerca de 55 minutos, tech lead, PM, pedidos, plataforma e segurança fecharam um sistema de webhooks de saída para a Atlas Comercial, a MaxDistribuição e a Nova Cargo. A única memória da decisão era `TRANSCRICAO.md`.

O trabalho foi transformar essa call, lida contra o código que já existe, num pacote em que cada documento ocupa uma altura: o PRD diz por que e o quê, o RFC propõe a arquitetura e o que ficou em aberto, os ADRs registram cada decisão fechada, o FDD diz como construir, e o tracker aponta a fala ou o arquivo de onde cada item saiu. Nada em `src/`, `prisma/`, `tests/` ou na transcrição foi reescrito. O enunciado original do curso continua em [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Ferramentas de IA utilizadas

- **Cursor (agente Grok), com o repositório aberto.** Leu `TRANSCRICAO.md` e os módulos de pedidos, erro, auth, logger e Prisma. Produziu os rascunhos e foi corrigido nos pontos em que a call se corrige sozinha ou em que o código não faz o que a primeira leitura sugeria.
- **GitHub CLI (`gh`).** Fork público do repositório base para `marprimevo/mba-ia-desafio-design-docs-com-ia`, com `upstream` apontando para `devfullcycle`.
- **Referência de ADR do curso.** O fluxo e o formato MADR do plugin [adrs-management](https://github.com/devfullcycle/claude-mkt-place/tree/main/plugins/adrs-management) serviram de molde (status, contexto, decisão, alternativas, consequências). Os prompts de PRD e de FDD no Notion foram consultados; o conteúdo da página não veio no fetch, então a estrutura obrigatória do enunciado prevaleceu.

## Workflow adotado

1. Fork, clone em `MBAProjects` (mesmo padrão dos outros desafios do curso) e `.venv` local, só para isolar o ambiente. A aplicação é Node; o virtualenv não entra no desenho da feature.
2. Leitura da transcrição inteira e do caminho `changeStatus`, da máquina de status, de `AppError`, de `requireRole` e do schema Prisma, antes de escrever requisito.
3. ADRs das decisões fechadas, na ordem em que a call as cravou: outbox, worker, retry/DLQ, HMAC, at-least-once, reuso do código, snapshot.
4. RFC em cima desses ADRs, curto, com as alternativas que morreram na mesa e o que ficou para depois.
5. FDD com fluxo, contrato HTTP, matriz `WEBHOOK_` e a seção de integração nomeando arquivo real.
6. PRD por último, como consolidação de produto, sem repetir payload e status code do FDD.
7. Tracker varrendo os documentos prontos. Onde não havia timestamp nem arquivo, o texto saía.
8. Este README, com o processo já encerrado.

## Prompts customizados

Filtro do que não entra, usado antes de redigir PRD e RFC:

```text
Leia TRANSCRICAO.md e separe em quatro listas, cada item com timestamp e falante:
1) decisão fechada por Larissa ou por concordância explícita;
2) requisito funcional que Marcos ou Bruno pediram e ninguém retirou;
3) ideia descartada, com a frase que a mata e o trade-off dito na hora;
4) ponto adiado ou "observar depois".
Não promova item da lista 3 ou 4 a requisito. Se Marcos disser uma coisa e Larissa corrigir em seguida, vale a correção.
```

Amarração ao código, usado antes do FDD e do ADR de reuso:

```text
Para cada decisão da lista 1, aponte o arquivo em src/ ou prisma/ que ela estende.
O gancho citado na call é OrderService.changeStatus. Confirme no código o que a transação já faz
(order, order_status_history, estoque) e se OrderService.create também muda status.
Não proponha arquivo que não existe. Se o padrão de erro, de role ou de id já estiver
em http-errors.ts, auth.middleware.ts ou schema.prisma, o FDD reutiliza esse caminho
em vez de criar um paralelo.
```

## Iterações e ajustes

A primeira leitura tratava o `customer_id` como sujeito do JWT, porque Marcos disse isso às 09:31. Dois minutos depois Bruno observa que o JWT é do operador, e Larissa fecha: o id do cliente vai no body ou no path. O PRD e o contrato do `POST` seguem a correção. Um documento que amarrasse o webhook a `req.user.id` estaria mentindo sobre a call.

A mesma passada inicial empurrava e-mail de falha, painel, Redis e POST síncrono para a lista de requisitos, porque todos foram falados. E-mail e painel foram recusados por Larissa. Redis e síncrono foram recusados por Diego e Bruno. Rate limit não foi recusado: ficou para observar, e por isso mora nas questões em aberto do RFC, não no escopo do PRD.

Havia ainda um número fácil de “arredondar”. Diego lista cinco intervalos (1 min, 5 min, 30 min, 2 h, 12 h) e uma janela de quase 15 horas, que é a soma deles. Larissa resume “5 tentativas” com a mesma lista. Um backoff genérico de três retries, ou cinco chamadas que deixam o degrau de 12 horas de fora, perde a conta. O ADR-003 e o FDD registram a leitura que preserva os cinco intervalos: um envio imediato mais cinco retentativas. Os 10 segundos do produto (tempo até a primeira tentativa) também não são os 10 segundos do timeout HTTP; misturar os dois furava a meta e o critério de falha ao mesmo tempo.

Por último, `OrderService.create` grava `PENDING` sem passar por `changeStatus`. A call só autorizou o gancho nesse método. O FDD não emite webhook na criação. A codificação hex ou base64 do HMAC não aparece na transcrição e não foi inventada: ficou para os dois dias de revisão da Sofia.

## Como navegar a entrega

Ordem sugerida:

1. [docs/PRD.md](docs/PRD.md) — problema, escopo, o que ficou de fora, meta de 10 segundos.
2. [docs/RFC.md](docs/RFC.md) — proposta, alternativas mortas, o que ainda está aberto.
3. [docs/adrs/](docs/adrs/README.md) — uma decisão por arquivo, do ADR-001 ao ADR-007.
4. [docs/FDD.md](docs/FDD.md) — fluxo, contratos, erros `WEBHOOK_`, integração com o código.
5. [docs/TRACKER.md](docs/TRACKER.md) — de onde veio cada item.
6. [TRANSCRICAO.md](TRANSCRICAO.md) — a call, intacta.

O fork público é [marprimevo/mba-ia-desafio-design-docs-com-ia](https://github.com/marprimevo/mba-ia-desafio-design-docs-com-ia). `upstream` aponta para o repositório base do curso.
