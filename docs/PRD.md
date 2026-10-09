# PRD: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| Versão | v1 |
| Data | 28/09/2026 |
| Responsável | Marcos (Product Manager) |
| Revisores | Larissa (Tech Lead), Bruno (time de Pedidos), Diego (time de Plataforma), Sofia (Segurança) |
| Status | Em revisão |
| Documentos relacionados | [RFC-001](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) · [Ata da reunião](levantamento-tecnico/ATA-REUNIAO-WEBHOOKS.md) |

> **Convenção:** itens marcados com **🔶** são estimativas ou metas propostas por este PRD que **não** foram definidas na reunião e precisam ser validadas pelos revisores. Todo o restante vem das decisões registradas na ata e nas ADRs.

---

## 1. Resumo e contexto da feature

A feature adiciona ao Order Management System (OMS) um **sistema de webhooks de saída**: sempre que o status de um pedido muda, a plataforma envia automaticamente uma notificação HTTP para o endpoint cadastrado pelo cliente, em menos de 10 segundos, assinada e com garantia de entrega at-least-once. O cliente gerencia seus webhooks pela API: cadastra, escolhe quais status quer receber, consulta o histórico de entregas e rotaciona a secret. Administradores podem reprocessar entregas que falharam em definitivo.

**Onde será implantada**

- No OMS existente (Node.js + TypeScript, MySQL), como um novo módulo da API REST versionada (`/api/v1`).
- Com um novo processo de background (worker), responsável pelas entregas, que roda ao lado da API e usa o mesmo banco.
- Sem nova infraestrutura: nenhuma fila, cache ou serviço externo adicional.

---

## 2. Problema e motivação

**Problemas priorizados**

- **Integração lenta e cara para os clientes.** Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) precisam saber quando o status dos seus pedidos muda. Hoje fazem polling periódico em `GET /orders` para detectar mudanças, o que deixa a integração lenta e cara para eles.
- **Ausência de notificação.** O OMS não possui nenhum mecanismo de eventos ou notificação externa; toda informação precisa ser buscada pelo cliente.
- **Risco comercial concreto.** Os três clientes fizeram um pedido formal. A Atlas sinalizou que pode migrar para um concorrente se a solução não for entregue, e pediu a entrega até o **fim de novembro**.

**Motivação**

Para esses clientes, "tempo real" significa **qualquer coisa abaixo de 10 segundos**. O essencial é deixar de consultar e atualizar manualmente. Uma notificação ativa elimina o polling, reduz a latência de percepção de mudanças e retém clientes estratégicos.

---

## 3. Público-alvo e cenários de uso

**Público-alvo**

| Público | Necessidade |
| --- | --- |
| **Clientes B2B integradores** (inicialmente Atlas Comercial, MaxDistribuição e Nova Cargo) | Receber mudanças de status dos seus pedidos sem polling, de forma segura e confiável |
| **Usuários da API que representam o cliente** | Cadastrar e gerenciar webhooks pela API, autenticados com o JWT da plataforma |
| **Administradores da plataforma** (role ADMIN) | Reprocessar entregas que falharam em definitivo, com trilha de auditoria |
| **Times de engenharia e operação** | Diagnosticar entregas por histórico, logs e dead letter |

**Cenários de uso chave**

1. **Notificação de despacho:** o pedido de um cliente passa para `SHIPPED`; em poucos segundos o sistema do cliente recebe a notificação e atualiza o rastreio, sem consultar a API.
2. **Assinatura seletiva:** o cliente só quer saber de `SHIPPED` e `DELIVERED`; mudanças para outros status não geram notificação para ele.
3. **Manutenção do cliente:** o endpoint do cliente fica 2 horas fora do ar por manutenção planejada; as notificações são reenviadas automaticamente e entregues quando ele volta.
4. **Vazamento de secret:** o cliente descobre que vazou a secret num log; pede uma nova pela API e tem 24 horas para atualizar seus sistemas sem perder notificações.
5. **Investigação de entrega:** o cliente questiona uma notificação que não chegou; ele ou o suporte consultam o histórico de entregas do webhook, com status, resposta e tempo de resposta.
6. **Recuperação de falha definitiva:** depois de ~15 horas de falhas, um evento vai para a dead letter; um administrador corrige a causa com o cliente e reprocessa o evento.
7. **Múltiplos endpoints:** um cliente com mais de um webhook cadastrado identifica qual cadastro originou cada entrega pelo header `X-Webhook-Id`.

