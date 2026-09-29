# ADR-004 — Autenticação das entregas com HMAC-SHA256 e secret por endpoint

| Campo | Valor |
| --- | --- |
| Status | Aceito |
| Data | Reunião técnica de quinta-feira, 09:00 (data do calendário não registrada) |
| Decisores | Segurança (Sofia), Tech Lead (Larissa), Engenharia de Plataforma (Diego), Engenharia de Pedidos (Bruno) |
| Relacionados | [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md) |

## Status

**Aceito** na reunião técnica da feature e confirmado no resumo de encerramento.

## Contexto

Os webhooks enviam dados de pedidos para endpoints **fora da infraestrutura** da plataforma. O cliente precisa conseguir verificar que a requisição veio realmente da plataforma e que o payload não foi adulterado no caminho.

Restrições:

- A solução precisa ser **simples de verificar pelo cliente**, com bibliotecas amplamente disponíveis.
- **Vazamento de secret é um risco real:** já houve cliente que vazou secret em log da própria aplicação.
- A troca de secret não pode quebrar a integração do cliente de uma hora para outra.

## Decisão

1. Cada entrega é **assinada com HMAC-SHA256 sobre o corpo do request**, e a assinatura vai no header **`X-Signature`**. O cliente verifica do lado dele.
2. **Cada endpoint de webhook tem uma secret única**, nunca uma secret global da plataforma. A secret é **gerada pela plataforma e devolvida na criação** do webhook.
3. A secret é **rotacionável pela API**: ao rotacionar, a antiga continua válida **por 24 horas em paralelo** e depois é invalidada.
4. **TLS obrigatório:** a URL do webhook precisa ser `https`; `http` é recusado com erro de validação no schema Zod, seguindo o padrão dos `*.schemas.ts` do projeto.
5. O header **`X-Timestamp`** leva o momento do envio, para o cliente poder detectar replay attack se quiser.

> **Justificativa principal:** HMAC-SHA256 é o padrão de mercado e qualquer cliente tem biblioteca para verificá-lo; a secret por endpoint limita o impacto de um vazamento a um único cadastro, e a rotação com convivência de 24 h permite reagir a vazamentos sem interromper a integração.

## Alternativas Consideradas

### A1. Secret global da plataforma

- **Prós:** gestão de uma única chave; nada a armazenar por cadastro.
- **Contras:** se uma vaza, todas vazam: um vazamento compromete todos os clientes.
- **Descarte:** secret única por endpoint.

### A2. Secret fixa, sem rotação

- **Prós:** sem estado de convivência entre duas secrets; menos endpoints.
- **Contras:** um cliente que vaze a secret, como já aconteceu, não teria como substituí-la sem recadastrar o webhook.
- **Descarte:** rotação pela API com grace period de 24 h, para dar tempo ao cliente de migrar os sistemas dele.

## Consequências

**Positivas**

- O cliente consegue validar **origem e integridade** de cada entrega com bibliotecas padrão.
- Um vazamento fica **restrito a um endpoint** e pode ser corrigido por rotação.
- A exigência de HTTPS protege o conteúdo em trânsito, com custo de implementação mínimo (validação de schema).
- `X-Timestamp` dá ao cliente um meio de rejeitar requisições antigas reenviadas por terceiros.

**Negativas e trade-offs**

- A plataforma passa a **guardar uma secret por endpoint** e a gerenciar, por até 24 h, duas secrets válidas no mesmo cadastro.
- A verificação da assinatura e a proteção contra replay ficam **sob responsabilidade do cliente**.
- Clientes que só aceitam `http` não podem se integrar.

**Riscos e impactos operacionais**

- Geração e armazenamento de secret são pontos sensíveis: a Engenharia de Segurança exige **pelo menos 2 dias úteis de revisão** do código de HMAC e de geração de secret antes do deploy.
- Secrets não podem aparecer em logs. O logger do projeto já aplica redaction de campos sensíveis ([src/shared/logger/index.ts](../../src/shared/logger/index.ts)), e a lista precisará cobrir a secret.

**Pontos em aberto** (da ata, sem decisão)

- Como a assinatura se comporta durante as 24 h de convivência das duas secrets (LAC-03).
- Formato da assinatura (hex/base64, prefixo) e se o `X-Timestamp` entra no cálculo do HMAC; a reunião definiu apenas "sobre o corpo do request" (LAC-04).
- Forma de armazenamento da secret e se ela volta a ser exibida após a criação (LAC-05).
- Contrato do endpoint de rotação (LAC-01).

## Referências

- Código: [src/modules/customers/customer.schemas.ts](../../src/modules/customers/customer.schemas.ts) (exemplo do padrão de schemas Zod), [src/middlewares/validate.middleware.ts](../../src/middlewares/validate.middleware.ts) (validação e erro `VALIDATION_ERROR`), [src/shared/logger/index.ts](../../src/shared/logger/index.ts) (redaction).
- Detalhamento e rastreabilidade: [ata da reunião](../levantamento-tecnico/ATA-REUNIAO-WEBHOOKS.md) — DT-04, DT-16; RF-08, RF-14; RNF-10 a RNF-13, RNF-17; ALT-08.
