# ADR-002 — Worker em processo separado com polling de 2 segundos

| Campo | Valor |
| --- | --- |
| Status | Aceito |
| Data | Reunião técnica de quinta-feira, 09:00 (data do calendário não registrada) |
| Decisores | Tech Lead (Larissa), Engenharia de Plataforma (Diego), Engenharia de Pedidos (Bruno), Produto (Marcos) |
| Relacionados | [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md) |

## Status

**Aceito** na reunião técnica da feature e confirmado no resumo de encerramento.

## Contexto

Com a outbox definida no [ADR-001](ADR-001-outbox-no-mysql.md), é preciso decidir **como** e **onde** os eventos são lidos e enviados.

Restrições:

- **Latência:** para os clientes, abaixo de 10 segundos já é "tempo real".
- **O MySQL não oferece notificação para processos externos**, ao contrário do `NOTIFY/LISTEN` do Postgres; triggers só executam SQL.
- **Resiliência a reinícios da API:** o envio não pode parar quando a API reinicia.
- **Mesma stack:** o worker precisa do mesmo banco e do mesmo Prisma.
- Os clientes **não exigem ordenação global**, apenas saber que cada pedido mudou.

## Decisão

O envio é feito por um **worker em processo Node separado da API**, que faz **polling em loop a cada 2 segundos**, buscando os eventos pendentes mais antigos da outbox, processando e marcando o resultado.

> **Justificativa principal:** o polling de 2 s atende com folga o requisito de menos de 10 s sem depender de um mecanismo de notificação que o MySQL não tem, e o processo separado garante que o worker não seja perdido quando a API reinicia.

Fatores complementares:

| Aspecto | Decisão |
| --- | --- |
| Entry-point | Novo arquivo `src/worker.ts` (a criar), no molde de [src/server.ts](../../src/server.ts), com script `npm run worker` em [package.json](../../package.json) |
| Lógica de processamento | Dentro do módulo de webhooks (`webhook.worker.ts` ou `webhook.processor.ts`) |
| Banco | Mesmo banco e mesma `DATABASE_URL`, com **instância própria de `PrismaClient`**, porque o client é por processo |
| Concorrência | **Um único worker**; processamento em ordem de `created_at` da outbox |
| Latência aceita | Até 2 s no pior caso, aceito por engenharia e produto |

## Alternativas Consideradas

### A1. Trigger de banco para acionar o worker de forma reativa

- **Prós:** reação imediata à inserção, sem consultas periódicas.
- **Contras:** trigger no MySQL não notifica processo externo; avisar o worker exigiria improvisos, como escrever em arquivo ou chamar um endpoint.
- **Descarte:** o polling de 2 s já cumpre o requisito de latência.

### A2. Worker dentro do processo da API

- **Prós:** um único processo para implantar; reaproveita a instância de `PrismaClient` da API.
- **Contras:** se a API reinicia, o worker é perdido; o envio passa a competir com o atendimento HTTP.
- **Descarte:** o worker precisa ser outro processo, ainda que use o mesmo banco e a mesma stack.

### A3. Múltiplos workers em paralelo

- **Prós:** maior vazão de envio.
- **Contras:** perde a garantia de ordem dos eventos de um mesmo pedido.
- **Descarte:** adiado. Quando for necessário escalar, as opções levantadas são particionar por `order_id` ou usar lock pessimista.

## Consequências

**Positivas**

- Atende a latência exigida com uma solução simples e sem infraestrutura nova.
- Envio isolado da API: reinícios, deploys ou lentidão de um não derrubam o outro.
- Com worker único, os eventos de um mesmo pedido chegam na ordem em que aconteceram.

**Negativas e trade-offs**

- **Latência de até 2 s** introduzida pelo intervalo de polling.
- **Consultas contínuas ao banco** a cada 2 s, mesmo sem eventos.
- **Ordenação garantida apenas por `order_id` e enquanto houver um único worker**; não há ordenação global. Registrado como limitação conhecida.
- **Vazão limitada** a um worker até que a estratégia de escala seja decidida.

**Riscos e impactos operacionais**

- A aplicação passa a ter **dois processos** para implantar, monitorar e reiniciar (API e worker).
- O worker abre seu próprio pool de conexões no mesmo MySQL.
- Se o worker parar, os eventos se acumulam na outbox (sem perda, pela persistência do [ADR-001](ADR-001-outbox-no-mysql.md)) até ele voltar.

**Pontos em aberto** (da ata, sem decisão)

- Nome do arquivo de processamento: `webhook.worker.ts` ou `webhook.processor.ts` (QA-03).
- Recuperação de eventos presos em "processando" se o worker cair no meio do envio (LAC-08).

## Referências

- Código: [src/server.ts](../../src/server.ts) (modelo de entry-point com bootstrap e graceful shutdown), [src/config/database.ts](../../src/config/database.ts) (`createPrismaClient`), [package.json](../../package.json) (scripts).
- Detalhamento e rastreabilidade: [ata da reunião](../levantamento-tecnico/ATA-REUNIAO-WEBHOOKS.md) — DT-02, DT-07; ALT-03, ALT-04; FE-06; LIM-01, LIM-02.
