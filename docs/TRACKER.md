# Tracker de Rastreabilidade — Sistema de Webhooks de Notificação de Pedidos

Este Tracker liga cada item registrado em [PRD](PRD.md), [RFC](RFC.md), [FDD](FDD.md) e [ADRs](adrs/README.md) à sua origem: uma fala da reunião em [TRANSCRICAO.md](../TRANSCRICAO.md) ou um arquivo real do código. Os documentos foram escritos sem citações de horário; é aqui que o rastreio fala a fala fica registrado.

**Legenda**

- **Fonte:** `TRANSCRICAO` (Localização = `[hh:mm] Nome`; quando o item combina falas, até três separadas por `;`) ou `CODIGO` (Localização = caminho do arquivo no repositório).
- **Tipos:** Contexto, Restrição, Decisão, Alternativa descartada, Trade-off, Limitação, Risco, Objetivo/Métrica, Escopo, Fora de escopo, Evolução futura, Requisito Funcional, Requisito Não Funcional, Contrato, Erro, Integração, Dependência, Questão em aberto, Lacuna, Proposta (pendente de validação), Estimativa (pendente de validação).
- **Propostas e estimativas 🔶:** definições que os documentos criaram porque a reunião não decidiu. A Localização aponta para **onde o tema foi levantado** na reunião ou para o **arquivo de código** que motivou a proposta; o resumo deixa explícito que não houve decisão.

## Metodologia e cobertura

São considerados **itens identificáveis** os requisitos (RF/RNF), as decisões, restrições, alternativas, trade-offs e limitações, os riscos, os objetivos e métricas, os itens de escopo e fora de escopo, as evoluções, as questões em aberto, os contratos, os erros, os pontos de integração, as dependências e as propostas 🔶.

Ficam fora do denominador os itens **derivados** de outros já rastreados: critérios de aceite (FDD CA-01 a CA-16 e checklist do PRD), estratégia de testes do PRD, cenários de uso narrativos do PRD, TL;DR da RFC e consequências positivas das ADRs.

| Documento | Itens identificáveis | Linhas no Tracker | Cobertura | Itens derivados (fora do denominador) | Cobertura incluindo derivados |
| --- | --- | --- | --- | --- | --- |
| ADRs (6 arquivos) | 122 | 122 | 100% | 22 | 85% |
| RFC | 73 | 73 | 100% | 8 | 90% |
| FDD | 136 | 136 | 100% | 16 | 89% |
| PRD | 100 | 100 | 100% | 30 | 77% |
| **Total** | **431** | **431** | **100%** | **76** | **85%** |

