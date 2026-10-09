# Prompt — Geração do Tracker de Rastreabilidade

## Como usar

| Item | Descrição |
| --- | --- |
| Objetivo | Gerar `docs/TRACKER.md`, tabela que liga cada item de PRD, RFC, FDD e ADRs à origem na transcrição ou no código |
| Quando usar | Depois de produzir ou alterar qualquer um dos documentos do pacote |
| Pré-requisitos | `TRANSCRICAO.md`, a ata (`docs/levantamento-tecnico/ATA-REUNIAO-WEBHOOKS.md`), `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` e `docs/adrs/ADR-*.md` |
| Execução | Abrir uma sessão de IA com acesso de leitura e escrita ao repositório (e, de preferência, a um shell) e colar **todo o conteúdo abaixo da linha** |
| Resultado esperado | `docs/TRACKER.md` com cobertura ≥ 80%, ≥ 70% de linhas com timestamp válido e ≥ 5 linhas de código real |

---

# Contexto

Os documentos do pacote (PRD, RFC, FDD e ADRs) foram escritos **sem** citações de horário. O Tracker é o lugar onde o rastreio fala a fala fica registrado: cada decisão, requisito, restrição, alternativa, risco etc. precisa apontar para **de onde veio**, seja uma fala da reunião (`TRANSCRICAO.md`) ou um arquivo real do código. É a principal defesa do pacote contra alucinações da IA.

# Fontes

1. `docs/levantamento-tecnico/ATA-REUNIAO-WEBHOOKS.md`: todos os itens da reunião já com `[hh:mm] Nome` validados. Use como índice de timestamps.
2. `TRANSCRICAO.md`: fonte final para confirmar cada timestamp.
3. `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` e `docs/adrs/ADR-*.md`: os itens a rastrear.
4. Código em `src/`, `prisma/`, `tests/`, `package.json` e `vitest.config.ts`: para itens com origem no código.

# Formato obrigatório

Arquivo `docs/TRACKER.md` com:

1. Cabeçalho curto e legenda (Fonte, Tipos e convenção 🔶).
2. Seção **Metodologia e cobertura**:
   - definição de "item identificável";
   - lista dos itens derivados excluídos do denominador e o motivo;
   - tabela Documento | Itens identificáveis | Linhas | Cobertura | Derivados | Cobertura incluindo derivados.
3. **Uma única tabela** com exatamente este cabeçalho:

```
| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
```

4. Tabela final de **resumo por fonte** (linhas e percentual).

Regras das colunas:

- **ID:** único. Convenção:
  - ADRs: `ADR-00N`, `ADR-00N-CTX-xx`, `-DEC-xx`, `-ALT-xx`, `-CONS-xx`, `-QA-xx`;
  - RFC: `RFC-CTX`, `-OBJ`, `-NOBJ`, `-COMP`, `-GAR`, `-ALT`, `-QA`, `-IMP`, `-RISK`, `-OOS`, `-NEXT`;
  - FDD: `FDD-OBJ`, `-ESC`, `-EXC`, `-DADOS`, `-FLUXO`, `-CONTRATO`, `-ERRO`, `-RES`, `-OBS`, `-INT`, `-ENV`, `-DEP`, `-RISK`, `-PROP-01…23`;
  - PRD: `PRD-CTX`, `-PROB`, `-PUB`, `-OBJ`, `-INC`, `-OOS`, `-EVO`, `-FR-01…`, `-RES`, `-NFR`, `-ARQ`, `-DEC`, `-DEP`, `-RISK`, `-QA`.
- **Documento:** caminho do arquivo onde o item aparece (`docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/adrs/ADR-00N-...md`).
- **Tipo:** Contexto, Restrição, Decisão, Alternativa descartada, Trade-off, Limitação, Risco, Objetivo/Métrica, Escopo, Fora de escopo, Evolução futura, Requisito Funcional, Requisito Não Funcional, Contrato, Erro, Integração, Dependência, Questão em aberto, Lacuna, Proposta (pendente de validação), Estimativa (pendente de validação).
- **Conteúdo (resumo):** uma linha.
- **Fonte:** `TRANSCRICAO` ou `CODIGO` (somente esses dois valores).
- **Localização:**
  - para `TRANSCRICAO`: `[hh:mm] Nome` (até três, separados por `;`);
  - para `CODIGO`: caminho real do arquivo (ex.: `src/modules/orders/order.service.ts`).

