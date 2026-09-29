# Ata de Reunião — Sistema de Webhooks de Notificação de Pedidos

> **Propósito deste documento:** registrar, de forma estruturada e rastreável, tudo o que foi discutido na reunião técnica transcrita em [TRANSCRICAO.md](../../TRANSCRICAO.md), para servir de **fonte única de contexto** na produção futura de PRD, RFC, ADRs, FDD e Tracker, sem necessidade de reler a transcrição.
>
> **Regra de rastreabilidade:** todo item traz a origem no formato `[hh:mm] Nome`. Quando um item vem do código-fonte, a origem é o caminho do arquivo. Itens marcados como **(obs. do analista)** não foram ditos na reunião: são constatações objetivas sobre o código ou sobre lacunas da conversa, e **não** devem ser tratados como decisão.
>
> **Documento complementar:** [LEVANTAMENTO-TECNICO.md](LEVANTAMENTO-TECNICO.md) (análise do código existente).

---

## Sumário

1. [Dados da reunião](#1-dados-da-reunião)
2. [Resumo executivo](#2-resumo-executivo)
3. [Contexto de negócio e problema](#3-contexto-de-negócio-e-problema)
4. [Decisões técnicas](#4-decisões-técnicas)
5. [Requisitos funcionais](#5-requisitos-funcionais)
6. [Requisitos não funcionais](#6-requisitos-não-funcionais)
7. [Especificações técnicas secundárias](#7-especificações-técnicas-secundárias)
8. [Integração com o código existente](#8-integração-com-o-código-existente)
9. [Alternativas descartadas](#9-alternativas-descartadas)
10. [Fora de escopo](#10-fora-de-escopo)
11. [Evoluções futuras](#11-evoluções-futuras)
12. [Limitações conhecidas aceitas](#12-limitações-conhecidas-aceitas)
13. [Questões em aberto](#13-questões-em-aberto)
14. [Lacunas não discutidas](#14-lacunas-não-discutidas)
15. [Planejamento, prazo e riscos](#15-planejamento-prazo-e-riscos)
16. [Itens de ação](#16-itens-de-ação)
17. [Glossário](#17-glossário)
18. [Guia de uso: o que vai para cada documento](#18-guia-de-uso-o-que-vai-para-cada-documento)

---

## 1. Dados da reunião

| Campo | Valor |
| --- | --- |
| Tema | Sistema de Webhooks de Notificação de Pedidos |
| Data | Quinta-feira, 09:00 (data do calendário não registrada na transcrição) |
| Duração | ~55 minutos (09:00 → 09:53) |
| Formato | Call remota (Meet) |
| Condução | Larissa |

| Participante | Papel | Foco na reunião |
| --- | --- | --- |
| **Larissa** | Tech Lead (condução) | Fechamento das decisões, estimativa, resumo final |
| **Marcos** | Product Manager | Contexto de negócio, requisitos funcionais, prazo com clientes |
| **Bruno** | Engenheiro Pleno, time de Pedidos | Integração com `orders`, padrões do código, CRUD |
| **Diego** | Engenheiro Sênior, time de Plataforma | Arquitetura (outbox, worker, retry, DLQ, entrega) — entrou às 09:05 |
| **Sofia** | Engenheira de Segurança | HMAC, secrets, TLS, limites, permissões, revisão de segurança |

Saídas da call: Marcos e Sofia saíram às 09:50. As decisões de **UUID na outbox** (09:51) e **snapshot do payload** (09:52) foram tomadas depois disso, apenas entre Larissa, Diego e Bruno.

---

## 2. Resumo executivo

Três clientes B2B pediram notificação em tempo real (abaixo de 10 s) da mudança de status dos seus pedidos. O time decidiu construir **webhooks de saída (outbound)** usando o **padrão Outbox no MySQL existente**: o evento é gravado na tabela de outbox **dentro da mesma transação** do `changeStatus` do módulo de pedidos, e um **worker em processo separado**, fazendo **polling a cada 2 s**, envia as chamadas HTTP.

A entrega é **at-least-once**, com `X-Event-Id` para deduplicação no cliente. Falhas usam **backoff exponencial de 1m/5m/30m/2h/12h (5 tentativas)** e, esgotadas, vão para uma **DLQ em tabela separada**, com **replay manual por endpoint restrito a ADMIN**. Cada endpoint tem **secret própria**, as requisições são **assinadas com HMAC-SHA256**, a secret pode ser **rotacionada com 24 h de convivência** e a URL precisa ser **HTTPS**.

A implementação **reaproveita os padrões do projeto** (módulo em `src/modules/webhooks`, `AppError`, códigos `WEBHOOK_*`, Pino, error middleware, Zod, `requireRole`). Ficaram **fora**: e-mail de alerta, dashboard visual e rate limiting (este em observação). A estimativa é de **3 sprints**, incluindo **2 dias úteis de revisão de segurança**, com prazo alvo no **fim de novembro**.

Resumo oficial da própria reunião: `[09:48] Larissa`.

---

## 3. Contexto de negócio e problema

| ID | Item | Origem |
| --- | --- | --- |
| CTX-01 | Três clientes B2B fizeram pedido formal: **Atlas Comercial, MaxDistribuição e Nova Cargo**. | `[09:00] Marcos` |
| CTX-02 | Os clientes querem ser notificados **em tempo real** quando o status dos pedidos deles muda. | `[09:00] Marcos` |
| CTX-03 | **Situação atual:** os clientes fazem polling em `GET /orders` periodicamente; a integração está **lenta e cara** para eles. | `[09:00] Marcos` |
| CTX-04 | **Risco comercial:** a Atlas sinalizou que pode **migrar para o concorrente** se a entrega não acontecer até o fim do trimestre. | `[09:00] Marcos` |
| CTX-05 | Para os clientes, **"tempo real" = qualquer coisa abaixo de 10 segundos**; o essencial é não precisar ficar atualizando manualmente. | `[09:02] Marcos` |
| CTX-06 | Os webhooks são **somente de saída** (da plataforma para o cliente). Os clientes querem receber, não enviar. | `[09:02] Sofia`, `[09:02] Marcos`, `[09:03] Sofia` |
| CTX-07 | Os clientes **nunca pediram ordenação global**; querem apenas saber se cada pedido deles mudou. | `[09:14] Marcos` |
| CTX-08 | Os clientes usarão a **API da plataforma diretamente**, autenticados com o JWT do sistema, por meio de **usuários que representam o cliente**. | `[09:32] Marcos` |
| CTX-09 | O prazo pedido pela Atlas é o **fim de novembro**. | `[09:45] Marcos` |
| CTX-10 | **Estado atual do código (obs. do analista):** a aplicação não possui mecanismo de eventos, filas, jobs em background nem notificação externa. | Código (`src/`) |

---

## 4. Decisões técnicas

Decisões **fechadas** na reunião. As seis primeiras são as decisões arquiteturais principais (candidatas naturais a ADR); as demais são decisões de desenho complementares.

### 4.1 Decisões arquiteturais principais

#### DT-01 — Padrão Outbox no MySQL existente

| Aspecto | Conteúdo | Origem |
| --- | --- | --- |
| Decisão | Ao mudar o status do pedido, inserir o evento numa tabela de outbox (nome sugerido `webhook_outbox`) **na mesma transação SQL** que atualiza `orders` e `order_status_history`. Um worker separado lê a tabela e dispara o HTTP. | `[09:06] Diego`, `[09:08] Larissa` |
| Motivação | Transação commitou → evento registrado; rollback → evento some junto. "Não tem inconsistência possível." | `[09:06] Diego` |
| Por que não síncrono | A transação de `changeStatus` já é pesada; um HTTP no meio faria um cliente lento travar mudanças de status de outros pedidos; e não é possível dar rollback no status se o cliente estiver fora do ar. | `[09:04] Bruno` |
| Por que não Redis Streams | Exigiria subir mais infraestrutura; time pequeno; "overengineering". | `[09:07] Larissa`, `[09:07] Diego` |
| Estados da outbox | Índice no campo de status (**pendente, processando, falhou, entregue**) e em `created_at`. | `[09:08] Diego` |
| Leitura | Worker lê **só os pendentes, em batch pequeno**, processa e marca como entregue. | `[09:08] Diego` |
| Filtro na inserção | Só insere na outbox se algum webhook do cliente quiser aquele status (ver RF-06). | `[09:34] Bruno`, `[09:34] Diego` |
| Regra de falha | Se a inserção na outbox falhar, a transação inteira dá rollback. "Não pode ter caso de status mudar e evento não sair." | `[09:40] Bruno`, `[09:41] Diego` |

#### DT-02 — Worker em processo separado, com polling de 2 segundos

| Aspecto | Conteúdo | Origem |
| --- | --- | --- |
| Decisão | Worker em **polling em loop a cada 2 s**, buscando os eventos pendentes **mais antigos**. | `[09:09] Diego`, `[09:10] Larissa` |
| Latência | Larissa registrou: "A latência mínima vai ser 2 segundos no pior caso. Aceitamos." Atende com folga o requisito de < 10 s. | `[09:10] Larissa`, `[09:09] Diego`, `[09:10] Marcos` |
| Processo separado | O worker **não pode rodar na mesma instância da API**: se a API reinicia, perde o worker. | `[09:11] Diego` |
| Entry-point | Novo arquivo `src/worker.ts` (no molde de `src/server.ts`) e script `npm run worker`. | `[09:11] Larissa`, `[09:28] Bruno` |
| Lógica | Fica dentro do módulo: `src/modules/webhooks/webhook.worker.ts` **ou** `webhook.processor.ts` (nome não fechado, ver QA-03). | `[09:28] Bruno` |
| Banco e stack | Mesmo banco, mesma stack, mesma `DATABASE_URL`; **instância própria de `PrismaClient`**, porque o PrismaClient é por processo. | `[09:11] Bruno`, `[09:11] Diego`, `[09:30] Bruno` |
| Por que não trigger | MySQL não tem listener nativo (como `NOTIFY/LISTEN` do Postgres); trigger só executa SQL e não notifica processo externo; avisar o worker exigiria improviso. | `[09:09] Bruno`, `[09:09] Diego` |

#### DT-03 — Retry com backoff exponencial e DLQ em tabela separada

| Aspecto | Conteúdo | Origem |
| --- | --- | --- |
| Decisão | Backoff exponencial com **5 tentativas**, intervalos **1 min, 5 min, 30 min, 2 h, 12 h**. | `[09:15] Diego`, `[09:17] Diego`, `[09:17] Larissa` |
| Janela total | "Quase 15 horas entre primeira falha e última tentativa." | `[09:17] Diego` |
| Aceite de produto | Cliente fora do ar por 15 h "já tá com problema sério dele"; aceitável. | `[09:17] Marcos` |
| Justificativa das 5 | 3 é pouco: uma indisponibilidade matinal seria esgotada em ~30 min; já houve cliente com **2 h de manutenção planejada**. | `[09:16] Diego` |
| Falha permanente | Esgotadas as tentativas, o evento é considerado falha permanente e vai para a DLQ. | `[09:15] Diego` |
| DLQ | Tabela **separada** (`webhook_dead_letter`) com **payload, motivo da falha e timestamp**. Mantém a outbox limpa e serve de evidência para debug e reprocessamento. | `[09:18] Diego`, `[09:48] Larissa` |
| Reprocessamento | **Manual**, via endpoint admin que **recoloca o evento na outbox como pendente** (ver RF-09). | `[09:18] Diego`, `[09:19] Larissa` |
| Timeout como falha | Resposta acima de 10 s é tratada como falha e marcada para retry (ver RNF-04). | `[09:42] Diego` |

#### DT-04 — Autenticação por HMAC-SHA256 com secret por endpoint

| Aspecto | Conteúdo | Origem |
| --- | --- | --- |
| Problema | Os eventos com dados de pedidos saem da infraestrutura; o cliente precisa validar que a requisição veio da plataforma e que o payload não foi adulterado. | `[09:19] Sofia` |
| Decisão | Assinar com **HMAC-SHA256 sobre o corpo do request**, usando secret compartilhada; assinatura enviada no header **`X-Signature`**; verificação no lado do cliente. | `[09:20] Sofia`, `[09:22] Sofia` |
| Por que SHA-256 | Padrão de mercado; "todo cliente sério tem biblioteca pra isso". | `[09:20] Sofia` |
| Secret por endpoint | Cada endpoint de webhook tem **secret única**, nunca uma secret global: "se vaza uma, vaza tudo". | `[09:21] Sofia` |
| Geração | A secret é **gerada pela plataforma** e **devolvida na criação** do webhook. | `[09:31] Marcos` |
| Rotação | O cliente pode **pedir nova secret pela API**; a antiga continua válida **por 24 h em paralelo**; depois disso, é invalidada. | `[09:21] Sofia`, `[09:22] Sofia` |
| Evidência | Já houve cliente que vazou secret em log de aplicação. | `[09:22] Diego` |

#### DT-05 — Garantia de entrega at-least-once com `X-Event-Id`

| Aspecto | Conteúdo | Origem |
| --- | --- | --- |
| Decisão | Garantia **at-least-once**: o cliente pode receber o mesmo evento mais de uma vez e deve estar preparado. | `[09:24] Diego`, `[09:26] Larissa` |
| Mecanismo | Header **`X-Event-Id`** com **UUID gerado quando o evento entra na outbox**, único por evento; o cliente deduplica por ele. | `[09:25] Diego` |
| Trade-off assumido | Transfere a responsabilidade de deduplicação para o cliente. | `[09:25] Sofia` |
| Justificativa | Padrão de mercado (Stripe e GitHub fazem assim). Exactly-once exigiria coordenação dos dois lados; at-least-once com event_id "resolve 99% dos casos". | `[09:25] Diego` |
| Comunicação | Marcos vai documentar isso **em destaque no portal de desenvolvedor**. | `[09:26] Marcos` |

#### DT-06 — Reuso máximo dos padrões existentes do projeto

| Aspecto | Conteúdo | Origem |
| --- | --- | --- |
| Decisão | "Reuso máximo do que já existe": `AppError`, Pino, error middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro. Webhook fica como um módulo igual aos outros. | `[09:30] Larissa`, `[09:48] Larissa` |
| Módulo | `src/modules/webhooks` com controller, service, repository, routes e schemas. | `[09:27] Bruno` |
| Erros | Seguir `AppError` e subclasses (como `InsufficientStockError`, `InvalidStatusTransitionError`), com **prefixo `WEBHOOK_`** em todos os códigos do módulo. Exemplos citados: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`. | `[09:28] Bruno`, `[09:29] Larissa` |
| Logger | Pino, já usado no projeto; **nenhum logger novo**. | `[09:29] Bruno` |
| Error middleware | O middleware centralizado já trata `AppError`, Zod e Prisma; vai tratar os erros do módulo **sem alteração**. | `[09:29] Bruno` |
| RBAC | Reaproveitar o `requireRole` existente no endpoint de replay. | `[09:36] Larissa` |
| Validação de URL | A exigência de HTTPS "é só uma validação no schema Zod". | `[09:23] Sofia` |

### 4.2 Decisões de desenho complementares

| ID | Decisão | Origem |
| --- | --- | --- |
| DT-07 | **Ordenação por pedido com worker único:** o worker processa em ordem de `created_at` da outbox; a ordem é garantida **apenas por `order_id` e enquanto houver um único worker**. Não há garantia de ordenação global. | `[09:12] Diego`, `[09:13] Larissa` |
| DT-08 | **Publicação por função que recebe o `tx`:** `publishWebhookEvent(tx, order, fromStatus, toStatus)`, chamada pelo `order.service`; "função pura recebendo o tx", **sem injetar o repository inteiro** no `OrderService`. | `[09:41] Bruno`, `[09:41] Diego` |
| DT-09 | **Filtro de eventos na inserção da outbox**, não no envio: se nenhum webhook do cliente quer aquele status, a linha nem é criada. | `[09:34] Bruno`, `[09:34] Diego` |
| DT-10 | **Snapshot do payload na inserção:** a outbox guarda o payload já renderizado, refletindo o estado do pedido no momento da mudança de status. | `[09:52] Larissa`, `[09:52] Diego`, `[09:52] Bruno` |
| DT-11 | **IDs em UUID** na outbox, seguindo o padrão do resto do projeto (não auto incremento). | `[09:51] Larissa` |
| DT-12 | **`customer_id` explícito na requisição** (body ou path) nas rotas de configuração; **não vem do JWT**. | `[09:32] Larissa` |
| DT-13 | **Permissões:** replay de DLQ exige role **ADMIN**; CRUD de configuração aceita **qualquer role autenticada** (por enquanto). | `[09:36] Sofia`, `[09:36] Larissa`, `[09:37] Sofia` |
| DT-14 | **Timeout de 10 s** no HTTP do worker; estourou, conta como falha e vai para retry. | `[09:42] Diego` |
| DT-15 | **Limite de 64 KB de payload**, com **erro** (não truncamento) se ultrapassar. Classificado como requisito não funcional, não decisão arquitetural. | `[09:23] Sofia`, `[09:24] Diego`, `[09:24] Larissa` |
| DT-16 | **HTTPS obrigatório** na URL; `http` é recusado com erro de validação. Não é decisão arquitetural, apenas validação de schema. | `[09:23] Sofia` |

---

## 5. Requisitos funcionais

| ID | Requisito | Detalhes discutidos | Origem |
| --- | --- | --- | --- |
| **RF-01** | **Notificar o cliente quando o status de um pedido muda** | Evento `order.status_changed`, enviado por HTTP POST para a URL cadastrada, a partir da mudança de status do pedido. | `[09:00] Marcos`, `[09:43] Diego` |
| **RF-02** | **Cadastrar webhook** | `POST`. Campos: `url`; lista de status que deseja receber; `customer_id` no body ou no path (ver QA-01). A **secret é gerada pela plataforma e devolvida na criação**. | `[09:31] Marcos`, `[09:32] Larissa` |
| **RF-03** | **Editar webhook** | `PATCH`. | `[09:33] Bruno` |
| **RF-04** | **Remover webhook** | `DELETE`. | `[09:33] Bruno` |
| **RF-05** | **Listar webhooks de um cliente** | `GET` que retorna os webhooks de um `customer`. | `[09:33] Bruno` |
| **RF-06** | **Filtro de eventos por endpoint** | Cada webhook escolhe quais status quer ouvir (ex.: só `SHIPPED` e `DELIVERED`). O filtro é aplicado **na inserção na outbox**. | `[09:33] Bruno`, `[09:33] Marcos`, `[09:34] Bruno` |
| **RF-07** | **Histórico de entregas** | `GET /webhooks/:id/deliveries`. Exemplo do PM: "os últimos 100 webhooks que vocês mandaram pra mim", com **sucesso/falha, payload, response e tempo de resposta**. | `[09:34] Marcos` |
| **RF-08** | **Rotação de secret** | Endpoint para o cliente pedir nova secret pela API; a antiga fica válida **24 h em paralelo** e depois é invalidada. | `[09:21] Sofia` |
| **RF-09** | **Replay manual de evento da DLQ** | `POST /admin/webhooks/dead-letter/:id/replay`. Recoloca o evento na outbox como pendente. | `[09:18] Diego`, `[09:35] Diego` |
| **RF-10** | **Auditoria do replay** | O endpoint de replay deve **registrar em log quem fez o replay**. | `[09:36] Sofia` |
| **RF-11** | **Retry automático de entregas com falha** | Backoff 1m/5m/30m/2h/12h, 5 tentativas (DT-03). | `[09:17] Larissa` |
| **RF-12** | **Mover para DLQ após esgotar tentativas** | Tabela `webhook_dead_letter` com payload, motivo da falha e timestamp. | `[09:15] Diego`, `[09:18] Diego` |
| **RF-13** | **Estado ativo do webhook** | A configuração armazena `url`, `secret`, `customer_id` e **estado ativo**. | `[09:21] Bruno`, `[09:21] Sofia` |
| **RF-14** | **Validação de URL HTTPS** | Cadastro com `http` é recusado com erro de validação. | `[09:23] Sofia` |

---

## 6. Requisitos não funcionais

| ID | Categoria | Requisito | Origem |
| --- | --- | --- | --- |
| **RNF-01** | Latência | Entrega **abaixo de 10 s** após a mudança de status (expectativa dos clientes). | `[09:02] Marcos` |
| **RNF-02** | Latência | Polling a cada **2 s**; latência de 2 s no pior caso aceita pelo time. | `[09:09] Diego`, `[09:10] Larissa` |
| **RNF-03** | Consistência | Atomicidade entre mudança de status e registro do evento: nunca pode haver status alterado sem evento. | `[09:06] Diego`, `[09:40] Bruno`, `[09:41] Diego` |
| **RNF-04** | Resiliência | **Timeout de 10 s** por chamada HTTP; estouro = falha e retry. | `[09:42] Diego` |
| **RNF-05** | Resiliência | Janela de retry de ~15 h para cobrir indisponibilidades e manutenções planejadas do cliente. | `[09:16] Diego`, `[09:17] Diego` |
| **RNF-06** | Entrega | Garantia **at-least-once**; clientes devem deduplicar por `X-Event-Id`. | `[09:24] Diego`, `[09:25] Diego` |
| **RNF-07** | Ordenação | Ordem garantida apenas **por `order_id`**, com worker único. | `[09:12] Diego`, `[09:13] Larissa` |
| **RNF-08** | Isolamento | Um cliente lento ou fora do ar **não pode travar** mudanças de status de outros pedidos. | `[09:04] Bruno` |
| **RNF-09** | Disponibilidade | O worker roda em **processo separado** da API; reinício da API não derruba o worker. | `[09:11] Diego` |
| **RNF-10** | Segurança | **HMAC-SHA256** sobre o corpo do request, header `X-Signature`. | `[09:20] Sofia`, `[09:22] Sofia` |
| **RNF-11** | Segurança | **Secret única por endpoint**, gerada pela plataforma. | `[09:21] Sofia`, `[09:31] Marcos` |
| **RNF-12** | Segurança | **Rotação de secret** com grace period de **24 h**. | `[09:21] Sofia` |
| **RNF-13** | Segurança | **TLS obrigatório**: URL `https`. | `[09:23] Sofia` |
| **RNF-14** | Segurança / Limite | Payload máximo de **64 KB**; acima disso, **erro** (não truncar). | `[09:23] Sofia`, `[09:24] Diego`, `[09:24] Larissa` |
| **RNF-15** | Segurança | Replay de DLQ restrito a role **ADMIN**. | `[09:36] Sofia` |
| **RNF-16** | Auditoria | Registrar em log quem executou o replay. | `[09:36] Sofia` |
| **RNF-17** | Anti-replay (lado cliente) | Header `X-Timestamp` para o cliente **poder** detectar replay attack, se quiser. | `[09:44] Diego` |
| **RNF-18** | Desempenho | Outbox com índices em **status** e **`created_at`**; worker lê em **batch pequeno**. | `[09:08] Diego` |
| **RNF-19** | Eficiência | Filtrar na inserção para não criar linhas desnecessárias na outbox. | `[09:34] Bruno` |
| **RNF-20** | Payload enxuto | Payload **sem os itens** do pedido, "pra não inflar". | `[09:43] Diego`, `[09:44] Bruno` |
| **RNF-21** | Observabilidade | Usar o **Pino** existente; nenhum logger novo. | `[09:29] Bruno` |
| **RNF-22** | Manutenibilidade | Seguir o padrão de módulos, schemas Zod e códigos de erro `WEBHOOK_*`. | `[09:27] Bruno`, `[09:29] Larissa`, `[09:30] Larissa` |

---

## 7. Especificações técnicas secundárias

### 7.1 Payload do evento

Formato JSON citado por Diego em `[09:43]`:

| Campo | Descrição discutida |
| --- | --- |
| `event_id` | Identificador único do evento (o mesmo UUID enviado em `X-Event-Id`) |
| `event_type` | `"order.status_changed"` |
| `timestamp` | ISO 8601 |
| `order_id` | Id do pedido |
| `order_number` | Número legível do pedido |
| `from_status` | Status anterior |
| `to_status` | Novo status |
| `customer_id` | Cliente dono do pedido |
| "campos básicos da order, tipo `total_cents`" | Lista exata não fechada (ver QA-04) |

- **Não inclui `items`.** Se o cliente quiser detalhes, consulta `GET /orders/:id`. `[09:43] Diego`
- **Snapshot renderizado na inserção da outbox.** `[09:52] Larissa`

### 7.2 Headers da requisição de entrega

| Header | Conteúdo | Origem |
| --- | --- | --- |
| `X-Event-Id` | UUID do evento, gerado ao entrar na outbox | `[09:25] Diego`, `[09:44] Diego` |
| `X-Signature` | Assinatura HMAC-SHA256 do corpo | `[09:20] Sofia`, `[09:44] Diego` |
| `X-Timestamp` | Timestamp **do envio** (permite ao cliente detectar replay attack) | `[09:44] Diego` |
| `X-Webhook-Id` | Id do endpoint de webhook, para clientes com vários cadastros saberem qual disparou | `[09:44] Sofia`, `[09:45] Diego` |
| `Content-Type` | `application/json` | `[09:44] Diego` |

### 7.3 Endpoints citados

| Método | Caminho | Finalidade | Autorização | Origem |
| --- | --- | --- | --- | --- |
| POST | (não definido) | Cadastrar webhook | Qualquer role autenticada | `[09:31] Marcos`, `[09:37] Sofia` |
| PATCH | (não definido) | Editar webhook | Qualquer role autenticada | `[09:33] Bruno` |
| DELETE | (não definido) | Remover webhook | Qualquer role autenticada | `[09:33] Bruno` |
| GET | (não definido) | Listar webhooks de um customer | Qualquer role autenticada | `[09:33] Bruno` |
| GET | `/webhooks/:id/deliveries` | Histórico de entregas | Qualquer role autenticada | `[09:34] Marcos` |
| (não definido) | (não definido) | Rotação de secret | Não discutido | `[09:21] Sofia` |
| POST | `/admin/webhooks/dead-letter/:id/replay` | Replay de evento da DLQ | **ADMIN** | `[09:18] Diego`, `[09:36] Sofia` |

> **(obs. do analista)** As rotas da API atual ficam sob o prefixo `/api/v1` (`src/app.ts`). A reunião citou os caminhos sem prefixo.

### 7.4 Estruturas de dados citadas

| Estrutura | Conteúdo discutido | Origem |
| --- | --- | --- |
| Tabela de configuração de webhook | `url`, `secret`, `customer_id`, estado ativo; lista de status de interesse | `[09:21] Bruno`, `[09:21] Sofia`, `[09:31] Marcos` |
| `webhook_outbox` (nome sugerido: "tipo webhook_outbox") | Evento com payload já renderizado; status pendente/processando/falhou/entregue; `created_at`; índices em status e `created_at`; id UUID | `[09:06] Diego`, `[09:08] Diego`, `[09:51] Larissa`, `[09:52] Larissa` |
| `webhook_dead_letter` | Payload, motivo da falha, timestamp | `[09:18] Diego` |
| Histórico de entregas | Sucesso/falha, payload, response, tempo de resposta (dados que o endpoint de deliveries precisa expor) | `[09:34] Marcos` |

### 7.5 Códigos de erro citados

| Código | Origem |
| --- | --- |
| `WEBHOOK_NOT_FOUND` | `[09:28] Bruno` |
| `WEBHOOK_INVALID_URL` | `[09:28] Bruno` |
| `WEBHOOK_SECRET_REQUIRED` | `[09:28] Bruno` |
| "etc." — demais códigos com o prefixo `WEBHOOK_` | `[09:28] Bruno`, `[09:29] Larissa` |

---

## 8. Integração com o código existente

Ganchos citados na reunião, confrontados com o código real.

| Ponto de integração | O que foi dito | Situação no código | Origem |
| --- | --- | --- | --- |
| `src/modules/orders/order.service.ts` — `changeStatus` | "A alteração crítica é dentro do service de orders, no método changeStatus." Inserir na outbox dentro da mesma transação, via `publishWebhookEvent(tx, order, fromStatus, toStatus)`. | `changeStatus` já roda em `this.prisma.$transaction(async (tx) => ...)`, possui `from`, `to` e `userId`, e declara `type TxClient = Prisma.TransactionClient`. | `[09:40] Bruno`, `[09:41] Bruno` |
| Transação atual do `changeStatus` | "Atualiza orders, insere na order_status_history, decrementa stock_quantity dos produtos do pedido." | **(obs. do analista)** No código, o débito de estoque ocorre só em `PENDING → PAID` e a reposição em `PAID/PROCESSING → CANCELLED` (`src/modules/orders/order.status.ts`); nas demais transições a transação atualiza `orders` e `order_status_history`. | `[09:04] Bruno`, `[09:40] Bruno` |
| `src/server.ts` | Modelo para o novo entry-point `src/worker.ts`. | Contém bootstrap, logger e graceful shutdown (`SIGINT`/`SIGTERM`). | `[09:11] Larissa` |
| `package.json` | Novo script `npm run worker`. | Scripts atuais: `dev`, `build`, `start`, `db:*`, `test`, `lint`, `format`. | `[09:11] Larissa` |
| `src/config/database.ts` | Worker com instância própria de PrismaClient. | `createPrismaClient()` já é uma factory exportada. | `[09:30] Bruno` |
| `src/modules/*` | Novo módulo `src/modules/webhooks` com controller, service, repository, routes, schemas. | Todos os módulos seguem essa estrutura. | `[09:27] Bruno` |
| `src/shared/errors/` (`app-error.ts`, `http-errors.ts`) | Reusar `AppError`; seguir o modelo de `InsufficientStockError` e `InvalidStatusTransitionError`. | Hierarquia existente com `statusCode`, `errorCode` e `details`. | `[09:28] Bruno` |
| `src/middlewares/error.middleware.ts` | Trata `AppError`, Zod e Prisma sem alteração. | Confirmado: traduz qualquer `AppError` pelo `statusCode`/`errorCode`. | `[09:29] Bruno` |
| `src/middlewares/auth.middleware.ts` — `requireRole` | Reaproveitar no replay com `ADMIN`. | `requireRole(...roles)` existe; hoje é usado em `GET /users/:id`. | `[09:36] Larissa` |
| `src/shared/logger/index.ts` | Pino já no projeto inteiro. | Logger Pino com redaction de `authorization`, `password`, `token` etc. | `[09:29] Bruno` |
| `*.schemas.ts` (Zod) | Validação de HTTPS no schema. | Todos os módulos validam entrada com Zod via `validate` middleware. | `[09:23] Sofia` |
| `GET /orders` e `GET /orders/:id` | Hoje usados em polling pelos clientes; `GET /orders/:id` continua sendo o caminho para detalhes do pedido. | Existem em `src/modules/orders/order.routes.ts`. | `[09:00] Marcos`, `[09:43] Diego` |
| Padrão de IDs | "Tudo é uuid." | Todos os models usam `@default(uuid())` em `CHAR(36)` (`prisma/schema.prisma`). | `[09:51] Larissa` |

> **(obs. do analista)** Sobre CTX-08 ("usuários que representam o cliente"): no `prisma/schema.prisma` atual não existe relação entre `User` e `Customer`; usuários têm apenas as roles `ADMIN` e `OPERATOR`. A reunião não discutiu como esse vínculo é feito. Isso é coerente com a decisão DT-12 (`customer_id` informado na requisição, não derivado do JWT).

---

## 9. Alternativas descartadas

Opções colocadas na mesa e **rejeitadas**, com o trade-off que motivou o descarte. Úteis para as seções "Alternativas consideradas" do RFC e dos ADRs.

| ID | Alternativa | Motivo do descarte | Origem |
| --- | --- | --- | --- |
| ALT-01 | **Disparo síncrono** do webhook dentro do service de orders | Transação já pesada; cliente lento travaria mudanças de status de outros pedidos; impossível dar rollback no status se o cliente estiver fora. "Síncrono está fora de questão." | `[09:04] Bruno`, `[09:06] Diego` |
| ALT-02 | **Redis Streams** (ou similar) como fila | Exige subir mais infraestrutura; time pequeno; "Subir Redis Cluster pra isso é overengineering." | `[09:07] Larissa`, `[09:07] Diego` |
| ALT-03 | **Trigger de banco** para acionar o worker de forma reativa | MySQL não tem `NOTIFY/LISTEN`; trigger não notifica processo externo; exigiria improviso (arquivo, endpoint). Polling de 2 s já atende. | `[09:09] Bruno`, `[09:09] Diego` |
| ALT-04 | **Worker dentro do processo da API** | Se a API reinicia, perde o worker. | `[09:11] Diego` |
| ALT-05 | **Retry indefinido** com backoff | Evento pode ficar pendurado para sempre se o cliente sumiu. | `[09:15] Diego` |
| ALT-06 | **3 tentativas** de retry | Pouco: esgotaria em ~30 min uma indisponibilidade matinal; já houve manutenção de 2 h. | `[09:16] Bruno`, `[09:16] Diego` |
| ALT-07 | **DLQ como status "failed" na própria outbox** | Tabela separada deixa a leitura da outbox mais limpa e guarda evidência para debug/reprocessamento. | `[09:17] Larissa`, `[09:18] Diego` |
| ALT-08 | **Secret global** da plataforma | "Se vaza uma, vaza tudo." | `[09:21] Sofia` |
| ALT-09 | **Truncar payload** acima do limite | Preferência por erro: "Se chegou nesse tamanho, tem algo errado." | `[09:23] Sofia`, `[09:24] Larissa` |
| ALT-10 | **Exactly-once** | Exigiria coordenação dos dois lados; muito mais complexo. | `[09:25] Diego` |
| ALT-11 | **`customer_id` implícito do JWT** | O JWT atual é do usuário operador, não do cliente. | `[09:31] Marcos`, `[09:32] Bruno`, `[09:32] Larissa` |
| ALT-12 | **Filtrar eventos no momento do envio** | Filtrar na inserção economiza linhas na outbox. | `[09:34] Diego`, `[09:34] Bruno` |
| ALT-13 | **Injetar o repository de webhook inteiro** no `OrderService` | Preferida uma função que recebe o `tx`. | `[09:41] Bruno`, `[09:41] Diego` |
| ALT-14 | **Incluir `items` no payload** | Inflaria o payload; detalhes via `GET /orders/:id`. | `[09:43] Diego` |
| ALT-15 | **Guardar só `order_id` e renderizar no envio** | O evento deixaria de refletir o estado do momento da mudança ("caso esquisito"). | `[09:51] Bruno`, `[09:52] Larissa` |
| ALT-16 | **ID auto incremental** na outbox | Seguir o padrão UUID do projeto. | `[09:51] Diego`, `[09:51] Larissa` |

---

## 10. Fora de escopo

Itens que **não** fazem parte desta entrega. Nenhum deles deve aparecer como requisito.

| ID | Item | Situação | Origem |
| --- | --- | --- | --- |
| FE-01 | **Webhooks de entrada (inbound)**: clientes enviando eventos para a plataforma | Descartado; só saída | `[09:02] Marcos`, `[09:03] Sofia` |
| FE-02 | **Notificação por e-mail** quando o webhook do cliente falha repetidamente (ex.: 3 falhas seguidas) | **Adiado** para a próxima fase | `[09:37] Marcos`, `[09:37] Larissa`, `[09:38] Marcos` |
| FE-03 | **Rate limiting de envio** para o cliente | **Não entra**; fica em observação ("observar e decidir depois") | `[09:38] Diego`, `[09:39] Diego`, `[09:39] Larissa` |
| FE-04 | **Dashboard / painel visual** para o cliente ver seus webhooks | Descartado nesta fase; "só endpoints". Painel é projeto separado do time de frontend | `[09:39] Marcos`, `[09:40] Larissa` |
| FE-05 | **Arquivamento das linhas entregues** da outbox (após ~30 dias) | Fora do escopo desta feature | `[09:08] Diego` |
| FE-06 | **Múltiplos workers em paralelo** e ordenação global | Não agora; "problema do futuro" | `[09:12] Diego`, `[09:13] Diego` |
| FE-07 | **Exactly-once** | Descartado em favor de at-least-once | `[09:25] Diego` |
| FE-08 | **Reprocessamento automático da DLQ** | Não previsto: o replay é **manual** via endpoint admin | `[09:18] Diego` |
| FE-09 | **Restrição de role no CRUD de configuração** | Não nesta fase: qualquer role autenticada; "mais pra frente a gente pode endurecer" | `[09:37] Sofia` |
| FE-10 | **Tipos de evento além de mudança de status** | **(obs. do analista)** Só `order.status_changed` foi discutido; nenhum outro evento foi mencionado | `[09:43] Diego` |

---

## 11. Evoluções futuras

Itens explicitamente sinalizados como possíveis próximos passos, com o gatilho mencionado.

| ID | Evolução | Gatilho / condição citada | Origem |
| --- | --- | --- | --- |
| EV-01 | Alerta por **e-mail** ao cliente quando o webhook falha repetidamente | "Talvez próxima fase, depois que a gente medir o impacto." | `[09:37] Larissa`, `[09:38] Marcos` |
| EV-02 | **Rate limiting** de saída | "A gente observa e implementa se virar problema." | `[09:39] Diego`, `[09:39] Larissa` |
| EV-03 | **Escalar para múltiplos workers** mantendo ordem por pedido | Particionar por `order_id` ou usar lock pessimista, quando for necessário escalar. | `[09:13] Bruno`, `[09:13] Diego` |
| EV-04 | **Endurecer permissões** do CRUD de configuração | "Mais pra frente a gente pode endurecer." | `[09:37] Sofia` |
| EV-05 | **Arquivamento** de eventos entregues após ~30 dias | Citado como atividade posterior, fora desta feature. | `[09:08] Diego` |
| EV-06 | **Painel visual** para clientes | Projeto separado do time de frontend. | `[09:40] Larissa` |

---

## 12. Limitações conhecidas aceitas

| ID | Limitação | Origem |
| --- | --- | --- |
| LIM-01 | Ordenação garantida **somente por `order_id` e com worker único**; sem ordenação global. "Documentamos como limitação conhecida." | `[09:13] Larissa` |
| LIM-02 | Latência de **2 s no pior caso** por causa do polling. | `[09:10] Larissa` |
| LIM-03 | Cliente **pode receber o mesmo evento mais de uma vez**; a deduplicação é responsabilidade dele. | `[09:24] Diego`, `[09:25] Sofia` |
| LIM-04 | Após ~15 h de falhas consecutivas, o evento vai para a DLQ e só volta por **replay manual**. | `[09:17] Diego`, `[09:18] Diego` |

---

## 13. Questões em aberto

Pontos levantados na reunião **sem decisão final**.

| ID | Questão | Estado | Origem |
| --- | --- | --- | --- |
| QA-01 | O `customer_id` nas rotas de configuração vai **no body ou no path**? | Larissa disse "no body ou no path", sem escolher | `[09:32] Larissa` |
| QA-02 | **Rate limiting** de saída: implementar ou não? | "Observar e decidir depois"; registrado como ponto em aberto | `[09:39] Diego`, `[09:39] Larissa` |
| QA-03 | Nome do arquivo de processamento: `webhook.worker.ts` **ou** `webhook.processor.ts`? | Ambos sugeridos, nenhum escolhido | `[09:28] Bruno` |
| QA-04 | Quais são exatamente os "campos básicos da order" no payload além de `total_cents`? | Só `total_cents` foi citado como exemplo | `[09:43] Diego` |
| QA-05 | Semântica da contagem de tentativas: são 5 tentativas **no total** ou 5 **retries após a primeira falha**? | Foram decididas "5 tentativas" com **5 intervalos** (1m/5m/30m/2h/12h), e Diego descreveu ~15 h "entre primeira falha e última tentativa". A leitura mais direta é 1 envio inicial + 5 retries, mas isso não foi dito explicitamente | `[09:17] Diego`, `[09:17] Larissa` |
| QA-06 | O histórico de entregas se limita aos **últimos 100** registros ou é paginado? | "Últimos 100" foi usado como exemplo ("Tipo...") | `[09:34] Marcos` |
| QA-07 | Quais outros códigos `WEBHOOK_*` compõem o catálogo? | Três citados + "etc." | `[09:28] Bruno` |
| QA-08 | Quando o CRUD de configuração terá permissões mais restritas? | "Por enquanto sim. Mais pra frente a gente pode endurecer." | `[09:37] Sofia` |

---

## 14. Lacunas não discutidas

**(obs. do analista)** Temas que um documento de implementação precisará definir e que **não foram tratados** na reunião. Não são decisões; devem ser levados ao time antes de entrar em PRD/FDD como definição.

| ID | Lacuna |
| --- | --- |
| LAC-01 | Caminho, método e contrato do endpoint de **rotação de secret** (só a existência e o grace period foram definidos). |
| LAC-02 | Caminhos exatos dos endpoints de CRUD de configuração (só os verbos foram citados). |
| LAC-03 | Como a assinatura se comporta **durante as 24 h de convivência** de duas secrets (assinar com qual? enviar duas assinaturas?). |
| LAC-04 | Formato da assinatura em `X-Signature` (hex/base64, prefixo) e se o `X-Timestamp` entra no cálculo do HMAC; a reunião disse apenas "sobre o corpo do request". |
| LAC-05 | Como a secret é armazenada e se volta a ser exibida depois da criação. |
| LAC-06 | Quais respostas HTTP do cliente contam como **sucesso** (ex.: faixa 2xx) e quais contam como falha, além do timeout. |
| LAC-07 | O que acontece com eventos pendentes quando um webhook é **desativado ou removido**. |
| LAC-08 | Recuperação de eventos presos em "processando" se o worker cair no meio do envio. |
| LAC-09 | Onde e como o histórico de entregas é persistido (tabela própria ou derivado da outbox/DLQ). |
| LAC-10 | Se o replay de DLQ **reinicia** o contador de tentativas. |
| LAC-11 | Métricas, alertas e tracing do worker (a reunião só definiu o uso do Pino). |
| LAC-12 | Métricas de sucesso da feature (ex.: taxa de entrega, redução de polling em `GET /orders`); a única meta quantitativa citada é < 10 s. |
| LAC-13 | Onde e quando validar o limite de 64 KB (na renderização do evento, na inserção ou no envio). |
| LAC-14 | Como vincular "usuários que representam o cliente" a um `customer` (ver obs. na seção 8). |

---

## 15. Planejamento, prazo e riscos

### 15.1 Estimativa

Estimativa de Larissa em `[09:46]`:

| Bloco | Esforço |
| --- | --- |
| Modelagem de outbox e DLQ | 1 sprint |
| Worker e retry | 1 sprint |
| CRUD de configuração e deliveries | ½ sprint |
| Integração no `order.service` e testes ponta a ponta | ½ sprint |
| HMAC, schemas, validações | "mais um pouco" |
| **Total** | **3 sprints, incluindo a revisão da Sofia no fim** (`[09:47] Larissa`) |

### 15.2 Prazo e marcos

| Item | Detalhe | Origem |
| --- | --- | --- |
| Prazo do cliente | Atlas quer **até o fim de novembro** | `[09:45] Marcos` |
| Pressão comercial | Atlas pode migrar para concorrente se não houver entrega até o **fim do trimestre** | `[09:00] Marcos` |
| Revisão de segurança | **Mínimo de 2 dias úteis** antes do deploy; foco em **HMAC e geração de secret** | `[09:46] Sofia`, `[09:49] Sofia` |

### 15.3 Riscos mencionados

| ID | Risco | Mitigação discutida | Origem |
| --- | --- | --- | --- |
| RSK-01 | Perda de cliente (Atlas) por atraso | Prazo de 3 sprints alinhado ao fim de novembro; PM confirma com o cliente | `[09:00] Marcos`, `[09:47] Marcos` |
| RSK-02 | Cliente lento travando mudanças de status | Outbox + worker assíncrono | `[09:04] Bruno` |
| RSK-03 | Inconsistência entre status e evento | Outbox na mesma transação, com rollback conjunto | `[09:06] Diego`, `[09:41] Diego` |
| RSK-04 | Acúmulo de eventos deixando o worker lento | Índices em status e `created_at`; batch pequeno; arquivamento futuro | `[09:07] Bruno`, `[09:08] Diego` |
| RSK-05 | Vazamento de secret | Secret por endpoint + rotação com grace period de 24 h | `[09:21] Sofia`, `[09:22] Diego` |
| RSK-06 | Adulteração ou falsificação de requisições | HMAC-SHA256 + TLS obrigatório | `[09:19] Sofia`, `[09:23] Sofia` |
| RSK-07 | Replay attack no cliente | `X-Timestamp` para detecção no cliente | `[09:44] Diego` |
| RSK-08 | Entregas duplicadas | `X-Event-Id` + documentação no portal | `[09:25] Diego`, `[09:26] Marcos` |
| RSK-09 | Indisponibilidade prolongada do cliente | Backoff de ~15 h + DLQ + replay manual | `[09:16] Diego`, `[09:18] Diego` |
| RSK-10 | Perda de ordenação ao escalar workers | Manter worker único; particionar por `order_id` ou lock pessimista no futuro | `[09:12] Diego`, `[09:13] Diego` |
| RSK-11 | Bombardeio de chamadas ao cliente (ex.: 50 pedidos em 1 min) | Observar; rate limiting se virar problema | `[09:38] Diego`, `[09:39] Diego` |
| RSK-12 | Payload anormalmente grande | Limite de 64 KB com erro | `[09:23] Sofia`, `[09:24] Diego` |

---

## 16. Itens de ação

| ID | Ação | Responsável | Quando | Origem |
| --- | --- | --- | --- | --- |
| AC-01 | Abrir o documento de design da feature | Larissa | Após a reunião | `[09:50] Larissa` |
| AC-02 | Marcar sessão de revisão do design com Bruno e Diego antes de começar a codar | Larissa | Antes do início da implementação | `[09:50] Larissa` |
| AC-03 | Confirmar o prazo com a Atlas | Marcos | Após a reunião | `[09:47] Marcos` |
| AC-04 | Atualizar os clientes | Marcos | "Hoje à tarde" | `[09:49] Marcos` |
| AC-05 | Documentar a semântica at-least-once / `X-Event-Id` em destaque no portal de desenvolvedor | Marcos | Não definido | `[09:26] Marcos` |
| AC-06 | Documentar no portal como integrar via API | Marcos | Não definido | `[09:40] Marcos` |
| AC-07 | Agendar a revisão de segurança (mín. 2 dias úteis) antes do deploy | Time (lembrete de Sofia) | Antes do deploy | `[09:46] Sofia`, `[09:49] Sofia` |

---

## 17. Glossário

| Termo | Significado no contexto |
| --- | --- |
| **Outbound webhook** | Chamada HTTP da plataforma para uma URL do cliente quando algo acontece |
| **Outbox** | Tabela onde o evento é gravado na mesma transação da mudança de negócio, para envio posterior por um worker |
| **Worker** | Processo Node separado da API que lê a outbox e envia os webhooks |
| **Polling** | Consulta periódica (aqui, a cada 2 s) à outbox em busca de eventos pendentes |
| **Backoff exponencial** | Intervalos crescentes entre tentativas (1m, 5m, 30m, 2h, 12h) |
| **DLQ (Dead Letter Queue)** | Tabela `webhook_dead_letter` para eventos que esgotaram as tentativas |
| **Replay** | Recolocar manualmente um evento da DLQ na outbox como pendente |
| **HMAC-SHA256** | Assinatura do corpo com secret compartilhada, permitindo ao cliente verificar origem e integridade |
| **Grace period** | Janela de 24 h em que a secret antiga e a nova valem juntas após uma rotação |
| **At-least-once** | Garantia de que o evento chega pelo menos uma vez, podendo chegar duplicado |
| **`X-Event-Id`** | UUID do evento usado pelo cliente para deduplicação |
| **Snapshot** | Payload renderizado e congelado no momento da mudança de status |
| **`publishWebhookEvent`** | Função proposta para enfileirar o evento na outbox usando o `tx` da transação corrente |

---

## 18. Guia de uso: o que vai para cada documento

Mapa para quem for produzir o pacote de design docs a partir desta ata.

| Documento | Pergunta que responde | Seções desta ata a consumir |
| --- | --- | --- |
| **PRD** | Por que e o quê? | §3 Contexto, §5 Requisitos funcionais, §6 RNFs (visão de produto), §10 Fora de escopo, §11 Evoluções, §15 Prazo e riscos, §12 Limitações (para comunicação ao cliente) |
| **RFC** | Como pretendemos resolver e o que está em aberto? | §2 Resumo, §4.1 Decisões principais (visão geral), §9 Alternativas descartadas, §13 Questões em aberto, §15.3 Riscos |
| **ADRs** | Por que decidimos exatamente assim? | Um ADR por decisão de §4.1 (DT-01 a DT-06), com alternativas de §9 e consequências/limitações de §12. Decisões de §4.2 podem virar ADRs adicionais ou ficar no FDD |
| **FDD** | Como construir em detalhe? | §4.2 Decisões complementares, §7 Especificações (payload, headers, endpoints, tabelas, erros), §8 Integração com o código, §14 Lacunas (a resolver antes de detalhar) |
| **Tracker** | De onde veio cada coisa? | Todos os IDs desta ata já trazem `[hh:mm] Nome` ou caminho de arquivo |

Correspondência entre as decisões desta ata e as seis decisões principais esperadas:

| Decisão principal | ID nesta ata |
| --- | --- |
| Padrão Outbox no MySQL | DT-01 |
| Política de retry com backoff e DLQ | DT-03 |
| Autenticação HMAC-SHA256 com secret por endpoint | DT-04 |
| Garantia at-least-once com `X-Event-Id` | DT-05 |
| Worker em processo separado em polling | DT-02 |
| Reuso dos padrões existentes do projeto | DT-06 |
