# ADR-001 — Padrão Outbox no MySQL para publicação de eventos de webhook

| Campo | Valor |
| --- | --- |
| Status | Aceito |
| Data | Reunião técnica de quinta-feira, 09:00 (data do calendário não registrada) |
| Decisores | Tech Lead (Larissa), Engenharia de Plataforma (Diego), Engenharia de Pedidos (Bruno) |
| Relacionados | [ADR-002](ADR-002-worker-separado-com-polling.md), [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md) |

## Status

**Aceito** na reunião técnica da feature e confirmado no resumo de encerramento.

## Contexto

Três clientes B2B precisam ser notificados quando o status dos seus pedidos muda, com latência abaixo de 10 segundos. A mudança de status acontece em `OrderService.changeStatus` ([src/modules/orders/order.service.ts](../../src/modules/orders/order.service.ts)), que já executa dentro de uma transação interativa (`prisma.$transaction`), atualiza `orders`, grava `order_status_history` e, conforme a transição, movimenta o estoque.

Restrições que moldaram a decisão:

- **A transação de status já é pesada.** Um HTTP para o cliente dentro dela faria um cliente lento travar a mudança de status de outros pedidos.
- **Não é possível desfazer o status por falha do cliente.** Dar rollback na mudança de status porque o cliente está fora do ar não é aceitável.
- **Nunca pode haver status alterado sem evento registrado.**
- **Time pequeno**, sem disposição para operar nova infraestrutura.
- A aplicação hoje **não tem** mecanismo de fila, eventos ou jobs; o único armazenamento disponível é o MySQL ([prisma/schema.prisma](../../prisma/schema.prisma)).

## Decisão

Adotar o **padrão Outbox no MySQL existente**: ao mudar o status do pedido, o evento é inserido numa tabela de outbox (nome sugerido `webhook_outbox`) **na mesma transação SQL** que atualiza `orders` e `order_status_history`. Um worker separado lê essa tabela e faz o envio HTTP (ver [ADR-002](ADR-002-worker-separado-com-polling.md)).

> **Justificativa principal:** se a transação commitou, o evento está registrado; se deu rollback, o evento some junto. Não há inconsistência possível entre status e evento, e isso sem subir nenhuma infraestrutura nova.

Decisões que compõem o padrão:

| Aspecto | Decisão |
| --- | --- |
| Ponto de publicação | Função `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que recebe o `tx` da transação corrente e é chamada pelo `changeStatus`, sem injetar o repository de webhook inteiro no `OrderService` |
| Falha na inserção | Se a outbox falhar ao inserir, a transação inteira dá rollback |
| Filtro de eventos | Aplicado **na inserção**: se nenhum webhook do cliente assina aquele status, nenhuma linha é criada |
| Conteúdo | Payload **renderizado na inserção** (snapshot do estado no momento da mudança) |
| Identificador | UUID, seguindo o padrão do projeto |
| Leitura eficiente | Índices em status (pendente, processando, falhou, entregue) e em `created_at`; leitura dos pendentes em batch pequeno |

## Alternativas Consideradas

### A1. Disparo síncrono dentro do `changeStatus`

- **Prós:** implementação mais direta; sem tabela nem processo adicionais.
- **Contras:** acopla a latência e a disponibilidade do cliente à transação de status; um cliente lento trava mudanças de outros pedidos; não há o que fazer com o status se o cliente estiver fora do ar.
- **Descarte:** considerado fora de questão pelo time.

### A2. Fila externa (Redis Streams ou similar)

- **Prós:** mecanismo dedicado a filas, mais reativo que polling.
- **Contras:** exige subir e operar mais infraestrutura; para um time pequeno, um Redis Cluster para esse caso seria overengineering. Além disso, gravar num sistema externo não participa da transação do MySQL, perdendo a atomicidade que motivou a decisão.
- **Descarte:** o Outbox no MySQL existente resolve o problema.

### A3. Filtrar eventos no momento do envio

- **Prós:** a outbox registraria todas as mudanças, independentemente das assinaturas.
- **Contras:** gera linhas que nunca serão entregues.
- **Descarte:** filtrar na inserção economiza linhas na tabela.

### A4. Guardar só o `order_id` e renderizar o payload no envio

- **Prós:** outbox mais enxuta.
- **Contras:** se o pedido mudar depois, o evento não refletiria o estado do momento da transição.
- **Descarte:** snapshot na inserção.

## Consequências

**Positivas**

- Consistência garantida entre mudança de status e registro do evento, pela própria transação do banco.
- Nenhuma infraestrutura nova: mesmo MySQL, mesmo Prisma, mesma stack.
- A mudança de status deixa de depender da disponibilidade dos clientes.
- O snapshot preserva o histórico fiel do que aconteceu em cada transição.

**Negativas e trade-offs**

- A transação do `changeStatus` passa a fazer uma escrita adicional (quando houver assinatura para o status), e uma falha nessa escrita impede a mudança de status. É um trade-off aceito conscientemente: preferível a status sem evento.
- `OrderService` passa a depender do módulo de webhooks, ainda que por uma única função.
- O MySQL passa a servir também como fila, recebendo leituras periódicas do worker.

**Riscos e impactos operacionais**

- **Crescimento da tabela:** mitigado por índices e leitura em batch. O arquivamento das linhas entregues (~30 dias) está **fora do escopo** desta feature.
- A latência final depende do intervalo de leitura do worker (ver [ADR-002](ADR-002-worker-separado-com-polling.md)).

**Pontos em aberto** (da ata, sem decisão)

- Campos exatos do payload além de `total_cents` (QA-04).
- Onde validar o limite de 64 KB: na renderização, na inserção ou no envio (LAC-13).

## Referências

- Código: [src/modules/orders/order.service.ts](../../src/modules/orders/order.service.ts) (`changeStatus`, `prisma.$transaction`, tipo `TxClient`), [prisma/schema.prisma](../../prisma/schema.prisma) (padrão UUID e índices).
- Detalhamento e rastreabilidade: [ata da reunião](../levantamento-tecnico/ATA-REUNIAO-WEBHOOKS.md) — DT-01, DT-08, DT-09, DT-10, DT-11; ALT-01, ALT-02, ALT-12, ALT-15.