---

## 4. Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta | Origem |
| --- | --- | --- | --- |
| Notificar mudanças em "tempo real" | Tempo entre a mudança de status e o início da entrega | **< 10 s** (intervalo de leitura de 2 s) | Decidido |
| Não perder nenhuma mudança de status | Percentual de mudanças de status (com assinatura ativa) que geram evento | **100%** | Decidido |
| Tolerar indisponibilidade do cliente | Janela de reentrega automática antes da dead letter | **~15 h**, cobrindo manutenções de 2 h | Decidido |
| Entregar no prazo pedido pelo cliente | Data de disponibilização em produção | **Até o fim de novembro** (3 sprints, com revisão de segurança) | Decidido |
| Reter os clientes solicitantes | Clientes solicitantes integrados via webhook | 🔶 **3 de 3** | Estimativa do PRD |
| Reduzir o polling dos clientes integrados | Chamadas de polling a `GET /orders` pelos clientes integrados | 🔶 Meta a definir com os clientes após a integração | Proposta do PRD |

---

## 5. Escopo

### Incluso

- Notificação automática de mudança de status de pedido (evento `order.status_changed`).
- Cadastro, edição, remoção e listagem de webhooks por cliente, via API autenticada.
- Escolha, por webhook, dos status de pedido que serão notificados.
- Secret única por webhook, gerada pela plataforma, com rotação e 24 h de convivência.
- Assinatura HMAC-SHA256 de todas as entregas e exigência de HTTPS.
- Reentrega automática com backoff (1 min, 5 min, 30 min, 2 h, 12 h).
- Dead letter para entregas que falharam em definitivo, com reprocessamento manual por administrador.
- Histórico de entregas por webhook.
- Identificação única de cada evento para deduplicação no cliente.
- Documentação da integração no portal do desenvolvedor.

### Fora de escopo

| Item | Situação |
| --- | --- |
| **Webhooks de entrada** (clientes enviando eventos para a plataforma) | **Descartado.** Os clientes querem apenas receber |
| **Alerta por e-mail** ao cliente quando o webhook falha repetidamente | **Adiado** para a próxima fase, depois de medir o impacto |
| **Dashboard / painel visual** para o cliente ver seus webhooks | **Descartado nesta fase.** Só endpoints; o painel é um projeto separado do time de frontend |
| **Rate limiting** de envio para o cliente | **Adiado / em observação.** Implementar se virar problema |
| **Arquivamento** de eventos entregues (após ~30 dias) | **Fora desta feature** |
| **Entrega exactly-once** e **ordenação global** de eventos | **Descartado.** Garantia at-least-once, com ordem apenas por pedido |
| **Múltiplos workers em paralelo** | **Adiado.** Problema para quando for preciso escalar |
| **Reprocessamento automático** da dead letter | **Não previsto.** O replay é manual |

### Evoluções futuras

- Alerta por e-mail em falhas repetidas.
- Rate limiting de saída.
- Escala para múltiplos workers, preservando a ordem por pedido.
- Restrição de permissões no cadastro de webhooks.
- Arquivamento de eventos entregues.
- Painel visual, pelo time de frontend.

---

## 6. Requisitos funcionais

