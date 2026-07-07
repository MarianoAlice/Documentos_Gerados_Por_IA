# ADR-001: Padrão Outbox no MySQL

## Status

Aceito — [09:08] Larissa

## Contexto

Clientes B2B precisam ser notificados quando o status de um pedido muda. A alteração de status hoje ocorre em transação Prisma que atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity` dos produtos (`src/modules/orders/order.service.ts` → `changeStatus()`).

Disparar a chamada HTTP do webhook de forma síncrona dentro dessa transação foi descartado na reunião: um endpoint lento do cliente bloquearia mudanças de status para outros pedidos, e indisponibilidade do cliente não pode causar rollback da mudança de status — o negócio já foi persistido.

Alternativas de infraestrutura externa (Redis Streams, filas dedicadas) foram consideradas, mas o time é pequeno e o MySQL já está em uso.

## Decisão

Adotar o **padrão Transactional Outbox** usando o MySQL existente:

1. Na mesma transação SQL que persiste a mudança de status, inserir um registro na tabela `webhook_outbox`.
2. Um worker separado lê a outbox e executa as chamadas HTTP de forma assíncrona.
3. Se a transação principal der rollback, o evento na outbox desaparece junto — não há inconsistência entre status persistido e evento registrado.

A integração no código existente será via função `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada dentro de `changeStatus()`, recebendo o client de transação Prisma (`tx`) — sem injetar o repository de webhooks inteiro no `OrderService`.

## Alternativas Consideradas

### Disparo síncrono no `OrderService`

**Descartada.** Bruno e Larissa argumentaram que qualquer latência ou indisponibilidade do cliente externo impactaria a transação de pedidos. Rollback por falha de HTTP é inaceitável para o domínio de pedidos.

### Redis Streams / fila externa

**Descartada.** Diego classificou como overengineering para o tamanho do time. Exigiria subir e operar infraestrutura adicional sem benefício proporcional nesta fase.

## Consequências

### Positivas

- Garantia de consistência entre mudança de status e registro do evento (atomicidade transacional).
- Reuso do MySQL e Prisma já presentes no projeto.
- Desacoplamento entre persistência de negócio e entrega HTTP.

### Negativas

- Latência adicional entre commit da transação e entrega ao cliente (mitigada pelo worker em polling de 2 s — ver [ADR-002](ADR-002-worker-polling-processo-separado.md)).
- Crescimento da tabela `webhook_outbox` exige estratégia futura de arquivamento (explicitamente fora do escopo desta fase).
- **Trade-off explícito:** consistência forte e simplicidade operacional em troca de entrega eventual (segundos, não milissegundos).
