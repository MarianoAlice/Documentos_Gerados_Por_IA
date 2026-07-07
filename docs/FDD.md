# FDD — Webhooks de Notificação de Pedidos

**Feature:** Sistema de Webhooks Outbound  
**Versão:** 1.1  
**Data:** 07/07/2026  
**RFC:** [RFC.md](RFC.md)  
**ADRs:** [adrs/](adrs/) (ADR-001 a ADR-007)

Este documento detalha **como implementar** a proposta do RFC. Decisões arquiteturais estão registradas nos ADRs; aqui ficam fluxos, contratos, erros e pontos de alteração no código existente.

---

## Contexto e motivação técnica

O OMS persiste mudanças de status em transação Prisma via `OrderService.changeStatus()` (`src/modules/orders/order.service.ts`). Hoje a transação:

1. Valida transição (`canTransition` em `src/modules/orders/order.status.ts`)
2. Ajusta estoque (`shouldDebitStock` / `shouldReplenishStock`)
3. Atualiza `orders.status`
4. Insere em `order_status_history`

Não existe camada de eventos ou notificações externas. Esta feature adiciona, sem alterar o contrato de `/orders`:

| Componente (**a criar**) | ADR |
|--------------------------|-----|
| Tabelas Prisma (config, outbox, DLQ, deliveries) — alterar `prisma/schema.prisma` | ADR-001, ADR-003, ADR-007 |
| Módulo `src/modules/webhooks/` (pasta e arquivos novos) | ADR-006 |
| Worker `src/worker.ts` (arquivo novo) | ADR-002 |
| Hook transacional em `changeStatus()` — **alterar** `src/modules/orders/order.service.ts` | ADR-001, ADR-007 |
| HMAC-SHA256 e headers outbound | ADR-004, ADR-005 |

---

## Objetivos técnicos

1. Enfileirar evento na **mesma `$transaction`** que persiste mudança de status ([ADR-001](adrs/ADR-001-outbox-no-mysql.md))
2. Entregar via worker em **< 10 s** — polling 2 s + processamento ([ADR-002](adrs/ADR-002-worker-polling-processo-separado.md))
3. Assinatura **HMAC-SHA256** verificável pelo cliente ([ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md))
4. **Retry**, DLQ e replay admin sem perder auditoria ([ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md))
5. **At-least-once** com `X-Event-Id` ([ADR-005](adrs/ADR-005-at-least-once-x-event-id.md))
6. Reutilizar `AppError`, Zod, Pino, `requireRole` ([ADR-006](adrs/ADR-006-reuso-padroes-modulo-webhooks.md))
7. **Snapshot** do payload na inserção ([ADR-007](adrs/ADR-007-snapshot-payload-na-insercao.md))

---

## Escopo e exclusões

### Incluso

- Modelagem Prisma (4 entidades)
- CRUD de webhooks + rotação de secret + histórico de entregas + replay DLQ
- Worker com polling, HMAC, retry
- Testes de integração em `tests/webhooks.test.ts` (**a criar**)
- Script `npm run worker` (**a adicionar** em `package.json`)

### Exclusões (conforme RFC)

E-mail de alerta, dashboard visual, rate limiting, arquivamento 30 dias, múltiplos workers, webhooks inbound, Redis/filas externas, disparo síncrono no `OrderService`.

---

## Fluxos detalhados

### Fluxo 1 — Cadastro de webhook

```
Cliente                    API (webhook.routes — a criar)    DB
  │ POST /webhooks           │                                │
  │─────────────────────────►│ authenticate                   │
  │                          │ validate() — Zod HTTPS         │
  │                          │ WebhookService.create()        │
  │                          │───────────────────────────────►│ INSERT webhook_endpoints
  │◄─────────────────────────│                                │
  │ 201 + secret (única vez) │                                │
```

Filtro de status (`subscribedStatuses`) avaliado **na inserção da outbox**, não no envio ([09:34] Bruno).

### Fluxo 2 — Criação de evento na outbox (transacional)