Os contratos de API (caminhos, payloads, códigos de status) estão detalhados no [FDD](FDD.md#6-contratos-públicos).

### RF-001 Notificação de mudança de status do pedido

Enviar uma notificação ao endpoint do cliente sempre que um pedido dele mudar para um status que ele assina.

**Fluxo principal**

- Um usuário muda o status de um pedido pelo fluxo existente.
- O sistema registra o evento junto com a mudança de status.
- Em poucos segundos, o sistema envia ao endpoint do cliente uma notificação com: identificador do evento, tipo (`order.status_changed`), data/hora, pedido, número do pedido, status anterior, novo status, cliente e valor total.

**Fluxos alternativos e exceções**

- Se nenhum webhook do cliente assina o novo status, nenhuma notificação é gerada.
- Os itens do pedido não são enviados; o cliente consulta o pedido pela API se precisar de detalhes.
- Se o registro do evento falhar, a mudança de status também não é efetivada.

**Erros previstos**

- Evento acima de 64 KB não é enviado.

**Prioridade:** Alta — escopo acordado da fase 1

---

### RF-002 Cadastro de webhook

Permitir que o cliente cadastre um endpoint para receber notificações.

**Fluxo principal**

- O usuário autenticado informa o cliente, a URL do endpoint e a lista de status que deseja receber.
- O sistema cria o webhook ativo e gera uma secret exclusiva.
- A secret é devolvida na resposta do cadastro.

**Fluxos alternativos e exceções**

- Qualquer usuário autenticado pode cadastrar webhooks nesta fase.

**Erros previstos**

- URL que não usa HTTPS.
- Cliente inexistente.
- Lista de status vazia ou com status inválido.

**Prioridade:** Alta — escopo acordado da fase 1

---

### RF-003 Edição de webhook

Permitir alterar a URL, os status assinados e o estado ativo de um webhook.

**Fluxo principal**

- O usuário autenticado altera um ou mais campos do webhook.
- As alterações valem para as próximas mudanças de status.

**Fluxos alternativos e exceções**

- Desativar o webhook interrompe o envio de novas notificações para ele.

**Erros previstos**

- Webhook inexistente.
- Nova URL sem HTTPS.

**Prioridade:** Alta — escopo acordado da fase 1

---

### RF-004 Remoção de webhook

Permitir remover um webhook cadastrado.

**Fluxo principal**

- O usuário autenticado remove o webhook e ele deixa de receber notificações.

**Fluxos alternativos e exceções**

- 🔶 Para preservar o histórico de entregas, recomenda-se desativar em vez de remover (comportamento detalhado no FDD, proposta P-13).

**Erros previstos**

- Webhook inexistente.

**Prioridade:** Alta — escopo acordado da fase 1

---

### RF-005 Listagem de webhooks do cliente

Listar os webhooks cadastrados para um cliente.

**Fluxo principal**

- O usuário autenticado consulta os webhooks de um cliente e recebe a lista com URL, status assinados e estado ativo.

**Fluxos alternativos e exceções**

- Cliente sem webhooks retorna lista vazia.
- A secret nunca aparece na listagem.

**Erros previstos**

- Identificação de cliente ausente ou inválida.

**Prioridade:** Alta — escopo acordado da fase 1

---

### RF-006 Filtro de eventos por webhook

Permitir que cada webhook escolha quais status de pedido deseja receber.

**Fluxo principal**

- No cadastro ou na edição, o usuário informa os status de interesse (ex.: apenas `SHIPPED` e `DELIVERED`).
- Somente mudanças para esses status geram notificação para aquele webhook.

**Fluxos alternativos e exceções**

- Um cliente com vários webhooks pode ter filtros diferentes em cada um.

**Erros previstos**

- Status inexistente na lista.

**Prioridade:** Alta — escopo acordado da fase 1

---

### RF-007 Histórico de entregas

Permitir consultar as entregas feitas para um webhook.

**Fluxo principal**

- O usuário consulta o histórico de um webhook e vê, por entrega: sucesso ou falha, payload enviado, resposta recebida e tempo de resposta.

**Fluxos alternativos e exceções**

- 🔶 Histórico paginado, com até 100 entregas por página, das mais recentes para as mais antigas (proposta P-06 do FDD).

**Erros previstos**

- Webhook inexistente.

**Prioridade:** Alta — escopo acordado da fase 1

---

### RF-008 Rotação de secret

Permitir que o cliente obtenha uma nova secret sem interromper a integração.

**Fluxo principal**

- O usuário solicita a rotação da secret de um webhook.
- O sistema gera e devolve uma nova secret.
- A secret anterior continua válida por **24 horas** em paralelo e depois é invalidada.

**Fluxos alternativos e exceções**

- Durante as 24 horas, as notificações continuam verificáveis tanto com a secret antiga quanto com a nova.

**Erros previstos**

- Webhook inexistente.

**Prioridade:** Alta — escopo acordado da fase 1

---

### RF-009 Reentrega automática

Reenviar automaticamente notificações que falharam.

**Fluxo principal**

- Se o endpoint do cliente responder com erro, não responder em 10 segundos ou estiver inacessível, a entrega é reagendada.
- As reentregas seguem os intervalos de 1 min, 5 min, 30 min, 2 h e 12 h.

**Fluxos alternativos e exceções**

- Se uma reentrega tiver sucesso, o ciclo termina.

**Erros previstos**

- Endpoint indisponível, lento ou respondendo com erro.

**Prioridade:** Alta — escopo acordado da fase 1

---

### RF-010 Dead letter

Separar e preservar notificações que falharam em definitivo.

**Fluxo principal**

- Esgotadas as tentativas, a notificação é movida para a dead letter com o payload, o motivo da falha e a data/hora.

**Fluxos alternativos e exceções**

- Eventos na dead letter não são reenviados automaticamente.

**Erros previstos**

- Nenhum para o cliente; a falha fica registrada para análise.

**Prioridade:** Alta — escopo acordado da fase 1

---

### RF-011 Reprocessamento manual da dead letter

Permitir que um administrador reprocesse uma notificação da dead letter.

**Fluxo principal**

- Um usuário com role **ADMIN** solicita o reprocessamento de um evento da dead letter.
- O evento volta para a fila de entrega como pendente.
- O sistema registra **quem** fez o reprocessamento, para auditoria.

**Fluxos alternativos e exceções**

- 🔶 A notificação reprocessada mantém o mesmo identificador de evento (proposta P-16 do FDD).

**Erros previstos**

- Usuário sem role ADMIN.
- Evento inexistente na dead letter.
- Evento já reprocessado.

**Prioridade:** Alta — escopo acordado da fase 1

---

### RF-012 Exigência de HTTPS

Aceitar apenas endpoints seguros.

**Fluxo principal**

- No cadastro e na edição, a URL precisa usar `https`.

**Fluxos alternativos e exceções**

- Não há exceção para `http`.

**Erros previstos**

- Cadastro ou edição com URL `http` é recusado com erro de validação.

**Prioridade:** Alta — escopo acordado da fase 1

---

### RF-013 Identificação e autenticidade da notificação

Permitir que o cliente identifique, valide e deduplique cada notificação.

**Fluxo principal**

- Toda notificação leva:
  - o identificador único do evento (`X-Event-Id`);
  - o identificador do webhook (`X-Webhook-Id`);
  - a assinatura HMAC-SHA256 do conteúdo (`X-Signature`);
  - a data/hora do envio (`X-Timestamp`).
- O cliente valida a assinatura com sua secret e descarta eventos já processados pelo identificador.

**Fluxos alternativos e exceções**

- O mesmo evento pode chegar mais de uma vez (garantia at-least-once); o identificador se mantém em todas as tentativas.

**Erros previstos**

- Assinatura inválida deve ser rejeitada pelo cliente.

**Prioridade:** Alta — escopo acordado da fase 1

---

## 7. Requisitos não funcionais

**Performance e latência**

- Notificação iniciada em menos de 10 s após a mudança de status; o sistema verifica novos eventos a cada 2 s.
- Timeout de 10 s por tentativa de entrega.
- Notificações limitadas a 64 KB; acima disso são recusadas, sem truncamento.
- A mudança de status de um pedido não pode ficar mais lenta por causa de clientes lentos ou fora do ar.

**Confiabilidade e integridade**

- Toda mudança de status com assinatura ativa gera evento; se o evento não puder ser registrado, a mudança de status não é efetivada.
- Garantia de entrega **at-least-once**.
- Ordem das notificações garantida **por pedido**, com um único processo de entrega; não há ordenação global.
- Reentrega por ~15 h antes da dead letter.
- O processo de entrega roda separado da API: reinícios da API não interrompem as entregas.

**Segurança**

- Assinatura HMAC-SHA256 de todas as notificações.
- Secret única por webhook, nunca global; rotação com 24 h de convivência.
- Somente endpoints HTTPS.
- A secret é exibida apenas no cadastro e na rotação, e nunca aparece em logs.
- Reprocessamento da dead letter restrito a administradores.
- Revisão de segurança obrigatória antes do deploy.

**Auditoria**

- Registro de quem executou cada reprocessamento da dead letter.
- Histórico de entregas consultável por webhook.

**Observabilidade**

- Logs estruturados de publicação, entrega, falha, dead letter e reprocessamento.
- Métricas e tracing conforme a [seção 9 do FDD](FDD.md#9-observabilidade).

**Compatibilidade**

- Nenhuma mudança nos endpoints existentes da API.
- Nova API no padrão REST JSON versionado (`/api/v1`) já usado pela plataforma.

**Manutenibilidade**

- A feature segue os padrões do projeto: estrutura de módulos, validação, tratamento de erros com códigos `WEBHOOK_*` e logger.

---

## 8. Arquitetura e abordagem

- A mudança de status registra o evento numa tabela de saída (**outbox**), no mesmo banco e na mesma transação.
- Um **processo de entrega separado** lê essa tabela a cada 2 s e envia as notificações, com reentrega e dead letter.
- A **API de gestão** é um novo módulo no padrão dos módulos existentes.

Visão técnica completa na [RFC-001](RFC.md) e detalhamento de implementação no [FDD](FDD.md).

---

## 9. Decisões e trade-offs principais

### Decisão: Outbox no MySQL existente ([ADR-001](adrs/ADR-001-outbox-no-mysql.md))

- **Justificativa:** o evento é registrado na mesma transação da mudança de status, então nunca há status alterado sem notificação; não exige nova infraestrutura.
- **Trade-off:** o banco passa a funcionar também como fila, e uma falha ao registrar o evento impede a mudança de status.

### Decisão: Processo de entrega separado com leitura a cada 2 s ([ADR-002](adrs/ADR-002-worker-separado-com-polling.md))

- **Justificativa:** atende com folga o requisito de menos de 10 s, sem depender de recursos que o MySQL não oferece, e sem ser afetado por reinícios da API.
- **Trade-off:** até 2 s de latência adicional; mais um processo para operar; ordem garantida só por pedido enquanto houver um único processo.

### Decisão: Reentrega com backoff e dead letter ([ADR-003](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md))

- **Justificativa:** cobre indisponibilidades reais de horas sem manter eventos pendentes para sempre.
- **Trade-off:** uma notificação pode chegar com até ~15 h de atraso, e o que cai na dead letter depende de ação manual.

### Decisão: HMAC-SHA256 com secret por webhook ([ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md))

- **Justificativa:** padrão de mercado, fácil de verificar pelo cliente; um vazamento afeta um único webhook.
- **Trade-off:** o cliente precisa implementar a verificação; a plataforma passa a gerenciar secrets e sua rotação.

### Decisão: Entrega at-least-once com identificador de evento ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md))

- **Justificativa:** padrão de mercado, com uma fração da complexidade de exactly-once.
- **Trade-off:** o cliente precisa deduplicar notificações repetidas.

### Decisão: Reuso dos padrões existentes ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md))

- **Justificativa:** menos código novo, previsibilidade para o time e aderência ao prazo.
- **Trade-off:** a feature herda as limitações atuais do projeto (por exemplo, ausência de stack de métricas).

---

## 10. Dependências

### Dependência organizacional: Revisão de segurança

A Engenharia de Segurança (Sofia) precisa de **pelo menos 2 dias úteis** para revisar a assinatura HMAC e a geração de secrets antes do deploy. A revisão deve ser agendada no fim da terceira sprint.

### Dependência organizacional: Documentação no portal do desenvolvedor

Produto (Marcos) documenta no portal como integrar via API, com destaque para a garantia at-least-once e a deduplicação por `X-Event-Id`. Os clientes dependem desse material para integrar.

### Dependência organizacional: Alinhamento de prazo com os clientes

Produto (Marcos) confirma o prazo de fim de novembro com a Atlas e atualiza os demais clientes solicitantes.

### Dependência técnica: Validação das definições de implementação

O FDD propõe definições para pontos que a reunião não decidiu (propostas P-01 a P-23, na [seção 14 do FDD](FDD.md#14-decisões-de-implementação-propostas--pendentes-de-validação)). Elas precisam ser aprovadas pelos revisores antes do início do desenvolvimento.

### Dependência técnica: Novo processo em produção

O deploy passa a incluir o processo de entrega, além da API, com a mesma configuração de banco. A operação precisa implantá-lo e monitorá-lo.

### Dependência externa: Preparação dos clientes

Cada cliente precisa expor um endpoint HTTPS, verificar a assinatura HMAC e deduplicar notificações pelo identificador de evento.

---

## 11. Riscos e mitigação

As probabilidades são **🔶 estimativas deste PRD**; a reunião não as quantificou.

### Perda do cliente Atlas por atraso na entrega

- **Probabilidade:** 🔶 média (há uma ameaça explícita de migração e o prazo é apertado: 3 sprints)
- **Impacto:** alto (perda de cliente B2B estratégico e receita associada)
- **Mitigação:**
  - Escopo enxuto, sem nova infraestrutura e com reuso dos padrões do projeto
  - Prazo de 3 sprints confirmado com o cliente
  - Itens não essenciais (e-mail, dashboard, rate limiting) fora desta fase
- **Plano de contingência:** comunicação antecipada ao cliente com uma nova data, caso a revisão de segurança ou as validações pendentes atrasem o deploy

### Vazamento de secret de um cliente

- **Probabilidade:** 🔶 média (já houve cliente que vazou secret em log)
- **Impacto:** alto para o cliente afetado (notificações falsificáveis); limitado aos demais clientes
- **Mitigação:**
  - Secret única por webhook
  - Rotação pela API com 24 h de convivência
  - Secret nunca exibida após o cadastro nem registrada em logs
- **Plano de contingência:** rotação imediata da secret do webhook afetado

### Indisponibilidade prolongada do endpoint do cliente

- **Probabilidade:** 🔶 média (já houve cliente com 2 h de manutenção planejada)
- **Impacto:** médio (notificações atrasadas; o cliente pode perder mudanças se não reagir)
- **Mitigação:**
  - Reentrega automática por ~15 h
  - Histórico de entregas consultável
- **Plano de contingência:** evento preservado na dead letter e reprocessado manualmente por um administrador

### Cliente processando notificações duplicadas

- **Probabilidade:** 🔶 média (duplicatas são esperadas no modelo at-least-once)
- **Impacto:** médio (ações repetidas no sistema do cliente)
- **Mitigação:**
  - Identificador único e estável por evento (`X-Event-Id`)
  - Documentação em destaque no portal do desenvolvedor
- **Plano de contingência:** suporte ao cliente com o histórico de entregas para identificar as duplicatas

### Excesso de notificações para um mesmo cliente

- **Probabilidade:** 🔶 baixa a média (muitos pedidos mudando de status em pouco tempo)
- **Impacto:** médio (sobrecarga no endpoint do cliente)
- **Mitigação:**
  - Filtro de eventos por webhook reduz o volume
  - Monitoramento do volume de entregas por cliente
- **Plano de contingência:** priorizar o rate limiting de saída, hoje adiado

### Processo de entrega parado sem que ninguém perceba

- **Probabilidade:** 🔶 baixa
- **Impacto:** alto (todos os clientes deixam de receber notificações)
- **Mitigação:**
  - Eventos persistidos; nada se perde enquanto o processo está parado
  - Alerta quando a notificação pendente mais antiga passar de 10 s (FDD)
- **Plano de contingência:** reiniciar o processo; os eventos acumulados são entregues na sequência

---

## 12. Critérios de aceitação

- Uma mudança de status assinada por um webhook gera uma notificação ao endpoint do cliente em **menos de 10 segundos**.
- Mudanças para status não assinados não geram notificação.
- Se o registro do evento falhar, a mudança de status não é efetivada.
- É possível cadastrar, editar, remover e listar webhooks de um cliente pela API autenticada.
- O cadastro devolve uma secret única; ela não aparece em nenhuma outra consulta nem nos logs.
- URLs sem HTTPS são recusadas.
- Toda notificação traz `X-Event-Id`, `X-Webhook-Id`, `X-Signature` e `X-Timestamp`, e a assinatura pode ser verificada com a secret do webhook.
- Após a rotação, a secret antiga continua válida por 24 horas e deixa de valer depois.
- Uma entrega que falha é reenviada nos intervalos de 1 min, 5 min, 30 min, 2 h e 12 h.
- Esgotadas as tentativas, o evento aparece na dead letter com payload, motivo e data/hora.
- Somente um administrador consegue reprocessar um evento da dead letter, e a ação fica registrada com o usuário que a executou.
- O histórico de entregas mostra sucesso ou falha, payload, resposta e tempo de resposta.
- Notificações acima de 64 KB não são enviadas.
- Os endpoints existentes da API continuam funcionando sem alteração de contrato.
- A revisão de segurança foi concluída antes do deploy.

---

## 13. Estratégia de testes e validação

**Tipos de teste obrigatórios**

- **Unitários:** assinatura HMAC e rotação de secret; cálculo dos intervalos de reentrega; filtro de eventos por status.
- **Integração:** cadastro e gestão de webhooks, registro do evento junto com a mudança de status (incluindo o rollback), histórico de entregas e reprocessamento da dead letter. Seguem o padrão de testes já usado no projeto (Vitest + Supertest com banco real).
- **Ponta a ponta:** mudança de status → processo de entrega → endpoint de teste, validando conteúdo, headers, assinatura e reentrega (previsto na estimativa da fase).
- **Segurança:** revisão dedicada da Engenharia de Segurança sobre assinatura, geração e armazenamento de secrets, e permissões do reprocessamento.

**Estratégia de validação**

- Medição do tempo entre a mudança de status e a entrega, contra a meta de 10 s.
- Simulação de indisponibilidade do endpoint para validar a reentrega e a dead letter.
- Validação da documentação do portal do desenvolvedor antes da liberação aos clientes.
- 🔶 Piloto com os clientes solicitantes antes da liberação geral, acompanhando a adoção (meta 3/3) e a redução do polling.

---

## 14. Questões em aberto

Pontos sem decisão, que não bloqueiam o escopo de produto mas precisam ser fechados antes da implementação:

- Se o cliente é informado no corpo ou no caminho das requisições de cadastro.
- Quais campos do pedido, além do valor total, entram na notificação.
- Se as 5 tentativas incluem o envio inicial.
- Limite do histórico de entregas (últimas 100 ou paginado).

Discussão completa na [RFC-001](RFC.md#questões-em-aberto) e propostas de definição na [seção 14 do FDD](FDD.md#14-decisões-de-implementação-propostas--pendentes-de-validação).