# Regras de conteúdo

1. **Cubra todos os itens identificáveis:** requisitos, decisões, restrições, alternativas, trade-offs e limitações, riscos, objetivos, escopo e fora de escopo, evoluções, questões em aberto, contratos, erros, integrações, dependências, propostas e estimativas. Critérios de aceite, testes, cenários narrativos, TL;DR e consequências positivas são derivados: declare-os fora do denominador e mostre também a cobertura incluindo-os.
2. **Nunca invente timestamps.** Todo `[hh:mm] Nome` deve existir literalmente em `TRANSCRICAO.md` (a linha começa com `[hh:mm] Nome:`).
3. **Propostas e estimativas 🔶** (FDD P-01 a P-23, estimativas de risco e metas do PRD) não foram decididas na reunião. Aponte a Localização para **onde o tema foi levantado** ou para o fato que sustenta a estimativa. Se a proposta nasce do código, aponte para o arquivo. O resumo deve deixar claro que é proposta ou estimativa.
4. **Itens do código:** use `CODIGO` quando a origem for o comportamento do código (ex.: pontos de integração do FDD, convenções de envelope, `NotFoundError` fixando `NOT_FOUND`, `validate` convertendo Zod em `VALIDATION_ERROR`). Mínimo de 5 linhas.
5. **Proporção:** pelo menos 70% das linhas com `TRANSCRICAO`.
6. Ordene por documento: ADRs → RFC → FDD → PRD.

# Checklist de aceite

- [ ] `docs/TRACKER.md` existe e segue o formato de tabela `| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |`.
- [ ] Pelo menos 80% dos itens identificáveis dos documentos têm linha correspondente.
- [ ] Pelo menos 70% das linhas têm Fonte = `TRANSCRICAO` com timestamp válido no formato `[hh:mm] Nome`.
- [ ] Pelo menos 5 linhas têm Fonte = `CODIGO` com caminho de arquivo real.
- [ ] IDs únicos; a coluna Documento aponta para arquivos existentes.
- [ ] As tabelas de cobertura e de resumo por fonte batem com as contagens reais.

# Verificação obrigatória antes de concluir

Execute (ou equivalente com ferramentas de busca, se não houver shell):

```bash
T=docs/TRACKER.md
rows=$(grep -E '^\| (ADR|RFC|FDD|PRD)-' $T)
echo "total: $(echo "$rows" | wc -l)"
for p in ADR RFC FDD PRD; do echo "$p: $(echo "$rows" | grep -c "^| $p-")"; done
echo "TRANSCRICAO: $(echo "$rows" | grep -c '| TRANSCRICAO |')"
echo "CODIGO: $(echo "$rows" | grep -c '| CODIGO |')"
# IDs duplicados
echo "$rows" | awk -F'|' '{gsub(/ /,"",$2); print $2}' | sort | uniq -d
# timestamps inexistentes na transcrição
grep -oE '\[[0-9]{2}:[0-9]{2}\] [A-Z][a-z]+' $T | sort -u | while read -r c; do grep -qF "$c:" TRANSCRICAO.md || echo "MISSING $c"; done
# caminhos de código inexistentes
echo "$rows" | grep '| CODIGO |' | awk -F'|' '{gsub(/^ +| +$/,"",$7); print $7}' | sort -u | while read -r f; do [ -f "$f" ] || echo "MISSING $f"; done
# documentos inexistentes
echo "$rows" | awk -F'|' '{gsub(/^ +| +$/,"",$3); print $3}' | sort -u | while read -r f; do [ -f "$f" ] || echo "MISSING $f"; done
```

Corrija qualquer falha e atualize as tabelas de cobertura e de resumo com os números reais.

# Formato da resposta final

1. Link para `docs/TRACKER.md`.
2. Contagem de linhas por documento e por fonte, com os percentuais.
3. Resultado das verificações (duplicados, timestamps, caminhos, documentos).