Distribuição por Fonte: ver o [resumo final](#resumo-por-fonte).

## Tabela de rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão Outbox no MySQL: evento gravado na mesma transação que atualiza orders e order_status_history | TRANSCRICAO | [09:06] Diego; [09:08] Larissa |
| ADR-001-CTX-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Transação de status já é pesada; HTTP no meio faria cliente lento travar outros pedidos | TRANSCRICAO | [09:04] Bruno |
| ADR-001-CTX-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Não é possível dar rollback no status se o cliente estiver fora do ar | TRANSCRICAO | [09:04] Bruno |
| ADR-001-CTX-03 | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Time pequeno, sem disposição para operar nova infraestrutura | TRANSCRICAO | [09:07] Diego |
| ADR-001-CTX-04 | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto | changeStatus roda em prisma.$transaction e atualiza pedido, histórico e estoque | CODIGO | src/modules/orders/order.service.ts |
| ADR-001-DEC-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Publicação via função publishWebhookEvent(tx, order, fromStatus, toStatus), sem injetar repository | TRANSCRICAO | [09:41] Bruno; [09:41] Diego |
| ADR-001-DEC-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Falha ao inserir na outbox provoca rollback da transação inteira | TRANSCRICAO | [09:40] Bruno |
| ADR-001-DEC-03 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Filtro de eventos aplicado na inserção da outbox | TRANSCRICAO | [09:34] Bruno; [09:34] Diego |
| ADR-001-DEC-04 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Payload renderizado na inserção (snapshot) | TRANSCRICAO | [09:52] Larissa; [09:52] Diego |
| ADR-001-DEC-05 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Identificador da outbox em UUID, seguindo o padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-001-DEC-06 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Índices em status e created_at; leitura dos pendentes em batch pequeno | TRANSCRICAO | [09:08] Diego |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa descartada | Disparo síncrono no changeStatus | TRANSCRICAO | [09:04] Bruno; [09:06] Diego |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa descartada | Redis Streams / fila externa (overengineering para time pequeno) | TRANSCRICAO | [09:07] Larissa; [09:07] Diego |
| ADR-001-ALT-03 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa descartada | Filtrar eventos no momento do envio | TRANSCRICAO | [09:34] Diego; [09:34] Bruno |
| ADR-001-ALT-04 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa descartada | Guardar só order_id e renderizar no envio | TRANSCRICAO | [09:51] Bruno; [09:52] Larissa |
| ADR-001-CONS-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Falha na escrita da outbox impede a mudança de status (preferível a status sem evento) | TRANSCRICAO | [09:40] Bruno; [09:41] Diego |
| ADR-001-CONS-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | OrderService passa a depender do módulo de webhooks por uma função | TRANSCRICAO | [09:41] Bruno |
| ADR-001-CONS-03 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | MySQL passa a servir também como fila lida pelo worker | TRANSCRICAO | [09:06] Diego |
| ADR-001-CONS-04 | docs/adrs/ADR-001-outbox-no-mysql.md | Risco | Crescimento da tabela; arquivamento de entregues fora do escopo | TRANSCRICAO | [09:07] Bruno; [09:08] Diego |
| ADR-001-CONS-05 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Latência final depende do intervalo de leitura do worker | TRANSCRICAO | [09:09] Diego |
| ADR-001-QA-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Questão em aberto | Campos do payload além de total_cents (QA-04) | TRANSCRICAO | [09:43] Diego |
| ADR-001-QA-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Lacuna | Onde validar o limite de 64 KB (LAC-13); limite definido, ponto de validação não | TRANSCRICAO | [09:24] Larissa |
| ADR-002 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Worker em processo separado com polling a cada 2 s | TRANSCRICAO | [09:09] Diego; [09:10] Larissa; [09:11] Diego |
| ADR-002-CTX-01 | docs/adrs/ADR-002-worker-separado-com-polling.md | Restrição | Clientes consideram tempo real abaixo de 10 s | TRANSCRICAO | [09:02] Marcos |
| ADR-002-CTX-02 | docs/adrs/ADR-002-worker-separado-com-polling.md | Restrição | MySQL não tem NOTIFY/LISTEN; trigger não notifica processo externo | TRANSCRICAO | [09:09] Diego |
| ADR-002-CTX-03 | docs/adrs/ADR-002-worker-separado-com-polling.md | Restrição | Clientes não exigem ordenação global | TRANSCRICAO | [09:14] Marcos |
| ADR-002-DEC-01 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Entry-point src/worker.ts e script npm run worker | TRANSCRICAO | [09:11] Larissa |
| ADR-002-DEC-02 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Lógica do worker dentro do módulo (webhook.worker.ts ou webhook.processor.ts) | TRANSCRICAO | [09:28] Bruno |
| ADR-002-DEC-03 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Mesma DATABASE_URL com instância própria de PrismaClient | TRANSCRICAO | [09:30] Bruno |
| ADR-002-DEC-04 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Worker único processando em ordem de created_at | TRANSCRICAO | [09:12] Diego |
| ADR-002-DEC-05 | docs/adrs/ADR-002-worker-separado-com-polling.md | Integração | Modelo de entry-point com bootstrap e graceful shutdown | CODIGO | src/server.ts |
| ADR-002-DEC-06 | docs/adrs/ADR-002-worker-separado-com-polling.md | Integração | Factory createPrismaClient reutilizável pelo worker | CODIGO | src/config/database.ts |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-separado-com-polling.md | Alternativa descartada | Trigger de banco para acionar o worker | TRANSCRICAO | [09:09] Bruno; [09:09] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-separado-com-polling.md | Alternativa descartada | Worker no mesmo processo da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT-03 | docs/adrs/ADR-002-worker-separado-com-polling.md | Alternativa descartada | Múltiplos workers em paralelo (adiado) | TRANSCRICAO | [09:13] Bruno; [09:13] Diego |
| ADR-002-CONS-01 | docs/adrs/ADR-002-worker-separado-com-polling.md | Trade-off | Latência de até 2 s no pior caso, aceita | TRANSCRICAO | [09:10] Larissa; [09:10] Marcos |
| ADR-002-CONS-02 | docs/adrs/ADR-002-worker-separado-com-polling.md | Trade-off | Consultas contínuas ao banco a cada 2 s | TRANSCRICAO | [09:09] Diego |
| ADR-002-CONS-03 | docs/adrs/ADR-002-worker-separado-com-polling.md | Limitação | Ordenação apenas por order_id e com worker único | TRANSCRICAO | [09:13] Larissa |
| ADR-002-CONS-04 | docs/adrs/ADR-002-worker-separado-com-polling.md | Trade-off | Vazão limitada a um worker até decidir estratégia de escala | TRANSCRICAO | [09:13] Diego |
| ADR-002-CONS-05 | docs/adrs/ADR-002-worker-separado-com-polling.md | Risco | Dois processos para implantar e operar | TRANSCRICAO | [09:11] Diego |
| ADR-002-CONS-06 | docs/adrs/ADR-002-worker-separado-com-polling.md | Risco | Worker abre pool próprio de conexões no mesmo MySQL | TRANSCRICAO | [09:30] Bruno |
| ADR-002-CONS-07 | docs/adrs/ADR-002-worker-separado-com-polling.md | Risco | Worker parado acumula eventos na outbox, sem perda | TRANSCRICAO | [09:06] Diego |
| ADR-002-QA-01 | docs/adrs/ADR-002-worker-separado-com-polling.md | Questão em aberto | Nome do arquivo do processamento (QA-03) | TRANSCRICAO | [09:28] Bruno |
| ADR-002-QA-02 | docs/adrs/ADR-002-worker-separado-com-polling.md | Lacuna | Recuperação de eventos presos em processando (LAC-08) | TRANSCRICAO | [09:08] Diego |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Decisão | Backoff exponencial com 5 tentativas: 1m, 5m, 30m, 2h, 12h | TRANSCRICAO | [09:17] Diego; [09:17] Larissa |
| ADR-003-CTX-01 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Restrição | Já houve cliente com 2 h de manutenção planejada | TRANSCRICAO | [09:16] Diego |
| ADR-003-CTX-02 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Restrição | Evento não pode ficar pendurado para sempre | TRANSCRICAO | [09:15] Diego |
| ADR-003-CTX-03 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Restrição | Mexer na fila de entrega não é atividade de operador | TRANSCRICAO | [09:36] Sofia |
| ADR-003-DEC-01 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Decisão | DLQ em tabela separada com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| ADR-003-DEC-02 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Decisão | Replay manual via POST /admin/webhooks/dead-letter/:id/replay, recolocando como pendente | TRANSCRICAO | [09:18] Diego; [09:35] Diego |
| ADR-003-DEC-03 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Decisão | Replay exige role ADMIN, reaproveitando requireRole | TRANSCRICAO | [09:36] Sofia; [09:36] Larissa |
| ADR-003-DEC-04 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Decisão | Replay registra quem executou, para auditoria | TRANSCRICAO | [09:36] Sofia |
| ADR-003-DEC-05 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Decisão | Timeout de 10 s tratado como falha e marcado para retry | TRANSCRICAO | [09:42] Diego |
| ADR-003-DEC-06 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Integração | requireRole existente usado no endpoint de replay | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-003-DEC-07 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Decisão | Janela de ~15 h aceita por produto | TRANSCRICAO | [09:17] Marcos |
| ADR-003-ALT-01 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Alternativa descartada | Retry indefinido com backoff | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-02 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Alternativa descartada | Apenas 3 tentativas | TRANSCRICAO | [09:16] Bruno; [09:16] Diego |
| ADR-003-ALT-03 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Alternativa descartada | DLQ como status failed na própria outbox | TRANSCRICAO | [09:17] Larissa; [09:18] Diego |
| ADR-003-CONS-01 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Trade-off | Entrega pode atrasar até ~15 h após a primeira falha | TRANSCRICAO | [09:17] Diego |
| ADR-003-CONS-02 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Trade-off | Eventos na DLQ só voltam por ação manual | TRANSCRICAO | [09:18] Diego |
| ADR-003-CONS-03 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Trade-off | Replay reenvia o evento e reforça a deduplicação no cliente | TRANSCRICAO | [09:24] Diego |
| ADR-003-CONS-04 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Trade-off | Timeout de 10 s pode classificar cliente lento como falha | TRANSCRICAO | [09:42] Diego |
| ADR-003-CONS-05 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Risco | Necessário acompanhar o volume da DLQ para acionar replay | TRANSCRICAO | [09:18] Diego |
| ADR-003-CONS-06 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Fora de escopo | Alerta por e-mail após falhas repetidas fica para próxima fase | TRANSCRICAO | [09:37] Larissa |
| ADR-003-QA-01 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Questão em aberto | 5 tentativas incluem o envio inicial? (QA-05) | TRANSCRICAO | [09:17] Larissa |
| ADR-003-QA-02 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Lacuna | Quais respostas HTTP contam como sucesso (LAC-06); só o timeout foi definido | TRANSCRICAO | [09:42] Diego |
| ADR-003-QA-03 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Lacuna | Replay reinicia o contador de tentativas? (LAC-10) | TRANSCRICAO | [09:18] Diego |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, assinatura no header X-Signature | TRANSCRICAO | [09:20] Sofia; [09:22] Sofia |
| ADR-004-CTX-01 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Restrição | Cliente precisa validar origem e integridade do payload | TRANSCRICAO | [09:19] Sofia |
| ADR-004-CTX-02 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Contexto | Já houve cliente que vazou secret em log | TRANSCRICAO | [09:22] Diego |
| ADR-004-CTX-03 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Restrição | Troca de secret não pode quebrar a integração de imediato | TRANSCRICAO | [09:21] Sofia |
| ADR-004-DEC-01 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | Secret única por endpoint, nunca global | TRANSCRICAO | [09:21] Sofia |
| ADR-004-DEC-02 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | Secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| ADR-004-DEC-03 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | Rotação pela API com 24 h de convivência | TRANSCRICAO | [09:21] Sofia |
| ADR-004-DEC-04 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | URL obrigatoriamente https, validada no schema Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-004-DEC-05 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | Header X-Timestamp para o cliente detectar replay attack | TRANSCRICAO | [09:44] Diego |
| ADR-004-ALT-01 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Alternativa descartada | Secret global da plataforma | TRANSCRICAO | [09:21] Sofia |
| ADR-004-ALT-02 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Alternativa descartada | Secret fixa, sem rotação | TRANSCRICAO | [09:21] Sofia; [09:22] Diego |
| ADR-004-CONS-01 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Trade-off | Plataforma passa a guardar secret por endpoint (url + secret + customer_id + ativo) | TRANSCRICAO | [09:21] Bruno |
| ADR-004-CONS-02 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Trade-off | Verificação da assinatura fica com o cliente | TRANSCRICAO | [09:20] Sofia |
| ADR-004-CONS-03 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Trade-off | Clientes com http não conseguem se integrar | TRANSCRICAO | [09:23] Sofia |
| ADR-004-CONS-04 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Risco | Revisão de segurança de pelo menos 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| ADR-004-CONS-05 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Risco | Secret não pode ir para log; logger já aplica redaction | CODIGO | src/shared/logger/index.ts |
| ADR-004-QA-01 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Lacuna | Assinatura durante as 24 h de convivência (LAC-03) | TRANSCRICAO | [09:21] Sofia |
| ADR-004-QA-02 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Lacuna | Formato da assinatura e uso do X-Timestamp no HMAC (LAC-04) | TRANSCRICAO | [09:22] Sofia |
| ADR-004-QA-03 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Lacuna | Armazenamento da secret e reexibição (LAC-05) | TRANSCRICAO | [09:31] Marcos |
| ADR-004-QA-04 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Lacuna | Contrato do endpoint de rotação (LAC-01) | TRANSCRICAO | [09:21] Sofia |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | Garantia at-least-once com X-Event-Id para deduplicação | TRANSCRICAO | [09:24] Diego; [09:25] Diego; [09:26] Larissa |
| ADR-005-CTX-01 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Restrição | Entrega única exigiria coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| ADR-005-CTX-02 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Restrição | Regra precisa ser comunicada claramente aos clientes | TRANSCRICAO | [09:26] Marcos |
| ADR-005-DEC-01 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | UUID gerado quando o evento entra na outbox, único por evento | TRANSCRICAO | [09:25] Diego |
| ADR-005-DEC-02 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | Mesmo identificador no campo event_id do payload | TRANSCRICAO | [09:43] Diego |
| ADR-005-DEC-03 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | Documentação em destaque no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| ADR-005-ALT-01 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| ADR-005-CONS-01 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Trade-off | Responsabilidade de deduplicar fica com o cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-005-CONS-02 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Trade-off | Duplicatas são esperadas (retries e replays) | TRANSCRICAO | [09:24] Diego |
| ADR-005-CONS-03 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Risco | Eficácia depende de documentação clara para os clientes | TRANSCRICAO | [09:26] Marcos |
| ADR-005-CONS-04 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Limitação | Sem ordem global; ordem por pedido depende do worker único | TRANSCRICAO | [09:12] Diego |
| ADR-005-QA-01 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Lacuna | Replay mantém ou gera novo event_id? Regra "único por evento" sugere manter | TRANSCRICAO | [09:25] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Reuso máximo dos padrões existentes; webhook como módulo igual aos outros | TRANSCRICAO | [09:30] Larissa |
| ADR-006-CTX-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Contexto | Cada domínio é um módulo com controller, service, repository, routes e schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-CTX-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Contexto | Composição de dependências e registro de rotas por módulo | CODIGO | src/routes/index.ts |
| ADR-006-CTX-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Restrição | Prazo de 3 sprints | TRANSCRICAO | [09:46] Larissa |
| ADR-006-DEC-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Novo módulo src/modules/webhooks | TRANSCRICAO | [09:27] Bruno |
| ADR-006-DEC-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Erros via AppError com prefixo WEBHOOK_ | TRANSCRICAO | [09:28] Bruno; [09:29] Larissa |
| ADR-006-DEC-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Integração | Modelo de InsufficientStockError e InvalidStatusTransitionError | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-DEC-04 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Error middleware existente trata os novos erros sem alteração | TRANSCRICAO | [09:29] Bruno |
| ADR-006-DEC-05 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Integração | Error middleware já traduz AppError, Zod e Prisma | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-DEC-06 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Logger Pino existente, nenhum logger novo | TRANSCRICAO | [09:29] Bruno |
| ADR-006-DEC-07 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | customer_id informado na requisição, não derivado do JWT | TRANSCRICAO | [09:32] Larissa |
| ADR-006-DEC-08 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | CRUD de configuração aceita qualquer role autenticada por enquanto | TRANSCRICAO | [09:37] Sofia |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Alternativa descartada | Estrutura e ferramentas próprias no módulo | TRANSCRICAO | [09:29] Bruno |
| ADR-006-ALT-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Alternativa descartada | Worker compartilhando o PrismaClient da API | TRANSCRICAO | [09:29] Diego; [09:30] Bruno |
| ADR-006-ALT-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Alternativa descartada | customer_id implícito no JWT | TRANSCRICAO | [09:31] Marcos; [09:32] Bruno |
| ADR-006-CONS-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Trade-off | Herda limitações atuais: projeto sem biblioteca de métricas ou tracing | CODIGO | package.json |
| ADR-006-CONS-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Trade-off | Autorização ampla no CRUD nesta fase | TRANSCRICAO | [09:37] Sofia |
| ADR-006-CONS-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Risco | Novos pontos de registro no composition root | CODIGO | src/app.ts |
| ADR-006-CONS-04 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Risco | Catálogo completo de códigos WEBHOOK_ ainda aberto | TRANSCRICAO | [09:28] Bruno |
| ADR-006-QA-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Questão em aberto | customer_id no body ou no path (QA-01) | TRANSCRICAO | [09:32] Larissa |
| ADR-006-QA-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Questão em aberto | Catálogo completo de códigos WEBHOOK_ (QA-07) | TRANSCRICAO | [09:28] Bruno |
| ADR-006-QA-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Questão em aberto | Quando endurecer permissões do CRUD (QA-08) | TRANSCRICAO | [09:37] Sofia |
| ADR-006-QA-04 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Lacuna | Vínculo entre usuários que representam o cliente e customer inexistente no schema (LAC-14) | CODIGO | prisma/schema.prisma |
| RFC-CTX-01 | docs/RFC.md | Contexto | Três clientes B2B fizeram pedido formal de notificação | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-02 | docs/RFC.md | Contexto | Clientes fazem polling em GET /orders; integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-03 | docs/RFC.md | Restrição | Tempo real = abaixo de 10 s | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-04 | docs/RFC.md | Contexto | Atlas pode migrar para concorrente | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-05 | docs/RFC.md | Restrição | Atlas pediu entrega até fim de novembro | TRANSCRICAO | [09:45] Marcos |
| RFC-CTX-06 | docs/RFC.md | Contexto | OMS sem mecanismo de eventos, filas ou jobs | CODIGO | src/app.ts |
| RFC-CTX-07 | docs/RFC.md | Contexto | changeStatus valida a máquina de estados, movimenta estoque e grava histórico em transação | CODIGO | src/modules/orders/order.service.ts |
| RFC-CTX-08 | docs/RFC.md | Restrição | Time pequeno, sem nova infraestrutura | TRANSCRICAO | [09:07] Diego |
| RFC-OBJ-01 | docs/RFC.md | Objetivo/Métrica | Notificar mudanças assinadas em menos de 10 s | TRANSCRICAO | [09:02] Marcos |
| RFC-OBJ-02 | docs/RFC.md | Objetivo/Métrica | Nenhuma mudança de status sem evento | TRANSCRICAO | [09:40] Bruno |
| RFC-OBJ-03 | docs/RFC.md | Objetivo/Métrica | Cliente verifica autenticidade e integridade | TRANSCRICAO | [09:19] Sofia |
| RFC-OBJ-04 | docs/RFC.md | Objetivo/Métrica | Cliente gerencia webhooks e acompanha entregas via API | TRANSCRICAO | [09:31] Marcos; [09:34] Marcos |
| RFC-NOBJ-01 | docs/RFC.md | Fora de escopo | Receber eventos dos clientes (inbound) | TRANSCRICAO | [09:02] Marcos |
| RFC-NOBJ-02 | docs/RFC.md | Fora de escopo | Exactly-once e ordenação global | TRANSCRICAO | [09:25] Diego; [09:13] Larissa |
| RFC-NOBJ-03 | docs/RFC.md | Fora de escopo | Interface visual de gestão | TRANSCRICAO | [09:40] Larissa |
| RFC-COMP-01 | docs/RFC.md | Decisão | Componente outbox transacional | TRANSCRICAO | [09:06] Diego |
| RFC-COMP-02 | docs/RFC.md | Decisão | Componente worker em processo separado | TRANSCRICAO | [09:11] Diego |
| RFC-COMP-03 | docs/RFC.md | Decisão | Componente retry e DLQ com replay ADMIN | TRANSCRICAO | [09:17] Larissa; [09:36] Larissa |
| RFC-COMP-04 | docs/RFC.md | Decisão | Componente entrega autenticada (HMAC, rotação, HTTPS, X-Timestamp) | TRANSCRICAO | [09:22] Sofia; [09:23] Sofia; [09:44] Diego |
| RFC-COMP-05 | docs/RFC.md | Decisão | Componente garantia at-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| RFC-COMP-06 | docs/RFC.md | Decisão | Módulo webhooks aderente aos padrões do projeto | TRANSCRICAO | [09:30] Larissa |
| RFC-COMP-07 | docs/RFC.md | Requisito Funcional | Capacidades da API: CRUD, histórico de entregas, rotação, replay | TRANSCRICAO | [09:33] Bruno; [09:34] Marcos; [09:35] Diego |
| RFC-GAR-01 | docs/RFC.md | Requisito Não Funcional | Latência: polling de 2 s, abaixo do teto de 10 s | TRANSCRICAO | [09:10] Larissa |
| RFC-GAR-02 | docs/RFC.md | Requisito Não Funcional | Consistência: toda mudança com assinatura gera evento | TRANSCRICAO | [09:41] Diego |
| RFC-GAR-03 | docs/RFC.md | Requisito Não Funcional | Entrega at-least-once | TRANSCRICAO | [09:24] Diego |
| RFC-GAR-04 | docs/RFC.md | Limitação | Ordenação por pedido com worker único | TRANSCRICAO | [09:13] Larissa |
| RFC-GAR-05 | docs/RFC.md | Requisito Não Funcional | Até 5 tentativas em ~15 h, depois DLQ | TRANSCRICAO | [09:17] Diego |
| RFC-GAR-06 | docs/RFC.md | Requisito Não Funcional | Timeout de 10 s por entrega | TRANSCRICAO | [09:42] Diego |
| RFC-GAR-07 | docs/RFC.md | Requisito Não Funcional | Evento máximo de 64 KB, erro sem truncar | TRANSCRICAO | [09:24] Diego; [09:24] Larissa |
| RFC-GAR-08 | docs/RFC.md | Requisito Não Funcional | Transporte HTTPS obrigatório | TRANSCRICAO | [09:23] Sofia |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Disparo síncrono no changeStatus | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger de banco | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Worker no processo da API | TRANSCRICAO | [09:11] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Retry indefinido | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | Apenas 3 tentativas | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-07 | docs/RFC.md | Alternativa descartada | DLQ como status na outbox | TRANSCRICAO | [09:18] Diego |
| RFC-ALT-08 | docs/RFC.md | Alternativa descartada | Secret global | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-09 | docs/RFC.md | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-10 | docs/RFC.md | Alternativa descartada | customer_id extraído do JWT | TRANSCRICAO | [09:32] Bruno |
| RFC-QA-01 | docs/RFC.md | Questão em aberto | Rate limiting de envio (observar) | TRANSCRICAO | [09:38] Diego; [09:39] Larissa |
| RFC-QA-02 | docs/RFC.md | Questão em aberto | Alerta por e-mail (próxima fase) | TRANSCRICAO | [09:37] Larissa |
| RFC-QA-03 | docs/RFC.md | Questão em aberto | Escala para múltiplos workers (particionar ou lock) | TRANSCRICAO | [09:13] Diego |
| RFC-QA-04 | docs/RFC.md | Questão em aberto | Endurecer permissões do CRUD | TRANSCRICAO | [09:37] Sofia |
| RFC-QA-05 | docs/RFC.md | Questão em aberto | customer_id no corpo ou no caminho | TRANSCRICAO | [09:32] Larissa |
| RFC-QA-06 | docs/RFC.md | Questão em aberto | Campos básicos do pedido além do total | TRANSCRICAO | [09:43] Diego |
| RFC-QA-07 | docs/RFC.md | Questão em aberto | 5 tentativas incluem o envio inicial? | TRANSCRICAO | [09:17] Larissa |
| RFC-QA-08 | docs/RFC.md | Questão em aberto | Histórico limitado a 100 ou paginado | TRANSCRICAO | [09:34] Marcos |
| RFC-QA-09 | docs/RFC.md | Lacuna | Lacunas não discutidas (assinatura na rotação, sucesso HTTP, eventos presos) remetidas ao FDD | TRANSCRICAO | [09:21] Sofia; [09:42] Diego; [09:08] Diego |
| RFC-IMP-01 | docs/RFC.md | Trade-off | changeStatus ganha escrita adicional na transação | CODIGO | src/modules/orders/order.service.ts |
| RFC-IMP-02 | docs/RFC.md | Integração | Novo módulo registrado no agregador de rotas e composition root | CODIGO | src/routes/index.ts |
| RFC-IMP-03 | docs/RFC.md | Trade-off | Consultas periódicas e crescimento da outbox | TRANSCRICAO | [09:08] Diego |
| RFC-IMP-04 | docs/RFC.md | Dependência | Clientes precisam de HTTPS, verificar HMAC e deduplicar | TRANSCRICAO | [09:23] Sofia; [09:25] Sofia |
| RFC-IMP-05 | docs/RFC.md | Restrição | Estimativa de 3 sprints com revisão de segurança | TRANSCRICAO | [09:46] Larissa; [09:47] Larissa |
| RFC-RISK-01 | docs/RFC.md | Risco | Perda do cliente Atlas por atraso | TRANSCRICAO | [09:00] Marcos |
| RFC-RISK-02 | docs/RFC.md | Risco | Cliente lento travando mudanças de status | TRANSCRICAO | [09:04] Bruno |
| RFC-RISK-03 | docs/RFC.md | Risco | Status alterado sem evento | TRANSCRICAO | [09:41] Diego |
| RFC-RISK-04 | docs/RFC.md | Risco | Acúmulo de eventos degradando o worker | TRANSCRICAO | [09:07] Bruno |
| RFC-RISK-05 | docs/RFC.md | Risco | Vazamento de secret | TRANSCRICAO | [09:22] Diego |
| RFC-RISK-06 | docs/RFC.md | Risco | Requisição falsificada ou adulterada | TRANSCRICAO | [09:19] Sofia |
| RFC-RISK-07 | docs/RFC.md | Risco | Reenvio malicioso de requisições antigas | TRANSCRICAO | [09:44] Diego |
| RFC-RISK-08 | docs/RFC.md | Risco | Cliente processando evento duplicado | TRANSCRICAO | [09:25] Diego |
| RFC-RISK-09 | docs/RFC.md | Risco | Indisponibilidade prolongada do cliente | TRANSCRICAO | [09:16] Diego |
| RFC-RISK-10 | docs/RFC.md | Risco | Volume alto de chamadas a um mesmo cliente | TRANSCRICAO | [09:38] Diego |
| RFC-RISK-11 | docs/RFC.md | Risco | Evento anormalmente grande | TRANSCRICAO | [09:23] Sofia |
| RFC-RISK-12 | docs/RFC.md | Risco | Falha de segurança em HMAC ou geração de secret | TRANSCRICAO | [09:46] Sofia |
| RFC-OOS-01 | docs/RFC.md | Fora de escopo | Webhooks de entrada | TRANSCRICAO | [09:02] Marcos |
| RFC-OOS-02 | docs/RFC.md | Fora de escopo | Alerta por e-mail | TRANSCRICAO | [09:37] Larissa |
| RFC-OOS-03 | docs/RFC.md | Fora de escopo | Dashboard / painel visual | TRANSCRICAO | [09:40] Larissa |
| RFC-OOS-04 | docs/RFC.md | Fora de escopo | Arquivamento de eventos entregues | TRANSCRICAO | [09:08] Diego |
| RFC-OOS-05 | docs/RFC.md | Fora de escopo | Exactly-once e ordenação global | TRANSCRICAO | [09:25] Diego |
| RFC-NEXT-01 | docs/RFC.md | Dependência | Sessão de revisão do design com Bruno e Diego antes de codar | TRANSCRICAO | [09:50] Larissa |
| RFC-NEXT-02 | docs/RFC.md | Dependência | Agendar revisão de segurança antes do deploy | TRANSCRICAO | [09:49] Sofia |
| FDD-OBJ-01 | docs/FDD.md | Objetivo/Métrica | Polling de 2 s; entrega iniciada bem abaixo de 10 s | TRANSCRICAO | [09:09] Diego |
| FDD-OBJ-02 | docs/FDD.md | Objetivo/Métrica | 100% das mudanças com assinatura geram evento | TRANSCRICAO | [09:40] Bruno |
| FDD-OBJ-03 | docs/FDD.md | Objetivo/Métrica | At-least-once com X-Event-Id estável | TRANSCRICAO | [09:24] Diego |
| FDD-OBJ-04 | docs/FDD.md | Objetivo/Métrica | changeStatus independente da disponibilidade dos clientes | TRANSCRICAO | [09:04] Bruno |
| FDD-OBJ-05 | docs/FDD.md | Objetivo/Métrica | Até 5 reenvios em ~15 h, depois DLQ | TRANSCRICAO | [09:17] Diego |
| FDD-OBJ-06 | docs/FDD.md | Objetivo/Métrica | Timeout de 10 s por chamada | TRANSCRICAO | [09:42] Diego |
| FDD-OBJ-07 | docs/FDD.md | Objetivo/Métrica | Eventos acima de 64 KB não enviados | TRANSCRICAO | [09:24] Diego |
| FDD-OBJ-08 | docs/FDD.md | Objetivo/Métrica | 100% das entregas assinadas; só URLs https | TRANSCRICAO | [09:20] Sofia; [09:23] Sofia |
| FDD-OBJ-09 | docs/FDD.md | Objetivo/Métrica | Nenhuma biblioteca nova; padrões existentes | TRANSCRICAO | [09:29] Bruno; [09:30] Larissa |
| FDD-ESC-01 | docs/FDD.md | Escopo | Módulo src/modules/webhooks com CRUD, histórico, rotação e replay | TRANSCRICAO | [09:27] Bruno |
| FDD-ESC-02 | docs/FDD.md | Escopo | Evento order.status_changed publicado a partir de changeStatus | TRANSCRICAO | [09:43] Diego; [09:40] Bruno |
| FDD-ESC-03 | docs/FDD.md | Escopo | Worker em src/worker.ts com entrega, retry e DLQ | TRANSCRICAO | [09:11] Larissa |
| FDD-EXC-01 | docs/FDD.md | Fora de escopo | Webhooks de entrada | TRANSCRICAO | [09:02] Marcos |
| FDD-EXC-02 | docs/FDD.md | Fora de escopo | Alerta por e-mail | TRANSCRICAO | [09:37] Larissa |
| FDD-EXC-03 | docs/FDD.md | Fora de escopo | Rate limiting de envio | TRANSCRICAO | [09:39] Larissa |
| FDD-EXC-04 | docs/FDD.md | Fora de escopo | Dashboard visual | TRANSCRICAO | [09:40] Larissa |
| FDD-EXC-05 | docs/FDD.md | Fora de escopo | Arquivamento de entregues | TRANSCRICAO | [09:08] Diego |
| FDD-EXC-06 | docs/FDD.md | Fora de escopo | Múltiplos workers e ordenação global | TRANSCRICAO | [09:13] Diego |
| FDD-EXC-07 | docs/FDD.md | Fora de escopo | Exactly-once | TRANSCRICAO | [09:25] Diego |
| FDD-EXC-08 | docs/FDD.md | Fora de escopo | Reprocessamento automático da DLQ (replay é manual) | TRANSCRICAO | [09:18] Diego |
| FDD-EXC-09 | docs/FDD.md | Fora de escopo | Tipos de evento além de order.status_changed | TRANSCRICAO | [09:43] Diego |
| FDD-DADOS-01 | docs/FDD.md | Decisão | webhook_endpoints com url, secret, customer_id e estado ativo; lista de status assinados | TRANSCRICAO | [09:21] Bruno; [09:31] Marcos |
| FDD-DADOS-02 | docs/FDD.md | Decisão | webhook_outbox com status pendente/processando/entregue/falhou e índices | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-03 | docs/FDD.md | Proposta (pendente de validação) | 🔶 Tabela webhook_deliveries para o histórico (P-15); dados exigidos pelo PM, persistência não decidida | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-04 | docs/FDD.md | Decisão | webhook_dead_letter com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-05 | docs/FDD.md | Restrição | Convenções UUID CHAR(36), @@map snake_case e índices | CODIGO | prisma/schema.prisma |
| FDD-FLUXO-01 | docs/FDD.md | Decisão | Criação do evento dentro da transação de changeStatus | TRANSCRICAO | [09:40] Bruno; [09:41] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Decisão | Filtro dos endpoints assinantes pelo novo status | TRANSCRICAO | [09:33] Marcos; [09:34] Bruno |
| FDD-FLUXO-03 | docs/FDD.md | Decisão | Worker busca pendentes mais antigos, processa e marca | TRANSCRICAO | [09:09] Diego; [09:08] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Decisão | Processamento sequencial por created_at com worker único | TRANSCRICAO | [09:12] Diego |
| FDD-FLUXO-05 | docs/FDD.md | Decisão | Tabela de retry 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-06 | docs/FDD.md | Decisão | Tentativas esgotadas movem o evento para a DLQ | TRANSCRICAO | [09:15] Diego; [09:18] Diego |
| FDD-FLUXO-07 | docs/FDD.md | Decisão | Replay ADMIN recoloca como pendente e registra quem executou | TRANSCRICAO | [09:18] Diego; [09:36] Sofia |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /webhooks com url e lista de status; secret gerada e devolvida | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET listagem de webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Proposta (pendente de validação) | 🔶 GET /webhooks/:id (P-02); consulta individual não citada na reunião | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | PATCH para editar webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | DELETE para remover webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | Rotação de secret pela API com 24 h de convivência | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries com sucesso/falha, payload, response e tempo | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay com role ADMIN | TRANSCRICAO | [09:35] Diego; [09:36] Sofia |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Payload JSON: event_id, event_type, timestamp, order_id, order_number, from/to_status, customer_id, total_cents; sem items | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | Headers X-Event-Id, X-Signature, X-Timestamp, Content-Type e X-Webhook-Id | TRANSCRICAO | [09:44] Diego; [09:44] Sofia |
| FDD-CONTRATO-11 | docs/FDD.md | Restrição | Listagens no envelope data + pagination existente | CODIGO | src/shared/http/response.ts |
| FDD-CONTRATO-12 | docs/FDD.md | Restrição | Erros no envelope error { code, message, details } existente | CODIGO | src/middlewares/error.middleware.ts |
| FDD-ERRO-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-03 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-04 | docs/FDD.md | Proposta (pendente de validação) | 🔶 WEBHOOK_CUSTOMER_NOT_FOUND (P-07); segue a regra do prefixo, código não citado | TRANSCRICAO | [09:29] Larissa |
| FDD-ERRO-05 | docs/FDD.md | Proposta (pendente de validação) | 🔶 WEBHOOK_DEAD_LETTER_NOT_FOUND (P-07) | TRANSCRICAO | [09:29] Larissa |
| FDD-ERRO-06 | docs/FDD.md | Proposta (pendente de validação) | 🔶 WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED (P-07) | TRANSCRICAO | [09:29] Larissa |
| FDD-ERRO-07 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT: cliente não respondeu em 10 s | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-08 | docs/FDD.md | Proposta (pendente de validação) | 🔶 WEBHOOK_DELIVERY_HTTP_ERROR: resposta fora de 2xx (P-12) | TRANSCRICAO | [09:14] Larissa |
| FDD-ERRO-09 | docs/FDD.md | Proposta (pendente de validação) | 🔶 WEBHOOK_DELIVERY_NETWORK_ERROR: cliente inacessível (P-07) | TRANSCRICAO | [09:14] Larissa |
| FDD-ERRO-10 | docs/FDD.md | Erro | WEBHOOK_MAX_ATTEMPTS_EXCEEDED: teto de tentativas atingido | TRANSCRICAO | [09:15] Diego |
| FDD-ERRO-11 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE: evento acima de 64 KB | TRANSCRICAO | [09:24] Larissa |
| FDD-ERRO-12 | docs/FDD.md | Proposta (pendente de validação) | 🔶 WEBHOOK_INACTIVE: pendentes de endpoint desativado (P-13) | TRANSCRICAO | [09:21] Bruno |
| FDD-ERRO-13 | docs/FDD.md | Integração | Reuso de VALIDATION_ERROR, UNAUTHORIZED e FORBIDDEN | CODIGO | src/shared/errors/http-errors.ts |
| FDD-RES-01 | docs/FDD.md | Requisito Não Funcional | Isolamento: entrega fora da transação, em outro processo | TRANSCRICAO | [09:04] Bruno; [09:11] Diego |
| FDD-RES-02 | docs/FDD.md | Requisito Não Funcional | Timeout de 10 s na chamada HTTP | TRANSCRICAO | [09:42] Diego |
| FDD-RES-03 | docs/FDD.md | Decisão | Retry com backoff e agendamento persistido | TRANSCRICAO | [09:15] Diego |
| FDD-RES-04 | docs/FDD.md | Proposta (pendente de validação) | 🔶 Classificação de falhas transitórias x permanentes (P-12) | TRANSCRICAO | [09:42] Diego |
| FDD-RES-05 | docs/FDD.md | Decisão | Fallback: DLQ e replay manual | TRANSCRICAO | [09:18] Diego |
| FDD-RES-06 | docs/FDD.md | Proposta (pendente de validação) | 🔶 Reclaim de eventos presos em processando (P-14) | TRANSCRICAO | [09:08] Diego |
| FDD-RES-07 | docs/FDD.md | Decisão | Idempotência no cliente via X-Event-Id | TRANSCRICAO | [09:25] Diego |
| FDD-RES-08 | docs/FDD.md | Limitação | Ordenação por pedido; eventos em retry podem ser ultrapassados | TRANSCRICAO | [09:13] Larissa |
| FDD-RES-09 | docs/FDD.md | Integração | Graceful shutdown no padrão do server.ts | CODIGO | src/server.ts |
| FDD-RES-10 | docs/FDD.md | Requisito Não Funcional | Falha do worker não perde eventos (persistidos na outbox) | TRANSCRICAO | [09:06] Diego |
| FDD-OBS-01 | docs/FDD.md | Requisito Não Funcional | Logs estruturados no Pino existente | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Requisito Não Funcional | Log de auditoria do replay com userId | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-03 | docs/FDD.md | Integração | Redaction de *.secret e *.previousSecret no logger | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-04 | docs/FDD.md | Proposta (pendente de validação) | 🔶 Métricas definidas e derivadas de logs até haver stack (P-17) | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-05 | docs/FDD.md | Proposta (pendente de validação) | 🔶 Tracing por event_id ligado ao X-Request-Id (P-17) | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus chama publishWebhookEvent(tx, ...) após gravar o histórico | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | OrderStatus como fonte dos valores válidos do filtro | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Erros 404 do módulo estendem AppError (NotFoundError fixa NOT_FOUND) | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-04 | docs/FDD.md | Integração | BadRequestError e ConflictError com código customizado | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-05 | docs/FDD.md | Integração | Error middleware sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | authenticate e requireRole('ADMIN') | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | validate com schemas Zod do módulo | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Registro do módulo em buildControllers | CODIGO | src/app.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Rotas /webhooks e /admin/webhooks no agregador | CODIGO | src/routes/index.ts |
| FDD-INT-10 | docs/FDD.md | Integração | src/worker.ts replica bootstrap e shutdown | CODIGO | src/server.ts |
| FDD-INT-11 | docs/FDD.md | Integração | Worker usa createPrismaClient | CODIGO | src/config/database.ts |
| FDD-INT-12 | docs/FDD.md | Integração | Novas variáveis WEBHOOK_* no envSchema | CODIGO | src/config/env.ts |
| FDD-INT-13 | docs/FDD.md | Integração | Redaction das secrets no logger | CODIGO | src/shared/logger/index.ts |
| FDD-INT-14 | docs/FDD.md | Integração | paginated() nas listagens | CODIGO | src/shared/http/response.ts |
| FDD-INT-15 | docs/FDD.md | Integração | Novos models, enum e relações inversas | CODIGO | prisma/schema.prisma |
| FDD-INT-16 | docs/FDD.md | Integração | Scripts worker e worker:dev | CODIGO | package.json |
| FDD-INT-17 | docs/FDD.md | Integração | Limpeza das novas tabelas no beforeEach | CODIGO | tests/setup.ts |
| FDD-INT-18 | docs/FDD.md | Integração | Fábrica createTestWebhook | CODIGO | tests/helpers/factories.ts |
| FDD-INT-19 | docs/FDD.md | Decisão | Assinatura publishWebhookEvent(tx, order, fromStatus, toStatus) | TRANSCRICAO | [09:41] Bruno |
| FDD-INT-20 | docs/FDD.md | Decisão | Estrutura do novo módulo e entry-point do worker | TRANSCRICAO | [09:27] Bruno; [09:28] Bruno |
| FDD-ENV-01 | docs/FDD.md | Requisito Não Funcional | WEBHOOK_POLL_INTERVAL_MS = 2000 | TRANSCRICAO | [09:09] Diego |
| FDD-ENV-02 | docs/FDD.md | Requisito Não Funcional | WEBHOOK_HTTP_TIMEOUT_MS = 10000 | TRANSCRICAO | [09:42] Diego |
| FDD-ENV-03 | docs/FDD.md | Requisito Não Funcional | WEBHOOK_MAX_PAYLOAD_BYTES = 65536 | TRANSCRICAO | [09:24] Diego |
| FDD-ENV-04 | docs/FDD.md | Proposta (pendente de validação) | 🔶 WEBHOOK_BATCH_SIZE = 10 (P-22); reunião definiu só "batch pequeno" | TRANSCRICAO | [09:08] Diego |
| FDD-ENV-05 | docs/FDD.md | Proposta (pendente de validação) | 🔶 WEBHOOK_PROCESSING_TIMEOUT_MS = 60000 (P-14) | TRANSCRICAO | [09:08] Diego |
| FDD-DEP-01 | docs/FDD.md | Restrição | Nenhuma biblioteca nova | TRANSCRICAO | [09:29] Bruno |
| FDD-DEP-02 | docs/FDD.md | Dependência | fetch nativo e node:crypto do Node 20 (engines) | CODIGO | package.json |
| FDD-DEP-03 | docs/FDD.md | Dependência | Migration aditiva no MySQL existente | CODIGO | prisma/schema.prisma |
| FDD-DEP-04 | docs/FDD.md | Restrição | Contrato de PATCH /orders/:id/status inalterado | CODIGO | src/modules/orders/order.routes.ts |
| FDD-DEP-05 | docs/FDD.md | Dependência | Deploy de dois processos (API e worker) | TRANSCRICAO | [09:11] Diego |
| FDD-DEP-06 | docs/FDD.md | Dependência | Revisão de segurança antes do deploy | TRANSCRICAO | [09:46] Sofia |
| FDD-RISK-01 | docs/FDD.md | Risco | Aumento da latência de changeStatus | TRANSCRICAO | [09:04] Bruno |
| FDD-RISK-02 | docs/FDD.md | Risco | Evento posterior ultrapassa evento em retry do mesmo pedido | TRANSCRICAO | [09:12] Diego |
| FDD-RISK-03 | docs/FDD.md | Risco | Worker parado sem ninguém perceber | TRANSCRICAO | [09:11] Diego |
| FDD-RISK-04 | docs/FDD.md | Risco | Crescimento contínuo da outbox e do histórico | TRANSCRICAO | [09:07] Bruno |
| FDD-RISK-05 | docs/FDD.md | Risco | Secret armazenada em texto no banco | TRANSCRICAO | [09:22] Diego |
| FDD-RISK-06 | docs/FDD.md | Risco | Duplicatas após timeout | TRANSCRICAO | [09:25] Diego |
| FDD-RISK-07 | docs/FDD.md | Risco | Diferença de relógio afetando X-Timestamp | TRANSCRICAO | [09:44] Diego |
| FDD-RISK-08 | docs/FDD.md | Risco | Endpoint lento ocupando o worker | TRANSCRICAO | [09:42] Diego |
| FDD-RISK-09 | docs/FDD.md | Risco | Testes dependentes de tempo | CODIGO | vitest.config.ts |
| FDD-PROP-01 | docs/FDD.md | Proposta (pendente de validação) | 🔶 P-01: customerId no corpo do POST e filtro obrigatório na listagem | TRANSCRICAO | [09:32] Larissa |
| FDD-PROP-02 | docs/FDD.md | Proposta (pendente de validação) | 🔶 P-02: caminhos /api/v1/webhooks e /webhooks/:id | TRANSCRICAO | [09:33] Bruno; [09:34] Marcos |
| FDD-PROP-03 | docs/FDD.md | Proposta (pendente de validação) | 🔶 P-03: arquivo webhook.processor.ts | TRANSCRICAO | [09:28] Bruno |
| FDD-PROP-04 | docs/FDD.md | Proposta (pendente de validação) | 🔶 P-04: payload só com os campos definidos; demais em aberto | TRANSCRICAO | [09:43] Diego |
| FDD-PROP-05 | docs/FDD.md | Proposta (pendente de validação) | 🔶 P-05: 1 envio inicial + 5 reenvios | TRANSCRICAO | [09:17] Larissa; [09:17] Diego |
| FDD-PROP-06 | docs/FDD.md | Proposta (pendente de validação) | 🔶 P-06: histórico paginado, máx. 100 por página | TRANSCRICAO | [09:34] Marcos |
| FDD-PROP-07 | docs/FDD.md | Proposta (pendente de validação) | 🔶 P-07: catálogo de códigos WEBHOOK_ | TRANSCRICAO | [09:28] Bruno |
| FDD-PROP-08 | docs/FDD.md | Proposta (pendente de validação) | 🔶 P-08: POST /webhooks/:id/rotate-secret | TRANSCRICAO | [09:21] Sofia |
| FDD-PROP-09 | docs/FDD.md | Proposta (pendente de validação) | 🔶 P-09: duas assinaturas durante as 24 h de rotação | TRANSCRICAO | [09:21] Sofia |
| FDD-PROP-10 | docs/FDD.md | Proposta (pendente de validação) | 🔶 P-10: X-Signature sha256=<hex> sobre o corpo cru; X-Timestamp ISO 8601 | TRANSCRICAO | [09:22] Sofia; [09:44] Diego |
| FDD-PROP-11 | docs/FDD.md | Proposta (pendente de validação) | 🔶 P-11: secret de 32 bytes hex, exibida só na criação e rotação | TRANSCRICAO | [09:31] Marcos |
| FDD-PROP-12 | docs/FDD.md | Proposta (pendente de validação) | 🔶 P-12: sucesso = qualquer 2xx | TRANSCRICAO | [09:42] Diego |
| FDD-PROP-13 | docs/FDD.md | Proposta (pendente de validação) | 🔶 P-13: DELETE em cascata; desativação descarta pendentes | TRANSCRICAO | [09:33] Bruno; [09:21] Bruno |
| FDD-PROP-14 | docs/FDD.md | Proposta (pendente de validação) | 🔶 P-14: reclaim de processando após 60 s | TRANSCRICAO | [09:08] Diego |
| FDD-PROP-15 | docs/FDD.md | Proposta (pendente de validação) | 🔶 P-15: tabela webhook_deliveries por tentativa | TRANSCRICAO | [09:34] Marcos |
| FDD-PROP-16 | docs/FDD.md | Proposta (pendente de validação) | 🔶 P-16: replay mantém event_id e zera tentativas | TRANSCRICAO | [09:25] Diego; [09:18] Diego |
| FDD-PROP-17 | docs/FDD.md | Proposta (pendente de validação) | 🔶 P-17: métricas via logs e event_id como correlação | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-PROP-18 | docs/FDD.md | Proposta (pendente de validação) | 🔶 P-18: limite de 64 KB validado no worker, excedente direto para DLQ | TRANSCRICAO | [09:24] Larissa |
| FDD-PROP-19 | docs/FDD.md | Proposta (pendente de validação) | 🔶 P-19: uma linha de outbox por evento × webhook | TRANSCRICAO | [09:44] Sofia |
| FDD-PROP-20 | docs/FDD.md | Proposta (pendente de validação) | 🔶 P-20: esquema https checado no service, pois validate converte Zod em VALIDATION_ERROR | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-PROP-21 | docs/FDD.md | Proposta (pendente de validação) | 🔶 P-21: outbox sem FK para orders; pedidos PENDING/CANCELLED são removíveis | CODIGO | src/modules/orders/order.service.ts |
| FDD-PROP-22 | docs/FDD.md | Proposta (pendente de validação) | 🔶 P-22: tamanho de lote e variáveis de ambiente | TRANSCRICAO | [09:08] Diego |
| FDD-PROP-23 | docs/FDD.md | Proposta (pendente de validação) | 🔶 P-23: cliente HTTP injetável no WebhookProcessor (padrão de DI por construtor) | CODIGO | src/app.ts |
| PRD-CTX-01 | docs/PRD.md | Contexto | Sistema de webhooks de saída para o cliente | TRANSCRICAO | [09:02] Marcos; [09:03] Sofia |
| PRD-CTX-02 | docs/PRD.md | Contexto | Novo módulo e processo de entrega ao lado da API, mesmo banco | TRANSCRICAO | [09:11] Larissa; [09:11] Diego |
| PRD-CTX-03 | docs/PRD.md | Restrição | Sem nova infraestrutura | TRANSCRICAO | [09:07] Diego |
| PRD-PROB-01 | docs/PRD.md | Contexto | Polling em GET /orders deixa a integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-02 | docs/PRD.md | Contexto | OMS sem mecanismo de eventos ou notificação externa | CODIGO | src/app.ts |
| PRD-PROB-03 | docs/PRD.md | Risco | Atlas pode migrar para concorrente | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-04 | docs/PRD.md | Restrição | Tempo real = abaixo de 10 s | TRANSCRICAO | [09:02] Marcos |
| PRD-PUB-01 | docs/PRD.md | Contexto | Clientes B2B integradores (Atlas, MaxDistribuição, Nova Cargo) | TRANSCRICAO | [09:00] Marcos |
| PRD-PUB-02 | docs/PRD.md | Contexto | Usuários da API que representam o cliente, com JWT da plataforma | TRANSCRICAO | [09:32] Marcos |
| PRD-PUB-03 | docs/PRD.md | Contexto | Administradores (role ADMIN) para replay | TRANSCRICAO | [09:36] Sofia |
| PRD-PUB-04 | docs/PRD.md | Contexto | Engenharia e operação usam histórico e dead letter para diagnóstico | TRANSCRICAO | [09:18] Diego |
| PRD-OBJ-01 | docs/PRD.md | Objetivo/Métrica | Início da entrega em menos de 10 s | TRANSCRICAO | [09:02] Marcos; [09:10] Larissa |
| PRD-OBJ-02 | docs/PRD.md | Objetivo/Métrica | 100% das mudanças com assinatura geram evento | TRANSCRICAO | [09:41] Diego |
| PRD-OBJ-03 | docs/PRD.md | Objetivo/Métrica | Janela de reentrega de ~15 h | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-04 | docs/PRD.md | Objetivo/Métrica | Produção até o fim de novembro, em 3 sprints | TRANSCRICAO | [09:45] Marcos; [09:47] Larissa |
| PRD-OBJ-05 | docs/PRD.md | Estimativa (pendente de validação) | 🔶 3 de 3 clientes solicitantes integrados; meta não definida na reunião | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-06 | docs/PRD.md | Estimativa (pendente de validação) | 🔶 Redução do polling em GET /orders; meta a definir com os clientes | TRANSCRICAO | [09:00] Marcos |
| PRD-INC-01 | docs/PRD.md | Escopo | Notificação order.status_changed | TRANSCRICAO | [09:43] Diego |
| PRD-INC-02 | docs/PRD.md | Escopo | Cadastro, edição, remoção e listagem de webhooks | TRANSCRICAO | [09:31] Marcos; [09:33] Bruno |
| PRD-INC-03 | docs/PRD.md | Escopo | Escolha dos status notificados por webhook | TRANSCRICAO | [09:33] Marcos |
| PRD-INC-04 | docs/PRD.md | Escopo | Secret única com rotação e 24 h de convivência | TRANSCRICAO | [09:21] Sofia |
| PRD-INC-05 | docs/PRD.md | Escopo | Assinatura HMAC-SHA256 e HTTPS | TRANSCRICAO | [09:22] Sofia; [09:23] Sofia |
| PRD-INC-06 | docs/PRD.md | Escopo | Reentrega com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| PRD-INC-07 | docs/PRD.md | Escopo | Dead letter com reprocessamento manual por admin | TRANSCRICAO | [09:18] Diego; [09:36] Sofia |
| PRD-INC-08 | docs/PRD.md | Escopo | Histórico de entregas por webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-INC-09 | docs/PRD.md | Escopo | Identificador único de evento para deduplicação | TRANSCRICAO | [09:25] Diego |
| PRD-INC-10 | docs/PRD.md | Escopo | Documentação no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos; [09:40] Marcos |
| PRD-OOS-01 | docs/PRD.md | Fora de escopo | Webhooks de entrada (descartado) | TRANSCRICAO | [09:02] Marcos |
| PRD-OOS-02 | docs/PRD.md | Fora de escopo | Alerta por e-mail (adiado) | TRANSCRICAO | [09:37] Larissa; [09:38] Marcos |
| PRD-OOS-03 | docs/PRD.md | Fora de escopo | Dashboard / painel visual (descartado nesta fase) | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de escopo | Rate limiting (adiado / em observação) | TRANSCRICAO | [09:39] Diego; [09:39] Larissa |
| PRD-OOS-05 | docs/PRD.md | Fora de escopo | Arquivamento de eventos entregues | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-06 | docs/PRD.md | Fora de escopo | Exactly-once e ordenação global (descartado) | TRANSCRICAO | [09:25] Diego; [09:13] Larissa |
| PRD-OOS-07 | docs/PRD.md | Fora de escopo | Múltiplos workers (adiado) | TRANSCRICAO | [09:13] Diego |
| PRD-OOS-08 | docs/PRD.md | Fora de escopo | Reprocessamento automático da dead letter (não previsto) | TRANSCRICAO | [09:18] Diego |
| PRD-EVO-01 | docs/PRD.md | Evolução futura | Alerta por e-mail em falhas repetidas | TRANSCRICAO | [09:37] Larissa |
| PRD-EVO-02 | docs/PRD.md | Evolução futura | Rate limiting de saída | TRANSCRICAO | [09:39] Diego |
| PRD-EVO-03 | docs/PRD.md | Evolução futura | Escala para múltiplos workers | TRANSCRICAO | [09:13] Diego |
| PRD-EVO-04 | docs/PRD.md | Evolução futura | Restrição de permissões no cadastro | TRANSCRICAO | [09:37] Sofia |
| PRD-EVO-05 | docs/PRD.md | Evolução futura | Arquivamento de eventos entregues | TRANSCRICAO | [09:08] Diego |
| PRD-EVO-06 | docs/PRD.md | Evolução futura | Painel visual pelo time de frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Notificação ao cliente quando o status do pedido muda | TRANSCRICAO | [09:00] Marcos; [09:43] Diego |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Cadastro de webhook com secret gerada pela plataforma | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Edição de webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Remoção de webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Listagem de webhooks do cliente | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Filtro de eventos por webhook | TRANSCRICAO | [09:33] Marcos; [09:34] Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Histórico de entregas com sucesso/falha, payload, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Rotação de secret com 24 h de convivência | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Reentrega automática com backoff | TRANSCRICAO | [09:15] Diego; [09:17] Larissa |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Dead letter com payload, motivo e data/hora | TRANSCRICAO | [09:18] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Reprocessamento manual por ADMIN com registro de quem executou | TRANSCRICAO | [09:36] Sofia; [09:36] Larissa |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Exigência de HTTPS no cadastro e na edição | TRANSCRICAO | [09:23] Sofia |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Identificação e autenticidade: X-Event-Id, X-Webhook-Id, X-Signature, X-Timestamp | TRANSCRICAO | [09:44] Diego; [09:44] Sofia |
| PRD-RES-01 | docs/PRD.md | Restrição | Itens do pedido não são enviados na notificação | TRANSCRICAO | [09:43] Diego |
| PRD-RES-02 | docs/PRD.md | Restrição | Qualquer usuário autenticado gerencia webhooks nesta fase | TRANSCRICAO | [09:37] Sofia |
| PRD-RES-03 | docs/PRD.md | Restrição | Se o registro do evento falhar, a mudança de status não é efetivada | TRANSCRICAO | [09:40] Bruno |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Notificação iniciada em menos de 10 s; verificação a cada 2 s | TRANSCRICAO | [09:09] Diego |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 s por tentativa | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Limite de 64 KB, recusado sem truncar | TRANSCRICAO | [09:23] Sofia; [09:24] Larissa |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Mudança de status não fica mais lenta por clientes lentos | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Toda mudança com assinatura gera evento (atomicidade) | TRANSCRICAO | [09:41] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Ordem por pedido com um único processo de entrega | TRANSCRICAO | [09:12] Diego; [09:13] Larissa |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Reentrega por ~15 h antes da dead letter | TRANSCRICAO | [09:17] Diego |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Processo de entrega separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Assinatura HMAC-SHA256 em todas as notificações | TRANSCRICAO | [09:20] Sofia |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Secret única por webhook com rotação de 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-12 | docs/PRD.md | Requisito Não Funcional | Somente endpoints HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-13 | docs/PRD.md | Requisito Não Funcional | Secret exibida no cadastro e na rotação, nunca em logs | TRANSCRICAO | [09:31] Marcos |
| PRD-NFR-14 | docs/PRD.md | Requisito Não Funcional | Reprocessamento restrito a administradores | TRANSCRICAO | [09:36] Sofia |
| PRD-NFR-15 | docs/PRD.md | Requisito Não Funcional | Revisão de segurança obrigatória antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-NFR-16 | docs/PRD.md | Requisito Não Funcional | Registro de quem executou cada reprocessamento | TRANSCRICAO | [09:36] Sofia |
| PRD-NFR-17 | docs/PRD.md | Requisito Não Funcional | Logs estruturados com o logger existente | TRANSCRICAO | [09:29] Bruno |
| PRD-NFR-18 | docs/PRD.md | Requisito Não Funcional | Nenhuma mudança nos endpoints existentes | CODIGO | src/routes/index.ts |
| PRD-NFR-19 | docs/PRD.md | Requisito Não Funcional | API REST JSON versionada em /api/v1 | CODIGO | src/app.ts |
| PRD-NFR-20 | docs/PRD.md | Requisito Não Funcional | Padrões do projeto com códigos WEBHOOK_ | TRANSCRICAO | [09:29] Larissa |
| PRD-ARQ-01 | docs/PRD.md | Decisão | Outbox + processo de entrega + módulo de gestão (resumo da arquitetura) | TRANSCRICAO | [09:48] Larissa |
| PRD-DEC-01 | docs/PRD.md | Trade-off | Outbox no MySQL: consistência vs banco como fila | TRANSCRICAO | [09:08] Larissa |
| PRD-DEC-02 | docs/PRD.md | Trade-off | Worker separado com 2 s: latência vs processo extra | TRANSCRICAO | [09:10] Larissa |
| PRD-DEC-03 | docs/PRD.md | Trade-off | Reentrega e dead letter: resiliência vs atraso de até ~15 h | TRANSCRICAO | [09:17] Larissa |
| PRD-DEC-04 | docs/PRD.md | Trade-off | HMAC por webhook: segurança vs verificação no cliente | TRANSCRICAO | [09:22] Sofia |
| PRD-DEC-05 | docs/PRD.md | Trade-off | At-least-once: simplicidade vs deduplicação no cliente | TRANSCRICAO | [09:26] Larissa |
| PRD-DEC-06 | docs/PRD.md | Trade-off | Reuso de padrões: prazo vs herdar limitações | TRANSCRICAO | [09:30] Larissa |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança de pelo menos 2 dias úteis | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Documentação no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos; [09:40] Marcos |
| PRD-DEP-03 | docs/PRD.md | Dependência | Confirmação do prazo com a Atlas e atualização dos clientes | TRANSCRICAO | [09:47] Marcos; [09:49] Marcos |
| PRD-DEP-04 | docs/PRD.md | Dependência | Validação das propostas P-01 a P-23 na sessão de revisão | TRANSCRICAO | [09:50] Larissa |
| PRD-DEP-05 | docs/PRD.md | Dependência | Novo processo em produção | TRANSCRICAO | [09:11] Diego |
| PRD-DEP-06 | docs/PRD.md | Dependência | Clientes com HTTPS, verificação HMAC e deduplicação | TRANSCRICAO | [09:25] Diego |
| PRD-RISK-01 | docs/PRD.md | Estimativa (pendente de validação) | 🔶 Perda da Atlas por atraso — probabilidade média estimada | TRANSCRICAO | [09:00] Marcos |
| PRD-RISK-02 | docs/PRD.md | Estimativa (pendente de validação) | 🔶 Vazamento de secret — probabilidade média (já ocorreu antes) | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-03 | docs/PRD.md | Estimativa (pendente de validação) | 🔶 Indisponibilidade prolongada — probabilidade média (manutenção de 2 h já vista) | TRANSCRICAO | [09:16] Diego |
| PRD-RISK-04 | docs/PRD.md | Estimativa (pendente de validação) | 🔶 Cliente processando duplicados — probabilidade média | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-05 | docs/PRD.md | Estimativa (pendente de validação) | 🔶 Excesso de notificações a um cliente — probabilidade baixa a média | TRANSCRICAO | [09:38] Diego |
| PRD-RISK-06 | docs/PRD.md | Estimativa (pendente de validação) | 🔶 Processo de entrega parado sem detecção — probabilidade baixa | TRANSCRICAO | [09:11] Diego |
| PRD-QA-01 | docs/PRD.md | Questão em aberto | Cliente no corpo ou no caminho | TRANSCRICAO | [09:32] Larissa |
| PRD-QA-02 | docs/PRD.md | Questão em aberto | Campos do pedido além do total | TRANSCRICAO | [09:43] Diego |
| PRD-QA-03 | docs/PRD.md | Questão em aberto | 5 tentativas incluem o envio inicial? | TRANSCRICAO | [09:17] Larissa |
| PRD-QA-04 | docs/PRD.md | Questão em aberto | Limite do histórico de entregas | TRANSCRICAO | [09:34] Marcos |

## Resumo por fonte

| Fonte | Linhas | Percentual |
| --- | --- | --- |
| TRANSCRICAO | 379 | 88% |
| CODIGO | 52 | 12% |
| **Total** | **431** | **100%** |
