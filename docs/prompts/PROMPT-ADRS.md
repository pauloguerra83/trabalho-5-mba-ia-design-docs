# Prompt — Geração das ADRs do Sistema de Webhooks

## Como usar

| Item | Descrição |
| --- | --- |
| Objetivo | Gerar os ADRs (Architecture Decision Records) da feature de Webhooks de Notificação de Pedidos em `docs/adrs/` e o índice da pasta |
| Quando usar | Para produzir as ADRs pela primeira vez ou regenerá-las depois de mudanças na ata ou na transcrição |
| Pré-requisitos | Existirem `TRANSCRICAO.md`, `docs/levantamento-tecnico/ATA-REUNIAO-WEBHOOKS.md` e `docs/levantamento-tecnico/LEVANTAMENTO-TECNICO.md` |
| Execução | Abrir uma sessão de IA com acesso de leitura e escrita ao repositório e colar **todo o conteúdo abaixo da linha** |
| Resultado esperado | 6 arquivos `docs/adrs/ADR-NNN-*.md` e `docs/adrs/README.md` reescrito como índice |

---

# Contexto

Você vai produzir os **ADRs (Architecture Decision Records)** da feature **Sistema de Webhooks de Notificação de Pedidos** de um Order Management System (Node.js + TypeScript, Express, Prisma, MySQL). As decisões foram tomadas numa reunião técnica registrada em `TRANSCRICAO.md` e já estruturadas numa ata.

Deve existir **um ADR para cada decisão arquitetural isolada**, com contexto e consequências. Cada ADR responde à pergunta:

> _Por que decidimos exatamente assim?_

Não assuma nada que não esteja nas fontes. Leia os arquivos antes de escrever.

# Fontes do conhecimento (em ordem de prioridade)

1. `docs/levantamento-tecnico/ATA-REUNIAO-WEBHOOKS.md`: fonte principal. Contém as decisões (DT-xx), alternativas descartadas (ALT-xx), fora de escopo (FE-xx), limitações (LIM-xx), questões em aberto (QA-xx) e lacunas (LAC-xx), todas com origem `[hh:mm] Nome`.
2. `TRANSCRICAO.md`: use para **validar** as decisões e para completar o que faltar na ata. Se algo não estiver na ata, confirme aqui antes de usar.
3. `docs/levantamento-tecnico/LEVANTAMENTO-TECNICO.md`: arquitetura e padrões do código existente.
4. Código-fonte em `src/` e `prisma/`: para citar arquivos, módulos e classes reais.

# Pasta e nome dos arquivos

- Pasta: `docs/adrs/`
- Nome: `ADR-{NNN}-{titulo-curto-da-decisao}.md`, com número de 3 dígitos e título em kebab-case, sem acentos.
- Exemplo: `ADR-001-outbox-no-mysql.md`
- A pasta deve conter **entre 5 e 8** arquivos nesse formato.

# ADRs a produzir

Gere **6 ADRs**, na ordem abaixo, que é a ordem de dependência entre as decisões. Os IDs referem-se à ata.

| Nº | Arquivo sugerido | Decisão | Decisões da ata a cobrir | Alternativas a usar |
| --- | --- | --- | --- | --- |
| 001 | `ADR-001-outbox-no-mysql.md` | Padrão Outbox no MySQL | DT-01, DT-08 (`publishWebhookEvent(tx, …)`), DT-09 (filtro na inserção), DT-10 (snapshot), DT-11 (UUID) | ALT-01 síncrono, ALT-02 Redis Streams, ALT-12 filtrar no envio, ALT-15 renderizar no envio |
| 002 | `ADR-002-worker-separado-com-polling.md` | Worker em processo separado em polling | DT-02, DT-07 (worker único, ordem por `order_id`) | ALT-03 trigger de banco, ALT-04 worker no processo da API, FE-06 múltiplos workers |
| 003 | `ADR-003-retry-backoff-exponencial-e-dlq.md` | Política de retry com backoff e DLQ | DT-03, DT-13 (replay só ADMIN), DT-14 (timeout 10 s como falha) | ALT-05 retry indefinido, ALT-06 três tentativas, ALT-07 DLQ como status na outbox |
| 004 | `ADR-004-hmac-sha256-secret-por-endpoint.md` | Autenticação HMAC-SHA256 com secret por endpoint | DT-04, DT-16 (HTTPS via Zod), `X-Timestamp` | ALT-08 secret global; secret fixa sem rotação (justificativa na DT-04 da ata) |
| 005 | `ADR-005-entrega-at-least-once-com-x-event-id.md` | Garantia at-least-once com `X-Event-Id` | DT-05 | ALT-10 exactly-once |
| 006 | `ADR-006-reuso-dos-padroes-existentes.md` | Reuso dos padrões existentes do projeto | DT-06, DT-11, DT-12 (`customer_id` fora do JWT), DT-13 | Padrões/ferramentas próprios, PrismaClient compartilhado entre API e worker (ambos discutidos na DT-06/DT-02 da ata), ALT-11 `customer_id` no JWT |

