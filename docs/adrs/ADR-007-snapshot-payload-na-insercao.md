# ADR-007: Snapshot do Payload na Inserção da Outbox

## Status

Aceito — [09:52] Larissa / Bruno

## Contexto

Ao modelar a tabela `webhook_outbox`, surgiu a dúvida se o registro deve armazenar o payload JSON completo no momento da inserção ou apenas referências (`order_id`) para renderizar no envio.

Pedidos podem ter campos alterados após a mudança de status. O evento de webhook representa o **estado no instante da transição**, não o estado atual do pedido.

## Decisão

Armazenar o **payload já renderizado (snapshot)** na inserção na `webhook_outbox`, dentro da mesma transação de `changeStatus()`.

O worker envia o JSON armazenado sem reconsultar ou remontar o payload a partir do pedido atual.

Campos do snapshot (definidos na reunião): `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`. **Itens do pedido não são incluídos** — cliente consulta `GET /orders/:id` se precisar de detalhes.

Limite de tamanho: **64 KB**; se ultrapassar, erro na inserção (não truncar).

## Alternativas Consideradas

### Renderizar payload no momento do envio (somente `order_id` na outbox)

**Descartada.** Larissa e Diego: se o pedido mudar depois, o evento refletiria estado incorreto — caso esquisito para auditoria e para o cliente.

### Incluir itens do pedido no payload

**Descartada.** Diego: inflaria payload sem necessidade; cliente pode buscar detalhes via API existente.

## Consequências

### Positivas

- Evento imutável e auditável — reflete exatamente o momento da transição.
- Worker mais simples: lê e envia, sem join adicional no envio.
- DLQ e histórico de entregas preservam o que foi (ou seria) enviado.

### Negativas

- Maior uso de armazenamento por linha na outbox (aceitável dado payload enxuto).
- Alterações futuras no formato do evento não retroagem a eventos já enfileirados.
- **Trade-off explícito:** fidelidade histórica e simplicidade do worker em troca de duplicar dados do pedido no registro do evento.
