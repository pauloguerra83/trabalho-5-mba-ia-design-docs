# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) do projeto.
Cada decisão arquitetural relevante é registrada em um arquivo individual, respondendo à pergunta *"Por que decidimos exatamente assim?"*.

## Convenção

- Nome do arquivo: `ADR-NNN-titulo-em-kebab-case.md` (ex.: `ADR-001-outbox-no-mysql.md`), com numeração sequencial.
- Seções obrigatórias: **Status**, **Contexto**, **Decisão**, **Alternativas Consideradas** e **Consequências**. Cada ADR traz também uma seção de **Referências**, com links para o código relacionado.
- Os ADRs registram a decisão, a justificativa e as consequências. Os responsáveis aparecem de forma macro (papel e nome) apenas na tabela de metadados. O rastreio fala a fala fica na [ata da reunião](../levantamento-tecnico/ATA-REUNIAO-WEBHOOKS.md) e no Tracker.
- Questões não decididas na reunião aparecem como "Pontos em aberto", com o ID correspondente da ata (QA-xx, LAC-xx).

## Índice — Sistema de Webhooks de Notificação de Pedidos

| ADR | Decisão | Status |
| --- | --- | --- |
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL para publicação de eventos de webhook | Aceito |
| [ADR-002](ADR-002-worker-separado-com-polling.md) | Worker em processo separado com polling de 2 segundos | Aceito |
| [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md) | Retry com backoff exponencial e Dead Letter Queue em tabela separada | Aceito |
| [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md) | Autenticação das entregas com HMAC-SHA256 e secret por endpoint | Aceito |
| [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md) | Garantia de entrega at-least-once com `X-Event-Id` para deduplicação | Aceito |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes.md) | Reuso dos padrões existentes do projeto no módulo de webhooks | Aceito |

## Fontes

- [TRANSCRICAO.md](../../TRANSCRICAO.md): reunião técnica que originou as decisões.
- [ATA-REUNIAO-WEBHOOKS.md](../levantamento-tecnico/ATA-REUNIAO-WEBHOOKS.md): ata estruturada com decisões, requisitos, alternativas e questões em aberto.
- [LEVANTAMENTO-TECNICO.md](../levantamento-tecnico/LEVANTAMENTO-TECNICO.md): análise do código existente.
