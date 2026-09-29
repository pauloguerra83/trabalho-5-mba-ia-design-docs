# Prompt — Geração do FDD do Sistema de Webhooks

## Como usar

| Item | Descrição |
| --- | --- |
| Objetivo | Gerar o FDD (Feature Design Document) da feature de Webhooks de Notificação de Pedidos em `docs/FDD.md` |
| Quando usar | Para produzir o FDD pela primeira vez ou regenerá-lo depois de mudanças na ata, nas ADRs ou na RFC |
| Pré-requisitos | Existirem `docs/levantamento-tecnico/ATA-REUNIAO-WEBHOOKS.md`, `docs/levantamento-tecnico/LEVANTAMENTO-TECNICO.md`, as ADRs em `docs/adrs/` e `docs/RFC.md` (ver `PROMPT-ADRS.md` e `PROMPT-RFC.md`) |
| Execução | Abrir uma sessão de IA com acesso de leitura e escrita ao repositório e colar **todo o conteúdo abaixo da linha** |
| Resultado esperado | `docs/FDD.md` acionável: um desenvolvedor consegue começar a codar a partir dele |

---

# Contexto

Você vai produzir o **FDD** da feature **Sistema de Webhooks de Notificação de Pedidos** de um Order Management System (Node.js 20 + TypeScript, Express, Prisma, MySQL, Zod, Pino). As decisões de arquitetura estão nas ADRs e na RFC. O FDD é o documento **mais técnico** do pacote e responde **"como construir, em detalhe"**: fluxos, contratos, erros, resiliência, observabilidade e integração com o código existente.

Leia as fontes e o código **antes** de escrever. Não repita o "porquê" das decisões: aponte para as ADRs.

# Fontes do conhecimento (em ordem de prioridade)

1. `docs/levantamento-tecnico/ATA-REUNIAO-WEBHOOKS.md`: requisitos (RF/RNF), decisões (DT), especificações secundárias (payload, headers, endpoints, tabelas, códigos de erro citados), fora de escopo (FE), questões em aberto (QA) e lacunas (LAC).
2. `docs/adrs/ADR-001` a `ADR-006` e `docs/RFC.md`: decisões fechadas e visão de arquitetura.
3. Código real: `src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`, `src/shared/errors/*`, `src/middlewares/*`, `src/app.ts`, `src/routes/index.ts`, `src/server.ts`, `src/config/*`, `src/shared/logger/index.ts`, `src/shared/http/response.ts`, `prisma/schema.prisma`, `package.json` e `tests/*`.
4. `docs/levantamento-tecnico/LEVANTAMENTO-TECNICO.md`: padrões e convenções do projeto.
5. `TRANSCRICAO.md`: apenas para validar algo que não esteja claro na ata.

# Saída

- Arquivo: `docs/FDD.md` (sobrescrever o conteúdo existente).
- Idioma: português, com termos técnicos em inglês.
- Metadados em tabela:
  - Autora: Larissa (Tech Lead).
  - Revisores: Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança).
  - Status: Rascunho para revisão.
  - Data.
  - Links: RFC, ADRs, ata, levantamento técnico.

# Estrutura obrigatória (use estes títulos)

1. **Contexto e motivação técnica:** problema e restrições técnicas; tabela bloco → decisão → ADR.
2. **Objetivos técnicos:** tabela de metas verificáveis (latência de 2 s, atomicidade, at-least-once, isolamento, retry, timeout de 10 s, 64 KB, HMAC/HTTPS, nenhuma dependência nova).
3. **Escopo e exclusões:** incluído e excluído (FE-xx da ata).
4. **Modelo de dados** (apoio): models Prisma no padrão de `prisma/schema.prisma`:
   - UUID `CHAR(36)`, `@@map` snake_case, índices;
   - endpoint de webhook, outbox (com status pendente/processando/entregue/falhou, tentativas, próxima tentativa, lock), tentativas de entrega e dead letter;
   - relações inversas em `Customer` e `User`.
