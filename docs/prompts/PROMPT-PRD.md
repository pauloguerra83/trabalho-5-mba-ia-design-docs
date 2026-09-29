# Prompt — Geração do PRD do Sistema de Webhooks

## Como usar

| Item | Descrição |
| --- | --- |
| Objetivo | Gerar o PRD (Product Requirements Document) da feature de Webhooks de Notificação de Pedidos em `docs/PRD.md` |
| Quando usar | Para produzir o PRD pela primeira vez ou regenerá-lo depois de mudanças na ata, nas ADRs, na RFC ou no FDD |
| Pré-requisitos | Existirem a ata (`docs/levantamento-tecnico/ATA-REUNIAO-WEBHOOKS.md`), as ADRs, `docs/RFC.md`, `docs/FDD.md` e o exemplo do curso `docs/mba-templates-guia/PRD-Exemplo.md` |
| Execução | Abrir uma sessão de IA com acesso de leitura e escrita ao repositório e colar **todo o conteúdo abaixo da linha** |
| Resultado esperado | `docs/PRD.md` no formato do curso, em nível de produto e negócio |

---

# Contexto

Você vai produzir o **PRD** da feature **Sistema de Webhooks de Notificação de Pedidos** de um Order Management System. O PRD responde **"por que e o quê"**: problema, público, escopo, requisitos e métricas de sucesso. Como é produzido depois da RFC, das ADRs e do FDD, ele é essencialmente uma **consolidação em linguagem de produto**. Detalhes de implementação (caminhos de API, payloads, tabelas) ficam no FDD e devem ser apenas referenciados.

Leia as fontes antes de escrever. Não assuma nada que não esteja nelas.

# Fontes do conhecimento (em ordem de prioridade)

1. `docs/levantamento-tecnico/ATA-REUNIAO-WEBHOOKS.md`: contexto de negócio (CTX), requisitos funcionais (RF), não funcionais (RNF), fora de escopo (FE), evoluções (EV), limitações (LIM), riscos (RSK), dependências, prazo e itens de ação.
2. `docs/adrs/`: decisões e trade-offs.
3. `docs/RFC.md` e `docs/FDD.md`: visão técnica, questões em aberto e propostas pendentes (seção 14 do FDD).
4. `docs/mba-templates-guia/PRD-Exemplo.md`: **formato a seguir**.
5. `TRANSCRICAO.md`: apenas para validar algo que não esteja claro na ata.

# Saída

- Arquivo: `docs/PRD.md` (sobrescrever o conteúdo existente).
- Idioma: português.
- Metadados em tabela:
  - Versão: v1.
  - Data.
  - **Responsável: Marcos (Product Manager).**
  - Revisores: Larissa, Bruno, Diego, Sofia, com papéis.
  - Status: Em revisão.
  - Links: RFC, FDD, ADRs, ata.

# Estrutura obrigatória (formato do PRD-Exemplo do curso)

1. **Resumo e contexto da feature:** o que é e onde será implantada.
2. **Problema e motivação:** problemas priorizados e por que resolver agora.
3. **Público-alvo e cenários de uso:** tabela de públicos com necessidades e cenários de uso numerados.
4. **Objetivos e métricas de sucesso:** tabela Objetivo | Métrica | Meta | Origem ("Decidido" ou 🔶). Pelo menos 1 meta quantitativa (ex.: < 10 s, 100% dos eventos, ~15 h de reentrega, prazo).
5. **Escopo:**
   - **Incluso**;
   - **Fora de escopo** em tabela Item | Situação, marcando cada item como **descartado** ou **adiado** conforme a reunião (mínimo 2);
   - **Evoluções futuras**.
6. **Requisitos funcionais:** no formato do exemplo:

   ```
   ### RF-NNN Nome
   Descrição
   **Fluxo principal** …
   **Fluxos alternativos e exceções** …
   **Erros previstos** …
   **Prioridade:** …
   ```

   Mínimo de **8 RFs discutidos na reunião**. Esperados:
   - notificação de mudança de status;
   - cadastro;
   - edição;
   - remoção;
   - listagem;
   - filtro de eventos;
   - histórico de entregas;
   - rotação de secret;
   - reentrega;
   - dead letter;
   - replay ADMIN com auditoria;
   - HTTPS;
   - identificação e autenticidade da notificação.

   Como a reunião não priorizou os RFs entre si, use "Alta — escopo acordado da fase 1".
