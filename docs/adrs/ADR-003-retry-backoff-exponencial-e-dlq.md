# ADR-003 — Retry com backoff exponencial e Dead Letter Queue em tabela separada

| Campo | Valor |
| --- | --- |
| Status | Aceito |
| Data | Reunião técnica de quinta-feira, 09:00 (data do calendário não registrada) |
| Decisores | Tech Lead (Larissa), Engenharia de Plataforma (Diego), Engenharia de Pedidos (Bruno), Produto (Marcos), Segurança (Sofia) |
| Relacionados | [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-com-polling.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md) |

## Status

**Aceito** na reunião técnica da feature e confirmado no resumo de encerramento.

## Contexto

As entregas vão para sistemas fora da infraestrutura da plataforma, que podem estar lentos ou indisponíveis. É preciso definir o que acontece quando uma entrega falha.

Restrições:

- Já houve cliente com **indisponibilidade de duas horas** por manutenção planejada.
- Um evento **não pode ficar pendurado para sempre** se o cliente sumir.
- Eventos que falharam em definitivo precisam de **evidência para debug e reprocessamento**.
- Mexer na fila de entrega **não é atividade de operador** e precisa de trilha de auditoria.

## Decisão

1. **Backoff exponencial com 5 tentativas**, nos intervalos **1 min, 5 min, 30 min, 2 h e 12 h**, somando quase 15 horas entre a primeira falha e a última tentativa.
2. **Timeout de 10 s** por chamada HTTP: um cliente que não responde nesse prazo é tratado como falha e vai para retry.
3. Esgotadas as tentativas, a falha é considerada permanente e o evento vai para uma **DLQ em tabela separada** (`webhook_dead_letter`), com **payload, motivo da falha e timestamp**.
4. **Reprocessamento manual** via `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente. O endpoint exige **role ADMIN**, reaproveitando o `requireRole` de [src/middlewares/auth.middleware.ts](../../src/middlewares/auth.middleware.ts), e **registra quem fez o replay** para auditoria.

> **Justificativa principal:** cinco tentativas espaçadas cobrem indisponibilidades reais de horas, como a manutenção de 2 h já vista, sem manter eventos pendurados indefinidamente; o que ainda assim falhar fica isolado e preservado para análise e replay controlado.

Aceite de produto: um cliente fora do ar por 15 h já tem um problema sério do lado dele; a janela é considerada aceitável.

## Alternativas Consideradas

### A1. Retry indefinido com backoff

- **Prós:** nenhum evento é abandonado automaticamente.
- **Contras:** o evento pode ficar pendurado para sempre se o cliente sumir.
- **Descarte:** preferido um teto de tentativas seguido de DLQ.

### A2. Três tentativas

- **Prós:** mais agressivo; falhas definitivas são identificadas mais cedo.
- **Contras:** uma indisponibilidade matinal esgotaria as tentativas em cerca de 30 minutos, insuficiente para a manutenção de 2 h já observada.
- **Descarte:** três tentativas foram consideradas poucas.

### A3. DLQ como status "failed" na própria outbox

- **Prós:** uma tabela a menos; o evento não muda de lugar.
- **Contras:** mistura eventos ativos e mortos na tabela que o worker lê continuamente.
- **Descarte:** a tabela separada deixa a leitura da outbox mais limpa e guarda evidência para debug e reprocessamento.

## Consequências

**Positivas**

- Tolerância a indisponibilidades de até ~15 h sem intervenção humana.
- Outbox enxuta, com apenas eventos em andamento; eventos mortos ficam preservados à parte.
- Reprocessamento controlado, restrito a administradores e auditável.
- Reuso do controle de acesso existente (`requireRole`).

**Negativas e trade-offs**

- Uma entrega pode chegar **com até ~15 h de atraso** após a primeira falha.
- Eventos na DLQ **só voltam por ação manual** de um ADMIN; não há reprocessamento automático.
- O replay reenvia o evento, reforçando a necessidade de deduplicação no cliente (ver [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)).
- Um timeout de 10 s pode classificar como falha um cliente apenas lento, gerando retries e duplicatas.

**Riscos e impactos operacionais**

- É necessário acompanhar o volume da DLQ para acionar o replay.
- O alerta ao cliente por e-mail após falhas repetidas está **fora de escopo** desta fase.

**Pontos em aberto** (da ata, sem decisão)

- Se as "5 tentativas" incluem o envio inicial ou são 5 retries após a primeira falha, já que foram definidos 5 intervalos (QA-05).
- Quais respostas HTTP do cliente contam como sucesso, além do timeout como falha (LAC-06).
- Se o replay reinicia o contador de tentativas (LAC-10).

## Referências

- Código: [src/middlewares/auth.middleware.ts](../../src/middlewares/auth.middleware.ts) (`requireRole`), [src/shared/logger/index.ts](../../src/shared/logger/index.ts) (log de auditoria do replay).
- Detalhamento e rastreabilidade: [ata da reunião](../levantamento-tecnico/ATA-REUNIAO-WEBHOOKS.md) — DT-03, DT-13, DT-14; RF-09 a RF-12; ALT-05, ALT-06, ALT-07; LIM-04.