5. **Fluxos detalhados:** diagramas de sequência Mermaid e regras para:
   - (a) criação do evento na outbox dentro da transação de `changeStatus`, via `publishWebhookEvent(tx, order, fromStatus, toStatus)`, com filtro na inserção e snapshot;
   - (b) processamento pelo worker (polling de 2 s, lote pequeno, ordem por `createdAt`, envio, registro da tentativa);
   - (c) retry (tabela com os intervalos 1m/5m/30m/2h/12h e o cálculo de `nextAttemptAt`);
   - (d) DLQ e replay por ADMIN.
6. **Contratos públicos:** convenções herdadas (`/api/v1`, JWT, camelCase, envelope de erro e de paginação). Para **cada endpoint**:
   - método e caminho;
   - **request de exemplo**;
   - **response de exemplo**;
   - tabela de **status codes**;
   - semântica.

   Endpoints:
   - cadastrar;
   - listar por cliente;
   - consultar;
   - editar;
   - remover;
   - rotacionar secret;
   - histórico de entregas (`/webhooks/:id/deliveries`);
   - replay (`/admin/webhooks/dead-letter/:id/replay`).

   Inclua também o **contrato de saída** (POST ao cliente), com os headers `X-Event-Id`, `X-Webhook-Id`, `X-Signature`, `X-Timestamp` e `Content-Type`, e o payload JSON em snake_case com os campos definidos na reunião, sem `items`. Mínimo: 4 endpoints com exemplos.
