# ADR-006 — Reuso dos padrões existentes do projeto no módulo de webhooks

| Campo | Valor |
| --- | --- |
| Status | Aceito |
| Data | Reunião técnica de quinta-feira, 09:00 (data do calendário não registrada) |
| Decisores | Tech Lead (Larissa), Engenharia de Pedidos (Bruno), Engenharia de Plataforma (Diego), Segurança (Sofia) |
| Relacionados | [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-com-polling.md), [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md), [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md) |

## Status

**Aceito** na reunião técnica da feature e confirmado no resumo de encerramento.

## Contexto

O OMS já possui padrões consolidados e aplicados de forma uniforme em todos os módulos:

| Padrão existente | Onde está |
| --- | --- |
| Módulo por domínio com controller, service, repository, routes e schemas | `src/modules/*` (ex.: [src/modules/orders/](../../src/modules/orders/)) |
| Hierarquia de erros com código estável em `UPPER_SNAKE_CASE` (`AppError`, `InsufficientStockError`, `InvalidStatusTransitionError`) | [src/shared/errors/app-error.ts](../../src/shared/errors/app-error.ts), [src/shared/errors/http-errors.ts](../../src/shared/errors/http-errors.ts) |
| Error middleware centralizado que traduz `AppError`, Zod e Prisma | [src/middlewares/error.middleware.ts](../../src/middlewares/error.middleware.ts) |
| Logger estruturado Pino | [src/shared/logger/index.ts](../../src/shared/logger/index.ts) |
| Validação de entrada com schemas Zod | [src/middlewares/validate.middleware.ts](../../src/middlewares/validate.middleware.ts) e `*.schemas.ts` |
| Controle de acesso por role | `requireRole` em [src/middlewares/auth.middleware.ts](../../src/middlewares/auth.middleware.ts) |
| Composição de dependências e registro de rotas | [src/app.ts](../../src/app.ts), [src/routes/index.ts](../../src/routes/index.ts) |
| Identificadores UUID | [prisma/schema.prisma](../../prisma/schema.prisma) |

A feature tem prazo apertado (3 sprints) e será desenvolvida pelo mesmo time que mantém a base. A questão era se o módulo de webhooks seguiria esses padrões ou introduziria estrutura e ferramentas próprias.

## Decisão

O módulo de webhooks segue **integralmente os padrões existentes** e é tratado como um módulo igual aos demais:

| Aspecto | Decisão |
| --- | --- |
| Estrutura | Módulo `src/modules/webhooks` (a criar), com controller, service, repository, routes e schemas |
| Erros | Subclasses de `AppError`, no modelo de `InsufficientStockError`/`InvalidStatusTransitionError`, com **prefixo `WEBHOOK_`** em todos os códigos (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) |
| Tratamento de erros | Error middleware atual, **sem alteração**: ele já traduz qualquer `AppError` |
| Logs | Pino existente; **nenhum logger novo** |
| Validação | Schemas Zod, incluindo a exigência de URL `https` |
| Autorização | `requireRole('ADMIN')` no replay da DLQ; demais rotas de configuração com autenticação normal, aceitando qualquer role por enquanto |
| Identificação do cliente | `customer_id` informado na requisição (body ou path), não derivado do JWT, que é do usuário interno |
| IDs | UUID, como o resto do projeto |
| Banco no worker | Mesma `DATABASE_URL`, com instância própria de `PrismaClient` (ver [ADR-002](ADR-002-worker-separado-com-polling.md)) |

> **Justificativa principal:** os padrões atuais já resolvem erros, logs, validação e autorização de forma uniforme; reaproveitá-los faz o módulo novo funcionar com a infraestrutura existente sem mudanças (o error middleware trata os novos erros sem alteração) e mantém a base previsível para o time.

## Alternativas Consideradas

### A1. Estrutura e ferramentas próprias para o módulo de webhooks

- **Prós:** liberdade para desenhar o módulo, com eventuais bibliotecas específicas para a feature.
- **Contras:** introduz formatos de erro, logs e organização divergentes do restante da base, exigindo adaptar o error middleware e a observabilidade existentes.
- **Descarte:** o time decidiu não introduzir nada novo e reaproveitar ao máximo o que já existe.

### A2. Worker compartilhando a instância de `PrismaClient` da API

- **Prós:** um único pool de conexões.
- **Contras:** incompatível com o worker em processo separado, já que o `PrismaClient` é por processo.
- **Descarte:** mesmo banco e mesma `DATABASE_URL`, com instância nova no worker.

### A3. `customer_id` implícito no JWT

- **Prós:** o cliente não precisaria informar o identificador.
- **Contras:** o JWT atual é do usuário operador, não do cliente; o modelo de autenticação existente não carrega essa informação.
- **Descarte:** `customer_id` explícito na requisição.

## Consequências

**Positivas**

- Erros do módulo chegam ao cliente no mesmo envelope `{ error: { code, message, details } }` das demais rotas, sem mudança no middleware.
- Logs da API e do worker no mesmo formato estruturado, com a mesma redaction de campos sensíveis.
- Curva de aprendizado mínima: quem conhece um módulo do OMS entende o de webhooks.
- Menos código novo, o que ajuda a cumprir o prazo de 3 sprints.

**Negativas e trade-offs**

- O módulo herda as limitações dos padrões atuais. Por exemplo, não há métricas nem tracing no projeto hoje, e o reuso do Pino não resolve isso.
- A autorização do CRUD de configuração fica ampla nesta fase (qualquer role autenticada); endurecê-la fica para uma fase futura.

**Riscos e impactos operacionais**

- Surgem novos pontos de registro no composition root ([src/app.ts](../../src/app.ts)) e no agregador de rotas ([src/routes/index.ts](../../src/routes/index.ts)), além de um ponto de integração no `OrderService` ([ADR-001](ADR-001-outbox-no-mysql.md)).
- O catálogo completo de códigos `WEBHOOK_*` ainda precisa ser fechado.

**Pontos em aberto** (da ata, sem decisão)

- `customer_id` no body ou no path (QA-01).
- Catálogo completo dos códigos `WEBHOOK_*` além dos três citados (QA-07).
- Quando o CRUD de configuração terá permissões mais restritas (QA-08).
- Como vincular "usuários que representam o cliente" a um `customer`, já que o schema atual não relaciona `User` e `Customer` (LAC-14).

## Referências

- Código: [src/shared/errors/app-error.ts](../../src/shared/errors/app-error.ts), [src/shared/errors/http-errors.ts](../../src/shared/errors/http-errors.ts), [src/middlewares/error.middleware.ts](../../src/middlewares/error.middleware.ts), [src/middlewares/auth.middleware.ts](../../src/middlewares/auth.middleware.ts), [src/middlewares/validate.middleware.ts](../../src/middlewares/validate.middleware.ts), [src/shared/logger/index.ts](../../src/shared/logger/index.ts), [src/app.ts](../../src/app.ts), [src/routes/index.ts](../../src/routes/index.ts), [prisma/schema.prisma](../../prisma/schema.prisma).
- Detalhamento e rastreabilidade: [ata da reunião](../levantamento-tecnico/ATA-REUNIAO-WEBHOOKS.md) — DT-06, DT-11, DT-12, DT-13; ALT-11, ALT-16; RNF-21, RNF-22.
