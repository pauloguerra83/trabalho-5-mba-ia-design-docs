# Prompt — Geração da RFC do Sistema de Webhooks

## Como usar

| Item | Descrição |
| --- | --- |
| Objetivo | Gerar a RFC (Request for Comments) da feature de Webhooks de Notificação de Pedidos em `docs/RFC.md` |
| Quando usar | Para produzir a RFC pela primeira vez ou regenerá-la depois de mudanças na ata ou nas ADRs |
| Pré-requisitos | Existirem `docs/levantamento-tecnico/ATA-REUNIAO-WEBHOOKS.md`, `docs/levantamento-tecnico/LEVANTAMENTO-TECNICO.md` e as ADRs em `docs/adrs/` (ver `docs/prompts/PROMPT-ADRS.md`) |
| Execução | Abrir uma sessão de IA com acesso de leitura e escrita ao repositório e colar **todo o conteúdo abaixo da linha** |
| Resultado esperado | `docs/RFC.md` com 2 a 4 páginas, em nível de arquitetura, pronto para revisão da equipe |

---

# Contexto

Você vai produzir a **RFC** da feature **Sistema de Webhooks de Notificação de Pedidos** de um Order Management System (Node.js + TypeScript, Express, Prisma, MySQL). As decisões foram tomadas numa reunião técnica (`TRANSCRICAO.md`), estruturadas numa ata e registradas em ADRs.

A RFC é a **proposta técnica submetida à equipe para revisão**. Ela opera em **nível de arquitetura**: apresenta a abordagem escolhida, as alternativas que foram colocadas na mesa e as questões deixadas em aberto. Responde **"o que propomos e por quê"**. O "como construir" em detalhe é responsabilidade do FDD e **não** deve ser duplicado aqui.

Não assuma nada que não esteja nas fontes. Leia os arquivos antes de escrever.

# Fontes do conhecimento (em ordem de prioridade)

1. `docs/levantamento-tecnico/ATA-REUNIAO-WEBHOOKS.md`: contexto (CTX), decisões (DT), alternativas descartadas (ALT), fora de escopo (FE), evoluções (EV), limitações (LIM), questões em aberto (QA), lacunas (LAC), riscos (RSK), estimativa e itens de ação.
2. `docs/adrs/ADR-001` a `ADR-006`: decisões fechadas, com justificativa e consequências.
3. `docs/levantamento-tecnico/LEVANTAMENTO-TECNICO.md`: arquitetura e padrões do código existente.
4. `TRANSCRICAO.md`: apenas para validar algo que não esteja claro na ata.
5. Código-fonte em `src/`: para links a arquivos reais.

# Saída

- Arquivo: `docs/RFC.md` (sobrescrever o conteúdo existente).
- Tamanho: **2 a 4 páginas** (cerca de 180 a 320 linhas de markdown, contando tabelas).
- Idioma: português, com termos técnicos consagrados em inglês.

# Estrutura obrigatória

1. **Metadados** (tabela):
   - RFC nº 001 e título.
   - Autora: **Larissa (Tech Lead)**.
   - Status: **Em revisão**.
   - Data de elaboração.
   - **Revisores:** os demais participantes da reunião, com papel — Marcos (Product Manager), Bruno (time de Pedidos), Diego (time de Plataforma), Sofia (Segurança).
   - Links para as ADRs, a ata, o levantamento técnico, o PRD e o FDD.
2. **Resumo executivo (TL;DR):** 5 a 6 bullets cobrindo problema, proposta (outbox + worker), resiliência (retry/DLQ), segurança (HMAC, secret por endpoint), garantia (at-least-once) e escopo/prazo.
3. **Contexto e problema:** necessidade de negócio (clientes B2B, polling em `GET /orders`, meta de menos de 10 s, risco comercial, prazo); limitações técnicas atuais (sem mecanismo de eventos; `changeStatus` transacional e pesado, com link para `src/modules/orders/order.service.ts`); lista curta de **objetivos** e **não objetivos**.
4. **Proposta técnica** (visão geral, sem detalhe de implementação):
   - diagrama Mermaid do fluxo: `changeStatus` → transação com outbox → worker em polling → endpoint do cliente → retry → DLQ → replay admin;
   - um parágrafo por componente, cada um com link para a ADR correspondente: outbox transacional, worker separado, retry e DLQ, entrega autenticada, garantia de entrega, API de gestão e aderência aos padrões;
   - a API de gestão é descrita **por capacidade** (cadastrar, editar, remover, listar, histórico de entregas, rotação de secret, replay), sem caminhos, payloads ou status codes;
   - tabela **"Garantias e limites"**: latência, consistência, entrega, ordenação, resiliência, timeout, tamanho do evento, transporte;
   - uma frase remetendo payload, contratos, modelagem e matriz de erros ao FDD.