7. **Matriz de erros previstos:** apenas códigos com prefixo `WEBHOOK_`, em duas tabelas:
   - erros da API: código, HTTP, classe e quando ocorre;
   - erros de entrega do worker: código, quando ocorre, se gera retry e destino final.

   Inclua os códigos citados na reunião (`WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) e indique os erros existentes reutilizados (`VALIDATION_ERROR`, `UNAUTHORIZED`, `FORBIDDEN`).
8. **Estratégias de resiliência:** isolamento, timeout (`AbortSignal.timeout`), retry/backoff persistido, classificação de falhas, fallback (DLQ + replay), recuperação de eventos presos, idempotência, ordenação (com a limitação durante retries), graceful shutdown.
9. **Observabilidade:** três subseções obrigatórias:
   - **Logs:** mensagens Pino em snake_case com campos e redaction da secret;
   - **Métricas:** nomes, tipos e fonte provisória, já que o projeto não tem stack de métricas;
   - **Tracing:** `event_id` como chave ponta a ponta, ligação com `X-Request-Id`, tracing distribuído como evolução.
10. **Integração com o sistema existente** (obrigatória): tabela com **pelo menos 4 caminhos de arquivo reais** (com link relativo `../src/...`), o tipo de mudança (alteração, reuso, sem alteração, modelo) e **como** o módulo se integra com cada um. Ao menos:
    - `order.service.ts`: onde e como `changeStatus` chama `publishWebhookEvent` com o `tx`;
    - `shared/errors/*`: como as classes são reutilizadas, observando que `NotFoundError` fixa o código `NOT_FOUND`;
    - `error.middleware.ts`: sem alteração;
    - `auth.middleware.ts`: `requireRole`;
    - `validate.middleware.ts`;
    - `app.ts` e `routes/index.ts`;
    - `server.ts`: modelo do `src/worker.ts`;
    - `config/database.ts` e `config/env.ts`;
    - `logger`;
    - `prisma/schema.prisma`;
    - `package.json`;
    - `tests/*`.

    Inclua a árvore dos novos arquivos, marcados "(a criar)", esqueletos curtos de código e a tabela de novas variáveis de ambiente.
11. **Dependências e compatibilidade:** bibliotecas (nenhuma nova: `fetch` nativo, `node:crypto`), migration aditiva, compatibilidade da API, deploy de dois processos, ordem de deploy e rollback.
12. **Critérios de aceite técnicos:** tabela ID → critério → como verificar (testes no padrão Vitest + Supertest).
13. **Riscos e mitigação:** riscos técnicos de implementação com impacto e mitigação. **Não invente probabilidade.**
14. **Decisões de implementação propostas — pendentes de validação:** ver a regra 2.
15. **Referências.**

# Regras de conteúdo

1. **Não invente decisões.** O que vem da ata, das ADRs ou do código é tratado como fato.
2. **Propor e marcar.** Pontos que a reunião **não** decidiu (QA-xx e LAC-xx da ata, além de escolhas técnicas necessárias para o código funcionar) devem receber uma definição **coerente com o código existente**, marcada no texto com **🔶 Proposta** e consolidada na seção 14, numa tabela com: proposta, ID da ata e papel que valida (Pedidos, Plataforma, Segurança, Produto). Exemplos:
   - `customer_id` no body ou no path;
   - caminhos do CRUD;
   - formato do `X-Signature`;
   - assinatura durante a rotação;
   - critério de sucesso HTTP;
   - eventos presos em processamento;
   - persistência do histórico;
   - replay e `event_id`;
   - onde validar os 64 KB;
   - semântica das 5 tentativas;
   - tamanho do lote;
   - granularidade da outbox.

   Os campos do payload além de `total_cents` (QA-04) permanecem em aberto; não os invente.
3. **Coerência com o código.** Antes de propor, verifique como o código se comporta. Exemplos:
   - `validate.middleware.ts` converte qualquer erro Zod em `VALIDATION_ERROR`, então um código `WEBHOOK_*` de validação precisa ser lançado no service;
   - `NotFoundError` não aceita código customizado.
4. **Sem rastreio de fala.** Não cite horários nem atribua falas (`[hh:mm] Nome`, "fulano disse"). Nomes só nos metadados e na coluna de validação por papel.
5. **Sem repetir as ADRs.** Aponte para elas quando o assunto for "por quê".
6. **Caminhos reais.** Link relativo apenas para arquivos existentes; arquivos novos marcados "(a criar)".
7. **Fora de escopo** (e-mail, rate limiting, dashboard, webhooks de entrada, arquivamento, múltiplos workers, exactly-once) nunca aparece como algo a implementar.
8. **Exemplos realistas:** UUIDs, `orderNumber` no formato `ORD-000123`, valores em centavos e domínios `example.com`.

# Checklist de aceite

- [ ] `docs/FDD.md` existe e está em Markdown.
- [ ] Contém as seções: Contexto e motivação técnica; Objetivos técnicos; Escopo e exclusões; Fluxos detalhados (outbox, worker, retry, DLQ); Contratos públicos; Matriz de erros previstos; Estratégias de resiliência; Observabilidade; Dependências e compatibilidade; Critérios de aceite técnicos; Riscos e mitigação; **Integração com o sistema existente**.
- [ ] "Contratos públicos" inclui pelo menos 4 endpoints HTTP com payload de exemplo (request e response) e status codes.
- [ ] A matriz de erros usa códigos com prefixo `WEBHOOK_`.
- [ ] "Integração com o sistema existente" referencia pelo menos 4 caminhos de arquivo reais e descreve como o módulo se integra com cada um.
- [ ] "Observabilidade" cita métricas, logs e tracing.
- [ ] Toda definição não decidida na reunião está marcada 🔶 e listada na seção 14 com o ID da ata.
- [ ] Nenhum `[hh:mm]` no documento.

# Verificação obrigatória antes de concluir

Execute e reporte o resultado:

1. Buscar os cabeçalhos das 12 seções obrigatórias.
2. Contar, em "Contratos públicos", os endpoints (`### 6.x`) e confirmar que cada um tem bloco `json` de request (quando houver corpo), de response e tabela de status.
3. Extrair os códigos da matriz (`WEBHOOK_[A-Z_]+`) e confirmar que todos têm o prefixo.
4. Extrair os links `../src/`, `../prisma/`, `../tests/` e `../package.json` e confirmar que cada arquivo existe (pelo menos 4 na seção de integração).
5. Confirmar as subseções Logs, Métricas e Tracing em "Observabilidade".
6. Buscar `\[[0-9]{2}:[0-9]{2}\]` e confirmar zero ocorrências.
7. Confirmar que cada QA/LAC citado na seção 14 existe na ata.

Se o shell não estiver disponível, faça as mesmas verificações com ferramentas de busca de arquivos e de conteúdo.

# Formato da resposta final

1. Link para `docs/FDD.md` e resumo de uma linha por seção.
2. Resultado de cada verificação acima.
3. Lista das propostas 🔶 que precisam de validação, agrupadas por quem valida.