7. **Requisitos não funcionais:** por categoria (performance e latência, confiabilidade, segurança, auditoria, observabilidade, compatibilidade, manutenibilidade).
8. **Arquitetura e abordagem:** curta, com link para a RFC.
9. **Decisões e trade-offs principais:** um bloco por ADR, com **Justificativa** e **Trade-off** e link para a ADR.
10. **Dependências:** blocos por tipo:
    - organizacional (revisão de segurança, portal do desenvolvedor, prazo com clientes);
    - técnica (validação das propostas do FDD, novo processo em produção);
    - externa (preparação dos clientes).
11. **Riscos e mitigação:** blocos com **Probabilidade**, **Impacto**, **Mitigação** e **Plano de contingência**. Mínimo de 2; esperados cerca de 6, a partir dos RSK da ata.
12. **Critérios de aceitação:** checklist verificável, em linguagem de produto.
13. **Estratégia de testes e validação:** tipos de teste obrigatórios e estratégia de validação.
14. **Questões em aberto:** curta, com links para a RFC e para a seção 14 do FDD.

# Regras de conteúdo

1. **Não invente requisitos, decisões ou restrições.** Tudo deve ter base na ata, nas ADRs, na RFC ou no FDD.
2. **Estimar e marcar.** A reunião não estimou probabilidades de risco nem definiu metas além das citadas. Quando o formato exigir, estime de forma fundamentada em fatos da reunião (ex.: "já houve cliente que vazou secret") e marque com **🔶** como estimativa do PRD a validar. Propostas do FDD usadas no PRD também recebem 🔶 e o ID (P-xx). Explique a convenção 🔶 no topo do documento.
3. **Nível de produto.** Não descreva tabelas, payload campo a campo, códigos HTTP ou caminhos de endpoint; remeta ao FDD. Nomes de headers podem aparecer quando explicam uma capacidade para o cliente.
4. **Sem rastreio de fala.** Não cite horários nem atribua falas (`[hh:mm] Nome`). Nomes só nos metadados e como responsáveis de dependências.
5. **Fora de escopo** nunca aparece como requisito.
6. **Links válidos** para `RFC.md`, `FDD.md`, `adrs/` e `levantamento-tecnico/`.

# Checklist de aceite

- [ ] `docs/PRD.md` existe e está em Markdown.
- [ ] Contém as seções: Resumo e contexto da feature; Problema e motivação; Público-alvo e cenários de uso; Objetivos e métricas de sucesso; Escopo (incluso e fora de escopo); Requisitos funcionais; Requisitos não funcionais; Decisões e trade-offs principais; Dependências; Riscos e mitigação; Critérios de aceitação; Estratégia de testes e validação.
- [ ] Identifica no mínimo 8 requisitos funcionais discutidos na reunião.
- [ ] Inclui pelo menos 1 objetivo com métrica e meta quantitativa.
- [ ] "Fora de escopo" lista pelo menos 2 itens explicitamente descartados ou adiados na reunião.
- [ ] "Riscos" inclui pelo menos 2 riscos com probabilidade, impacto e mitigação.
- [ ] Toda estimativa ou proposta não decidida na reunião está marcada com 🔶.
- [ ] Nenhum `[hh:mm]` no documento.

# Verificação obrigatória antes de concluir

Execute e reporte o resultado:

1. Buscar os 12 títulos de seção obrigatórios.
2. Contar `^### RF-` (≥ 8).
3. Confirmar pelo menos 1 meta numérica na tabela de objetivos.
4. Contar os itens de "Fora de escopo" marcados como descartado/adiado (≥ 2).
5. Contar, na seção de riscos, as linhas `**Probabilidade:**`, `**Impacto:**` e `**Mitigação:**` (a mesma quantidade, ≥ 2).
6. Buscar `\[[0-9]{2}:[0-9]{2}\]` e confirmar zero ocorrências.
7. Confirmar que todos os links relativos apontam para arquivos existentes.

Se o shell não estiver disponível, faça as mesmas verificações com ferramentas de busca de arquivos e de conteúdo.

# Formato da resposta final

1. Link para `docs/PRD.md` e resumo de uma linha por seção.
2. Resultado de cada verificação acima.
3. Lista das estimativas e propostas 🔶 que precisam de validação.
