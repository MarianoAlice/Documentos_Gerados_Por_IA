# ADR-005: Garantia At-Least-Once com X-Event-Id

## Status

Aceito — [09:26] Larissa

## Contexto

No padrão outbox com worker e retry, é possível que o cliente receba HTTP 200 mas a confirmação falhe antes do worker marcar o evento como entregue — ou que um retry ocorra após timeout. Garantir **exactly-once** exigiria coordenação bilateral complexa.

Clientes B2B precisam de um mecanismo simples para ignorar duplicatas sem perder eventos legítimos.

## Decisão

1. Semântica de entrega: **at-least-once** (o cliente pode receber o mesmo evento mais de uma vez).
2. Cada evento recebe **`event_id`** (UUID) gerado na inserção na `webhook_outbox`.
3. O worker envia o identificador no header **`X-Event-Id`** (também presente no corpo JSON do payload).
4. Responsabilidade de **deduplicação** fica com o cliente, usando `event_id` como chave idempotente.

Marcos documentará esse comportamento de forma destacada no portal de desenvolvedores.

## Alternativas Consideradas

### Exactly-once delivery

**Descartada.** Diego: exigiria coordenação dos dois lados; complexidade desproporcional. Stripe e GitHub usam at-least-once.

### Deduplicação apenas no corpo (sem header)

**Descartada em favor do header explícito.** Diego definiu `X-Event-Id` no header para facilitar inspeção e middlewares do lado do cliente.

## Consequências

### Positivas

- Modelo simples e amplamente adotado no mercado.
- Compatível com retry e timeout do worker sem lógica extra de confirmação bilateral.
- Cliente controla sua própria idempotência.

### Negativas

- Todo cliente deve implementar deduplicação — requisito de integração adicional.
- **Trade-off explícito:** simplicidade e confiabilidade de entrega em troca de delegar idempotência ao consumidor.