Decisões secundárias (formato do payload, headers, limite de 64 KB, timeout) **não** ganham ADR próprio. Elas entram como fatores dentro do ADR relacionado; o detalhamento é responsabilidade do FDD.

# Estrutura de cada ADR

Use exatamente este template, em português:

```markdown
# ADR-NNN — <Título da decisão>

| Campo | Valor |
| --- | --- |
| Status | Aceito |
| Data | Reunião técnica de quinta-feira, 09:00 (data do calendário não registrada) |
| Decisores | <papéis responsáveis pela decisão, com o nome entre parênteses. Ex.: Tech Lead (Larissa), Segurança (Sofia)> |
| Relacionados | <links para os outros ADRs relacionados> |

## Status
<"Aceito na reunião técnica da feature e confirmado no resumo de encerramento." — sem horários nem nomes>

## Contexto
<o problema e as restrições que moldaram a decisão>

## Decisão
<o que foi decidido e os fatores que influenciaram a escolha>
> **Justificativa principal:** <uma frase>

## Alternativas Consideradas
### A1. <alternativa>
- **Prós:** …
- **Contras:** …
- **Descarte:** <motivo>

## Consequências
**Positivas** …
**Negativas e trade-offs** …
**Riscos e impactos operacionais** …
**Pontos em aberto** (da ata, sem decisão) <itens com ID QA-xx / LAC-xx>

## Referências
- Código: <links relativos para arquivos reais>
- Detalhamento e rastreabilidade: <link para a ata> — <IDs DT/ALT/FE/LIM usados>
```

O que cada seção deve conter:

- **Status:** estado da decisão.
- **Contexto:** o problema e as restrições.
- **Decisão:** a escolha, os fatores que influenciaram e a justificativa principal em destaque.
- **Alternativas Consideradas:** prós e contras de cada alternativa e o motivo do descarte.
- **Consequências:** impactos técnicos, operacionais e riscos.

# Regras de conteúdo

1. **Não invente.** Toda decisão, restrição, número ou alternativa precisa ter origem na ata, na transcrição ou no código.
2. **ADR não é rastreio da conversa:** não cite pessoas nem horários no corpo do ADR (nada de `[hh:mm] Nome` nem "fulano disse"). Os responsáveis aparecem **apenas** na linha `Decisores` da tabela de metadados, de forma macro (papel + nome). Quando uma frase da reunião for essencial, parafraseie ou use aspas sem autor. O rastreio fala a fala fica na ata e no Tracker; nas Referências, aponte para a ata pelos IDs (DT, ALT etc.).
3. **Código real:** cite apenas caminhos que existem no repositório, com links relativos (ex.: `../../src/modules/orders/order.service.ts`). Arquivos que a feature ainda vai criar (`src/worker.ts`, `src/modules/webhooks`) devem ser marcados como **"(a criar)"**, sem link.
4. **Pelo menos 1 ADR** deve referenciar explicitamente arquivos, módulos ou classes do código base. Referências esperadas:
   - ADR-001: `src/modules/orders/order.service.ts` (`changeStatus`, `prisma.$transaction`) e `prisma/schema.prisma`.
   - ADR-002: `src/server.ts`, `src/config/database.ts` (`createPrismaClient`) e `package.json`.
   - ADR-003: `requireRole` em `src/middlewares/auth.middleware.ts`.
   - ADR-004: schemas Zod e `src/shared/logger/index.ts`.
   - ADR-006: `src/shared/errors/`, `src/middlewares/error.middleware.ts`, `src/shared/logger/index.ts`, `src/app.ts` e `src/routes/index.ts`.
