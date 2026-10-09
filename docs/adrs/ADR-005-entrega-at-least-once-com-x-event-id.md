# ADR-005 — Garantia de entrega at-least-once com `X-Event-Id` para deduplicação

| Campo | Valor |
| --- | --- |
| Status | Aceito |
| Data | Reunião técnica de quinta-feira, 09:00 (data do calendário não registrada) |
| Decisores | Engenharia de Plataforma (Diego), Tech Lead (Larissa), Segurança (Sofia), Produto (Marcos) |
| Relacionados | [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md), [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md) |

## Status

**Aceito** na reunião técnica da feature e confirmado no resumo de encerramento.

## Contexto

A combinação de outbox ([ADR-001](ADR-001-outbox-no-mysql.md)), worker ([ADR-002](ADR-002-worker-separado-com-polling.md)) e retry com replay ([ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md)) prioriza que nenhum evento se perca. Num sistema assim, o mesmo evento pode ser enviado mais de uma vez: por exemplo, quando o cliente processa a requisição mas a resposta não chega à plataforma, ou quando um evento da DLQ é reprocessado. É preciso definir qual garantia de entrega a plataforma oferece e como o cliente lida com duplicatas.

Restrições:

- **Time pequeno** e preferência por soluções simples, já expressa nas demais decisões.
- Garantir entrega única exigiria **coordenação entre os dois lados**.
- A regra precisa ser **comunicada claramente** aos clientes que vão integrar.

## Decisão

A plataforma garante entrega **at-least-once**: o cliente **pode receber o mesmo evento mais de uma vez** e deve estar preparado para isso.

Para permitir a deduplicação, cada requisição leva o header **`X-Event-Id`**, com um **UUID gerado quando o evento entra na outbox**, único por evento. O mesmo identificador vai no campo `event_id` do payload. O cliente descarta eventos cujo `event_id` já processou.

A semântica será documentada **em destaque no portal de desenvolvedor**, sob responsabilidade de Produto.

> **Justificativa principal:** é o padrão de mercado (adotado por provedores como Stripe e GitHub) e resolve a grande maioria dos casos com uma fração da complexidade de exactly-once, que exigiria coordenação dos dois lados.

## Alternativas Consideradas

### A1. Exactly-once

- **Prós:** o cliente nunca recebe duplicatas e não precisa implementar deduplicação.
- **Contras:** exigiria coordenação dos dois lados e tornaria a solução muito mais complexa.
- **Descarte:** at-least-once com identificador de evento.

## Consequências

**Positivas**

- Nenhum evento é descartado silenciosamente em falhas de rede ou do cliente; combinado com a outbox, o que foi registrado é entregue ou vai para a DLQ.
- Modelo simples de implementar e familiar para quem já integrou com provedores conhecidos.
- `X-Event-Id` também facilita o rastreio de um evento específico entre plataforma e cliente.

**Negativas e trade-offs**

- **A responsabilidade de deduplicar é do cliente.** Um cliente que ignore o `X-Event-Id` pode processar o mesmo evento duas vezes.
- Duplicatas são esperadas, não excepcionais: retries após timeout e replays da DLQ podem gerar reentregas.

**Riscos e impactos operacionais**

- A eficácia da decisão depende de **documentação clara** para os clientes.
- Não há garantia de ordem global entre eventos; a ordem por pedido depende do worker único ([ADR-002](ADR-002-worker-separado-com-polling.md)).

**Pontos em aberto** (da ata, sem decisão)

- Não foi explicitado se o replay de um evento da DLQ mantém o mesmo `event_id` ou gera um novo. A regra "único por evento" sugere manter, mas isso deve ser confirmado no FDD (relacionado a LAC-10).

## Referências

- Detalhamento e rastreabilidade: [ata da reunião](../levantamento-tecnico/ATA-REUNIAO-WEBHOOKS.md) — DT-05; RNF-06; ALT-10; FE-07; LIM-03; AC-05.