Ponto de integração crítico: `OrderService.changeStatus()` — ver seção [Integração com o sistema existente](#integração-com-o-sistema-existente).

```
OrderService.changeStatus()  —  prisma.$transaction(async (tx) => { ... })
  │
  ├─► findUnique order + items
  ├─► canTransition(from, to)          ← src/modules/orders/order.status.ts
  ├─► debitStock / replenishStock      ← se aplicável
  ├─► tx.order.update({ status: to })
  ├─► tx.orderStatusHistory.create(...)
  ├─► publishWebhookEvent(tx, order, from, to)   ← NOVO (antes do return)
  │     ├─► SELECT endpoints WHERE customer_id AND active
  │     ├─► FILTER to IN subscribedStatuses
  │     ├─► IF nenhum match → return (sem INSERT)
  │     └─► FOR EACH match: INSERT webhook_outbox (snapshot JSON, status=pending)
  └─► COMMIT  (ou ROLLBACK se qualquer passo falhar)
```

Função **a criar** em `src/modules/webhooks/webhook.publisher.ts`:

```typescript
export async function publishWebhookEvent(
  tx: Prisma.TransactionClient,
  order: Order,
  fromStatus: OrderStatus,
  toStatus: OrderStatus,
): Promise<void>
```

### Fluxo 3 — Processamento pelo worker

Entry point **a criar:** `src/worker.ts` → `WebhookProcessor` **a criar** em `src/modules/webhooks/webhook.processor.ts`.

```
Loop a cada 2s ([ADR-002]):
  1. SELECT pending FROM webhook_outbox
       WHERE status='pending' AND (next_retry_at IS NULL OR next_retry_at <= NOW())
       ORDER BY created_at ASC LIMIT 50
  2. Para cada evento: UPDATE status='processing'
  3. Carregar WebhookEndpoint (url, secret, previousSecret se em grace period)
  4. Ler payload snapshot da outbox (sem reconsultar order — [ADR-007])
  5. Assinar corpo com HMAC-SHA256 ([ADR-004])
  6. POST url — timeout 10s
  7. Sucesso (2xx):
       - UPDATE outbox status='delivered'
       - INSERT webhook_deliveries (success, httpStatus, durationMs)
  8. Falha:
       - INSERT webhook_deliveries (failure, ...)
       - attempt_count++
       - IF attempt_count < 5: status='pending', next_retry_at = backoff
       - ELSE: INSERT webhook_dead_letter, status='failed' ([ADR-003])
```

### Fluxo 4 — Retry e DLQ

| Tentativa | Intervalo antes da próxima |
|-----------|---------------------------|
| 1ª entrega | Imediata |
| 2ª (retry 1) | 1 min |
| 3ª (retry 2) | 5 min |
| 4ª (retry 3) | 30 min |
| 5ª (retry 4) | 2 h |
| 6ª (retry 5) | 12 h → DLQ se falhar |

Total: **5 retries** após a falha inicial ([ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md)).

### Fluxo 5 — Replay DLQ (ADMIN)

```
POST /admin/webhooks/dead-letter/:id/replay
  → authenticate + requireRole('ADMIN')
  → WebhookService.replayFromDlq(dlqId, userId)
  → INSERT webhook_outbox (payload da DLQ, status=pending, novo event_id)
  → Registrar replayed_by + replayed_at na DLQ ou tabela de auditoria
  → 202 Accepted
```

---

## Contratos públicos

**Base URL:** `/api/v1`  
**Auth inbound:** `Authorization: Bearer <jwt>`  
**Paginação:** helper `paginated()` de `src/shared/http/response.ts` (padrão existente)

### POST /webhooks — Criar webhook

| | |
|---|---|
| **Auth** | `authenticate` — qualquer role autenticada |
| **Status** | `201 Created` |

**Request:**
```json
{
  "customerId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "url": "https://cliente.example.com/webhooks/orders",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"]
}
```

**Response 201:**
```json
{
  "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "customerId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "url": "https://cliente.example.com/webhooks/orders",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_8f3a2b1c9d0e7f6a5b4c3d2e1f0a9b8c",
  "createdAt": "2026-07-07T12:00:00.000Z"
}
```

**Erros:** `400 WEBHOOK_INVALID_URL`, `404 WEBHOOK_CUSTOMER_NOT_FOUND`, `401 UNAUTHORIZED`

---

### PATCH /webhooks/:id — Atualizar webhook

| | |
|---|---|
| **Auth** | `authenticate` |
| **Status** | `200 OK` |

**Request:**
```json
{
  "url": "https://cliente.example.com/v2/webhooks",
  "subscribedStatuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true
}
```

**Response 200:** objeto webhook (sem `secret`)

**Erros:** `404 WEBHOOK_NOT_FOUND`, `400 WEBHOOK_INVALID_URL`

---

### GET /webhooks — Listar por customer

| | |
|---|---|
| **Query** | `customerId` (obrigatório), `page`, `pageSize` |
| **Status** | `200 OK` |

**Response 200:**
```json
{
  "data": [
    {
      "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "customerId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "url": "https://cliente.example.com/webhooks/orders",
      "subscribedStatuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-07-07T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

---

### DELETE /webhooks/:id — Remover webhook

| | |
|---|---|
| **Auth** | `authenticate` |
| **Status** | `204 No Content` |

**Erros:** `404 WEBHOOK_NOT_FOUND`

---

### GET /webhooks/:id/deliveries — Histórico de entregas

| | |
|---|---|
| **Auth** | `authenticate` |
| **Status** | `200 OK` |
| **Limite** | ~100 últimas entregas |

**Response 200:**
```json
{
  "data": [
    {
      "id": "c9bf9e57-1685-4c89-bafb-ff5af830e8bf",
      "eventId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "status": "success",
      "httpStatus": 200,
      "responseBody": "{\"received\":true}",
      "durationMs": 245,
      "attemptNumber": 1,
      "createdAt": "2026-07-07T12:05:02.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 1, "totalPages": 1 }
}
```

---

### POST /webhooks/:id/rotate-secret — Rotacionar secret

| | |
|---|---|
| **Auth** | `authenticate` |
| **Status** | `200 OK` |

**Response 200:**
```json
{
  "secret": "whsec_nova_secret_aqui",
  "previousSecretExpiresAt": "2026-07-08T12:00:00.000Z"
}
```

---

### POST /admin/webhooks/dead-letter/:id/replay — Replay DLQ

| | |
|---|---|
| **Auth** | `authenticate` + `requireRole('ADMIN')` |
| **Status** | `202 Accepted` |

**Response 202:**
```json
{
  "outboxId": "new-outbox-uuid",
  "replayedBy": "admin-user-uuid",
  "replayedAt": "2026-07-07T14:00:00.000Z"
}
```

**Erros:** `403 FORBIDDEN`, `404 WEBHOOK_DLQ_NOT_FOUND`, `409 WEBHOOK_ALREADY_REPLAYED`

---

### Outbound — POST do worker para endpoint do cliente

Sem autenticação JWT; segurança via HMAC ([ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)).

**Headers:**
```
Content-Type: application/json
X-Event-Id: 3fa85f64-5717-4562-b3fc-2c963f66afa6
X-Signature: sha256=a1b2c3d4e5f6...
X-Timestamp: 2026-07-07T12:05:01.234Z
X-Webhook-Id: f47ac10b-58cc-4372-a567-0e02b2c3d479
```

**Body (snapshot — [ADR-007](adrs/ADR-007-snapshot-payload-na-insercao.md)):**
```json
{
  "event_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-07T12:05:00.000Z",
  "order_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "order_number": "ORD-000042",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "total_cents": 15990
}
```

**Verificação HMAC (lado cliente):**
```
expected = HMAC-SHA256(secret, raw_request_body)
timingSafeEqual(X-Signature, "sha256=" + hex(expected))
```

Cliente deduplica por `event_id` / `X-Event-Id` ([ADR-005](adrs/ADR-005-at-least-once-x-event-id.md)).

---

## Matriz de erros previstos

| Código | HTTP | Classe sugerida (**a criar**) | Quando |
|--------|------|-------------------------------|--------|
| `WEBHOOK_NOT_FOUND` | 404 | `WebhookNotFoundError` | Webhook id inexistente |
| `WEBHOOK_INVALID_URL` | 400 | `WebhookInvalidUrlError` | URL não HTTPS ou malformada (Zod) |
| `WEBHOOK_SECRET_REQUIRED` | 400 | `WebhookSecretRequiredError` | Operação exige secret ausente |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | `WebhookPayloadTooLargeError` | Snapshot > 64 KB na inserção |
| `WEBHOOK_DLQ_NOT_FOUND` | 404 | `WebhookDlqNotFoundError` | ID da DLQ não existe |
| `WEBHOOK_ALREADY_REPLAYED` | 409 | `WebhookAlreadyReplayedError` | DLQ entry já reprocessada |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `WebhookCustomerNotFoundError` | `customerId` inválido no cadastro |
| `WEBHOOK_INACTIVE` | 422 | `WebhookInactiveError` | Entrega em webhook desativado |

Erros reutilizados do sistema (já existentes): `UNAUTHORIZED` (401), `FORBIDDEN` (403), `VALIDATION_ERROR` (400).

**Implementação:** adicionar classes `Webhook*` em `src/shared/errors/http-errors.ts` (**alterar**), exportar em `src/shared/errors/index.ts` (**alterar**). O `errorMiddleware` em `src/middlewares/error.middleware.ts` (existente) trata qualquer `AppError` sem alteração — mesmo padrão de `InsufficientStockError` e `InvalidStatusTransitionError`.

---

## Estratégias de resiliência

| Mecanismo | Configuração | ADR |
|-----------|--------------|-----|
| Transação atômica | Outbox + status na mesma `$transaction` Prisma | ADR-001 |
| Timeout HTTP | 10 s por tentativa | ADR-003 |
| Retry | 5 tentativas; backoff 1m/5m/30m/2h/12h | ADR-003 |
| DLQ | Tabela `webhook_dead_letter` após esgotar retries | ADR-003 |
| Fallback manual | Replay admin (`requireRole('ADMIN')`) | ADR-003 |
| Grace period secret | 24 h com secret anterior válida | ADR-004 |
| At-least-once | Cliente deduplica por `event_id` | ADR-005 |
| Payload imutável | Snapshot na inserção; limite 64 KB | ADR-007 |

**Sem fallback automático** para e-mail ou canal alternativo (fora de escopo — RFC Q5).

---

## Observabilidade

### Logs (Pino — `src/shared/logger/index.ts`)

| Evento | Campos | Processo |
|--------|--------|----------|
| `webhook.enqueued` | `eventId`, `orderId`, `webhookId`, `toStatus`, `requestId` | API |
| `webhook.delivery.attempt` | `eventId`, `webhookId`, `attempt`, `durationMs` | Worker |
| `webhook.delivery.success` | `eventId`, `httpStatus` | Worker |
| `webhook.delivery.failed` | `eventId`, `error`, `nextRetryAt` | Worker |
| `webhook.dlq.moved` | `eventId`, `reason` | Worker |
| `webhook.dlq.replayed` | `dlqId`, `replayedBy` | API |

**Nunca logar:** `secret`, `previousSecret`, assinaturas completas. Estender redação existente do Pino para esses campos.

Propagar `X-Request-Id` do `requestLogger` (`src/middlewares/request-logger.middleware.ts`) nos logs de enfileiramento na API.

### Métricas (propostas)

| Métrica | Tipo | Descrição |
|---------|------|-----------|
| `webhook_outbox_pending_count` | gauge | Eventos pendentes na outbox |
| `webhook_delivery_duration_ms` | histogram | Latência HTTP por entrega |
| `webhook_delivery_total` | counter | `status=success\|failure` |
| `webhook_dlq_total` | counter | Eventos movidos para DLQ |
| `webhook_retry_total` | counter | Retries executados |

### Tracing

- API: correlacionar enfileiramento com `X-Request-Id` existente
- Worker: `correlationId` por batch de polling
- Entregas: `eventId` como chave de correlação log-based (OpenTelemetry fica para evolução futura)

---

## Dependências e compatibilidade

- Node.js ≥ 20, TypeScript 5.6, Express 4.21, Prisma 5.22, MySQL 8.0
- `crypto` nativo para HMAC-SHA256 e geração de secret
- `fetch` nativo (Node 20+) para HTTP outbound no worker
- JWT e roles existentes (`ADMIN`, `OPERATOR`) — sem alteração em `src/modules/auth/`
- **Não quebra** contratos de `PATCH /orders/:id/status` nem demais endpoints de pedidos

---

## Critérios de aceite técnicos

- [ ] `publishWebhookEvent()` chamado dentro de `$transaction` em `changeStatus()`, após `orderStatusHistory.create`
- [ ] Rollback de status se inserção na outbox falhar
- [ ] Worker inicia via `npm run worker`, processo independente de `src/server.ts`
- [ ] Polling 2 s; eventos processados em ordem `created_at`
- [ ] HMAC verificável com secret do endpoint; grace period 24 h na rotação
- [ ] 5 retries com intervalos 1m/5m/30m/2h/12h antes de DLQ
- [ ] Replay DLQ retorna `403` para `OPERATOR`
- [ ] Payload snapshot sem `items`; rejeição se > 64 KB
- [ ] URL `http://` rejeitada no schema Zod de `src/modules/webhooks/webhook.schemas.ts` (**a criar**)
- [ ] Testes em `tests/webhooks.test.ts` (**a criar**) passando

---

## Riscos e mitigação

| Risco | Mitigação técnica |
|-------|-------------------|
| Regressão em `changeStatus()` | Transação única; estender `tests/orders.test.ts` com assert na outbox |
| Race condition entre workers (futuro) | v1 single-worker; status `processing` com timeout de stale |
| Secret em log | Redação Pino; revisão Sofia |
| Payload oversized | Validar tamanho antes de INSERT na outbox |
| Duplicata não tratada pelo cliente | `X-Event-Id` em header e corpo; documentação no portal |

---

## Integração com o sistema existente

Esta seção mapeia **onde e como** o módulo de webhooks se conecta ao código do OMS. Cada entrada indica explicitamente se o arquivo **já existe** (reutilizar ou alterar) ou **a criar** (ainda não presente no repositório).

**Legenda:** **Existente** = arquivo no repo hoje · **Alterar** = arquivo existe, recebe mudanças · **A criar** = arquivo novo nesta feature

### 1. `src/modules/orders/order.service.ts` — enfileiramento transacional (**Alterar**)

**Estado atual (existente):** `changeStatus()` executa `this.prisma.$transaction(async (tx) => { ... })` com validação de transição, estoque, update de `orders` e insert em `order_status_history` (linhas 131–178).

**Alteração:** após `tx.orderStatusHistory.create(...)` e **antes** do `findUnique` final, inserir chamada à função **a criar** em `src/modules/webhooks/webhook.publisher.ts`:

```typescript
import { publishWebhookEvent } from '../webhooks/webhook.publisher.js'; // módulo a criar

// dentro da $transaction, após orderStatusHistory.create:
await publishWebhookEvent(tx, order, from, to);
```

**Regras:**
- Usar o objeto `order` já carregado no início da transação (contém `customerId`, `orderNumber`, `totalCents`)
- Não injetar `WebhookRepository` no `OrderService` — apenas a função pura com `tx` ([ADR-001](adrs/ADR-001-outbox-no-mysql.md))
- Falha em `publishWebhookEvent` → rollback automático da transação Prisma
- **Não alterar** assinatura pública de `changeStatus(id, input, userId)` nem o contrato HTTP em `src/modules/orders/order.routes.ts` (existente)

### 2. `src/modules/orders/order.status.ts` — gatilho de eventos (**Existente** — somente leitura)

**Estado atual:** exporta `canTransition()`, `shouldDebitStock()`, `shouldReplenishStock()` e mapa de transições (`PENDING→PAID`, etc.).

**Integração:** o webhook **não modifica** esta máquina de estados. Eventos são gerados **somente** quando `changeStatus()` completa uma transição válida. O filtro `subscribedStatuses` do endpoint decide se a linha entra na outbox para aquele `toStatus` específico.

**Implicação:** criação de pedido (`POST /orders`, status inicial `PENDING`) **não** gera webhook — apenas transições via `PATCH /orders/:id/status`.

### 3. `src/app.ts` e `src/routes/index.ts` — composição e rotas (**Alterar**)

**Estado atual (existente):** `buildControllers(prisma)` instancia repositories → services → controllers manualmente; `buildApiRouter(controllers)` monta rotas em `/api/v1`.

**Alteração em `src/app.ts`:** importar e registrar módulo webhooks (**a criar**):
```typescript
import { WebhookRepository } from './modules/webhooks/webhook.repository.js';   // a criar
import { WebhookService } from './modules/webhooks/webhook.service.js';         // a criar
import { WebhookController } from './modules/webhooks/webhook.controller.js'; // a criar

// em buildControllers():
const webhookRepository = new WebhookRepository(prisma);
const webhookService = new WebhookService(webhookRepository, prisma);
const webhookController = new WebhookController(webhookService);

return {
  // ... controllers existentes
  webhooks: webhookController,
};
```

**Alteração em `src/routes/index.ts`:** estender tipo `Controllers` e registrar rotas (**funções a criar** em `webhook.routes.ts`):
```typescript
import { buildWebhookRouter } from '../modules/webhooks/webhook.routes.js';           // a criar
import { buildAdminWebhookRouter } from '../modules/webhooks/webhook.routes.js';      // a criar

router.use('/webhooks', buildWebhookRouter(controllers.webhooks));
router.use('/admin/webhooks', buildAdminWebhookRouter(controllers.webhooks));
```

Segue o mesmo padrão dos módulos existentes: `buildOrderRouter`, `buildCustomerRouter`, `buildProductRouter`, etc. (definidos em `src/modules/*/*.routes.ts`).

### 4. `src/middlewares/auth.middleware.ts` — autorização (**Existente** — reutilizar)

**Estado atual:** `authenticate` valida JWT e popula `req.user`; `requireRole(...)` restringe por role (usado hoje em `src/modules/users/user.routes.ts` para `ADMIN`).

**Integração:**
| Rota | Middleware |
|------|------------|
| CRUD `/webhooks/*` | `authenticate` |
| `POST /admin/webhooks/dead-letter/:id/replay` | `authenticate` + `requireRole('ADMIN')` |
| Worker outbound | Sem JWT — HMAC no destino |

Replay deve registrar `req.user.id` como `replayedBy` ([ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md)).

### 5. `src/middlewares/validate.middleware.ts` + `webhook.schemas.ts` (**Existente** + **A criar**)

**Estado atual (existente):** `validate({ body, query, params })` em `src/middlewares/validate.middleware.ts` aplica Zod declarativamente nas rotas.

**Integração — a criar** `src/modules/webhooks/webhook.schemas.ts` com:
- `createWebhookSchema` — `url` deve ser `https://` (refine Zod → `WEBHOOK_INVALID_URL`)
- `updateWebhookSchema`, `listWebhooksQuerySchema`
- `subscribedStatuses` como array de `OrderStatus` enum

Registrar nas rotas webhooks (**a criar**): `validate({ body: createWebhookSchema })` — mesmo padrão de `src/modules/orders/order.schemas.ts` (existente).

### 6. `src/shared/errors/http-errors.ts` — erros tipados (**Alterar**)

**Estado atual (existente):** hierarquia `AppError` com `statusCode`, `errorCode`, `details`; classes como `InvalidStatusTransitionError`, `InsufficientStockError`.

**Integração:** adicionar classes `Webhook*` (**a criar** neste arquivo) com `errorCode` prefixado `WEBHOOK_`. Exportar em `src/shared/errors/index.ts` (**alterar**). O `errorMiddleware` em `src/middlewares/error.middleware.ts` (existente) serializa automaticamente:

```json
{ "error": { "code": "WEBHOOK_INVALID_URL", "message": "...", "details": {} } }
```

### 7. `prisma/schema.prisma` — modelos de dados (**Alterar**)

**Estado atual (existente):** enums `UserRole`, `OrderStatus`; modelos `Order`, `OrderStatusHistory`, etc. com UUID `CHAR(36)`.

**Integração — modelos a adicionar:**

| Modelo | Propósito |
|--------|-----------|
| `WebhookEndpoint` | `url`, `secretHash`, `previousSecretHash`, `previousSecretExpiresAt`, `customerId`, `subscribedStatuses` (JSON), `active` |
| `WebhookOutbox` | `eventId`, `webhookEndpointId`, `payload` (JSON snapshot), `status`, `attemptCount`, `nextRetryAt`, índices em `(status, createdAt)` |
| `WebhookDeadLetter` | `payload`, `failureReason`, `originalOutboxId`, `failedAt` |
| `WebhookDelivery` | histórico: `eventId`, `httpStatus`, `responseBody`, `durationMs`, `attemptNumber`, `success` |

Rodar `npm run db:migrate`. Secrets armazenadas hasheadas (nunca plain text no banco).

### 8. `src/worker.ts` — processo separado (**A criar**)

**Estado atual:** arquivo **não existe**; API inicia em `src/server.ts` (existente) com graceful shutdown.

**Integração — a criar** entry point espelhando bootstrap de `src/server.ts`:
- Conectar `PrismaClient` próprio (mesma `DATABASE_URL`)
- Instanciar `WebhookProcessor` (**a criar** em `src/modules/webhooks/webhook.processor.ts`) com repository
- Loop de polling 2 s com `setInterval` ou `while + sleep`
- Graceful shutdown em `SIGINT`/`SIGTERM` (desconectar Prisma)

Script **a adicionar** em `package.json` (existente): `"worker": "tsx --env-file=.env src/worker.ts"`

### 9. `src/shared/logger/index.ts` — observabilidade (**Alterar**)

**Estado atual (existente):** Pino com redação de campos sensíveis (senhas, tokens).

**Integração:** estender lista de redação para `secret`, `previousSecret`, `signature`. Worker e API usam o mesmo logger — sem dependência nova ([ADR-006](adrs/ADR-006-reuso-padroes-modulo-webhooks.md)).

### 10. `tests/helpers/factories.ts` e `tests/orders.test.ts` — testes (**Alterar** + **A criar**)

**Estado atual (existente):** `getTestApp()`, `bootstrapAuthenticatedUser()`, `createTestCustomer()`, `createTestProduct()` em `tests/helpers/factories.ts`; `tests/orders.test.ts` cobre transições e estoque.

**Integração:**
- **Alterar** `tests/helpers/factories.ts`: adicionar `createTestWebhook(customerId, statuses)`
- **A criar** `tests/webhooks.test.ts` — CRUD, outbox transacional, HMAC, retry, DLQ, replay
- **Alterar** `tests/orders.test.ts`: após `PATCH /orders/:id/status`, assert `webhook_outbox` contém linha com snapshot correto

---

**Mapa resumido de arquivos tocados:**

| Arquivo | Status no repo | Tipo de alteração |
|---------|----------------|-------------------|
| `src/modules/orders/order.service.ts` | Existente | Alterar — hook transacional |
| `src/modules/orders/order.status.ts` | Existente | Somente leitura — define gatilhos |
| `src/modules/orders/order.routes.ts` | Existente | Somente leitura — contrato HTTP inalterado |
| `src/app.ts` | Existente | Alterar — wiring webhook |
| `src/routes/index.ts` | Existente | Alterar — rotas webhook |
| `src/middlewares/auth.middleware.ts` | Existente | Reutilizar — sem alteração |
| `src/middlewares/validate.middleware.ts` | Existente | Reutilizar — sem alteração |
| `src/middlewares/error.middleware.ts` | Existente | Reutilizar — sem alteração |
| `src/shared/errors/http-errors.ts` | Existente | Alterar — classes `Webhook*` |
| `src/shared/errors/index.ts` | Existente | Alterar — exports |
| `src/shared/logger/index.ts` | Existente | Alterar — redação de secrets |
| `prisma/schema.prisma` | Existente | Alterar — novos modelos |
| `package.json` | Existente | Alterar — script `worker` |
| `tests/helpers/factories.ts` | Existente | Alterar — `createTestWebhook()` |
| `tests/orders.test.ts` | Existente | Alterar — assert na outbox |
| `src/worker.ts` | **A criar** | Entry point worker |
| `src/modules/webhooks/webhook.publisher.ts` | **A criar** | `publishWebhookEvent()` |
| `src/modules/webhooks/webhook.processor.ts` | **A criar** | Loop de polling e entrega HTTP |
| `src/modules/webhooks/webhook.routes.ts` | **A criar** | `buildWebhookRouter`, `buildAdminWebhookRouter` |
| `src/modules/webhooks/webhook.controller.ts` | **A criar** | Handlers HTTP |
| `src/modules/webhooks/webhook.service.ts` | **A criar** | Regras de negócio |
| `src/modules/webhooks/webhook.repository.ts` | **A criar** | Acesso Prisma |
| `src/modules/webhooks/webhook.schemas.ts` | **A criar** | Schemas Zod |
| `tests/webhooks.test.ts` | **A criar** | Testes de integração |