5. **Alternativas consideradas:** tabela com alternativa, **trade-off que levou ao descarte** e ADR relacionada. Use **apenas alternativas reais discutidas e descartadas na reunião** (ALT-xx da ata): disparo síncrono, Redis Streams, trigger de banco, worker no processo da API, retry indefinido, 3 tentativas, DLQ como status na outbox, secret global, exactly-once, `customer_id` no JWT. Mínimo: 2.
6. **Questões em aberto**, em dois grupos:
   - **Adiadas ou em observação:** rate limiting, alerta por e-mail, escala para múltiplos workers, endurecer permissões do CRUD;
   - **Levantadas e não decididas:** `customer_id` no body ou no path, campos do payload, semântica das 5 tentativas, limite do histórico de entregas.
   - Uma nota curta sobre as lacunas não discutidas (LAC) que o FDD precisa fechar, com link para a ata. Mínimo: 2 questões.
7. **Impacto e riscos:**
   - tabela de impacto por área (fluxo de pedidos, arquitetura, banco, operação, clientes, prazo);
   - tabela de riscos com mitigação, a partir dos RSK da ata. **Não invente probabilidade** se a ata não a estimar.
8. **Fora de escopo:** lista curta (webhooks de entrada, e-mail, dashboard, arquivamento, exactly-once/ordenação global).
9. **Decisões relacionadas:** tabela com links para todas as ADRs.
10. **Próximos passos:** revisão da RFC, fechamento das questões em aberto, FDD e revisão de segurança.

# Regras de conteúdo

1. **Não invente.** Toda afirmação deve ter base na ata, nas ADRs ou no código.
2. **RFC não é rastreio da conversa.** Não cite horários nem atribua falas (nada de `[hh:mm] Nome` nem "fulano disse"). Os nomes dos participantes aparecem apenas nos metadados e, quando necessário, em "Próximos passos". O rastreio fala a fala fica na ata e no Tracker.
3. **Nível de arquitetura.** Não inclua payload campo a campo, schema de tabelas, contratos de endpoint, códigos de erro completos nem pseudocódigo. Headers só podem ser citados pelo nome, ao explicar uma garantia.
4. **Sem duplicar as ADRs.** Resuma cada decisão em um parágrafo e aponte para a ADR; o "por que exatamente assim" fica nela.
5. **Links válidos.** Links para ADRs no formato `adrs/ADR-NNN-*.md`; links de código apenas para arquivos existentes (ex.: `../src/modules/orders/order.service.ts`, `../src/app.ts`, `../src/routes/index.ts`).
6. **Fora de escopo** nunca aparece como proposta; só em "Fora de escopo", "Questões em aberto" ou "Riscos".

# Checklist de aceite

- [ ] `docs/RFC.md` existe e está em Markdown.
- [ ] Contém as seções: Metadados (autor, status, data, revisores), Resumo executivo (TL;DR), Contexto e problema, Proposta técnica, Alternativas consideradas, Questões em aberto, Impacto e riscos, Decisões relacionadas.
- [ ] Os revisores são os participantes da reunião.
- [ ] "Alternativas consideradas" lista pelo menos 2 alternativas descartadas na reunião, cada uma com o trade-off que motivou o descarte.
- [ ] "Questões em aberto" lista pelo menos 2 pontos adiados ou não decididos na reunião.
- [ ] Referencia, com link, pelo menos 2 ADRs do pacote.
- [ ] Tem 2 a 4 páginas e não duplica o detalhamento do FDD.
- [ ] Não contém `[hh:mm]` nem atribuição de falas.

# Verificação obrigatória antes de concluir

Execute e reporte o resultado:

1. Buscar os cabeçalhos das 8 seções obrigatórias em `docs/RFC.md`.
2. Contar as linhas da tabela de alternativas (≥ 2) e confirmar que cada uma tem trade-off e motivo de descarte.
3. Contar as questões em aberto (≥ 2).
4. Buscar `\]\(adrs/ADR-00[0-9]` e confirmar que há ≥ 2 links e que todos apontam para arquivos existentes em `docs/adrs/`.
5. Buscar `\[[0-9]{2}:[0-9]{2}\]` e confirmar zero ocorrências.
6. Confirmar que os links de código (`../src/...`) apontam para arquivos existentes.
7. Contar as linhas do arquivo e confirmar que o tamanho é compatível com 2 a 4 páginas.

Se o shell não estiver disponível, faça as mesmas verificações com ferramentas de busca de arquivos e de conteúdo.

# Formato da resposta final

1. Link para `docs/RFC.md` e resumo de uma linha por seção.
2. Resultado de cada verificação acima.
3. Questões em aberto que precisam de decisão dos revisores.