5. **Análise derivada:** prós, contras e consequências podem conter análise derivada das decisões, desde que coerente com as fontes. Nunca apresente como decidido algo que a reunião não decidiu.
6. **Pontos em aberto:** o que a reunião não decidiu vai em "Pontos em aberto", com o ID da ata (QA-xx, LAC-xx). Não resolva essas questões no ADR.
7. **Fora de escopo:** itens descartados ou adiados (e-mail de alerta, dashboard, rate limiting, webhooks de entrada, exactly-once, arquivamento, múltiplos workers) podem aparecer como alternativa descartada, risco ou evolução futura, **nunca como decisão**.
8. **Concisão:** cerca de 1 a 1,5 página por ADR. Sem schema de tabela, sem pseudocódigo, sem detalhe de implementação de baixo nível. Faça apenas o que é responsabilidade de uma ADR.
9. **Idioma:** português, com termos técnicos consagrados em inglês (outbox, worker, backoff, DLQ, at-least-once).

# README da pasta `docs/adrs/`

Reescreva `docs/adrs/README.md` como índice, contendo:

- a convenção de nome `ADR-NNN-titulo-em-kebab-case.md` e as seções obrigatórias;
- a regra de que os ADRs registram decisão, justificativa e consequências, com responsáveis apenas de forma macro, e que o rastreio fala a fala fica na ata e no Tracker (pontos em aberto referenciam os IDs QA/LAC da ata);
- uma tabela com número (com link), decisão e status de cada ADR;
- links para as fontes (transcrição, ata e levantamento técnico).

# Checklist de aceite

- [ ] A pasta `docs/adrs/` contém entre 5 e 8 arquivos no formato `ADR-NNN-titulo-em-kebab-case.md`.
- [ ] Cada ADR contém as seções Status, Contexto, Decisão, Alternativas Consideradas e Consequências:
  - Contexto descreve o problema e as restrições;
  - Decisão inclui os fatores que influenciaram a escolha e a justificativa principal;
  - Alternativas Consideradas inclui prós e contras de cada alternativa;
  - Consequências cobre impactos técnicos, operacionais e riscos.
- [ ] O conjunto cobre pelo menos 5 das 6 decisões principais: Outbox no MySQL; retry com backoff e DLQ; HMAC-SHA256 com secret por endpoint; at-least-once com `X-Event-Id`; worker em processo separado em polling; reuso dos padrões existentes.
- [ ] Os ADRs são concisos e objetivos, sem detalhes de implementação de baixo nível.
- [ ] Pelo menos 1 ADR referencia explicitamente arquivos, módulos ou classes do código base.
- [ ] Cada ADR tem pelo menos 1 alternativa considerada, com trade-off explícito.
- [ ] Nenhuma citação no formato `[hh:mm] Nome` no corpo dos ADRs; nomes de participantes só na linha `Decisores`.
- [ ] Todos os caminhos de código citados existem, exceto os marcados "(a criar)".
- [ ] Nenhum item fora de escopo aparece como decisão.
- [ ] `docs/adrs/README.md` foi reescrito como índice.

# Verificação obrigatória antes de concluir

Execute e reporte o resultado:

1. Listar `docs/adrs/` e confirmar que os arquivos seguem o regex `^ADR-[0-9]{3}-[a-z0-9-]+\.md$`.
2. Contar, por arquivo, as linhas `^## (Status|Contexto|Decisão|Alternativas Consideradas|Consequências)$`. O esperado é 5.
3. Buscar `\[[0-9]{2}:[0-9]{2}\]` em `docs/adrs/` e confirmar **zero** ocorrências; buscar `Larissa|Marcos|Bruno|Diego|Sofia` e confirmar que só aparecem nas linhas `Decisores`.
4. Extrair os links `../../...` dos ADRs e confirmar que cada caminho existe no repositório.
5. Buscar os termos fora de escopo (e-mail, dashboard, rate limiting, exactly-once, inbound) e confirmar que só aparecem como alternativa, risco ou evolução.

Se o shell não estiver disponível, faça as mesmas verificações com ferramentas de busca de arquivos e de conteúdo.

# Formato da resposta final

1. Tabela com os arquivos gerados (link e decisão).
2. Resultado de cada verificação acima, com as falhas encontradas e corrigidas.
3. Pontos em aberto que ficaram registrados nos ADRs, para serem resolvidos no FDD.
