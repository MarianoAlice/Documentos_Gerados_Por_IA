# toDo — Webhooks de Notificação de Pedidos

**Origem:** reunião técnica transcrita em `TRANSCRICAO.md`  
**Contexto do sistema atual:** `achados.md` (OMS em Node.js/TypeScript, Express, Prisma, MySQL)  
**Prazo estimado:** 3 sprints (revisão de segurança da Sofia incluída no final)

---

## Contexto do pedido

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) precisam ser notificados quando o status dos pedidos muda, sem depender de polling em `GET /orders`. Para eles, **"tempo real" = entrega em menos de 10 segundos**. Os webhooks são **somente outbound** (nossa plataforma envia para o cliente).

---

## Decisões arquiteturais e de produto

Itens explicitamente registrados como **decisão** na reunião.

| # | Decisão | Detalhe |
|---|---------|---------|
| D1 | **Padrão Outbox no MySQL** | Não disparar HTTP de forma síncrona no `changeStatus`. Inserir evento na `webhook_outbox` na **mesma transação SQL** que atualiza `orders`, `order_status_history` e estoque. Redis/filas externas descartadas. |
| D2 | **Worker em polling a cada 2 s** | Latência mínima aceita: ~2 s. MySQL não tem listener nativo; trigger não notifica processo externo. |
| D3 | **Worker como processo separado** | Entry point `src/worker.ts` + script `npm run worker`. Mesmo banco e `DATABASE_URL`, mas **PrismaClient em instância própria** (não compartilhar processo com a API). |
| D4 | **Retry: 5 tentativas com backoff** | Intervalos: **1 min → 5 min → 30 min → 2 h → 12 h** (~15 h de janela total). Após esgotar, mover para DLQ. |
| D5 | **DLQ em tabela separada** | `webhook_dead_letter` com payload, motivo da falha e timestamp. Outbox principal permanece limpa. |
| D6 | **Replay manual de DLQ (ADMIN)** | `POST /admin/webhooks/dead-letter/:id/replay` — recoloca evento na outbox como pendente. Exige role **ADMIN** via `requireRole`. Registrar **quem** fez o replay (auditoria). |
| D7 | **HMAC-SHA256** | Assinar o corpo do request; header `X-Signature`. **Secret única por endpoint** (não global). Rotação via API com **grace period de 24 h** (secret antiga válida em paralelo). |
| D8 | **Entrega at-least-once** | Cliente pode receber duplicatas; deduplicação pelo header **`X-Event-Id`** (UUID gerado na inserção na outbox). |
| D9 | **Reuso dos padrões do projeto** | Módulo em `src/modules/webhooks` (routes, controller, service, repository, schemas). Erros com prefixo **`WEBHOOK_`**. Reaproveitar `AppError`, Pino, `errorMiddleware`, Zod, `requireRole`. |
| D10 | **IDs em UUID** | Outbox e demais entidades seguem o padrão UUID do projeto (não auto-increment). |
| D11 | **Snapshot do payload na inserção** | Guardar payload **já renderizado** na outbox no momento da inserção — não re-renderizar no envio (preserva estado histórico). |

### Limitações conhecidas (documentar, não bloqueiam v1)

- **Ordering:** garantido por `order_id` e `created_at` enquanto houver **single-worker**. Múltiplos workers em paralelo quebram ordering global (problema futuro).
- **Arquivamento:** linhas entregues podem ser arquivadas após 30 dias — **fora do escopo** desta feature.

---

## Requisitos funcionais a implementar

### 1. Modelagem de dados (Prisma)

- [ ] Tabela de configuração de webhook: `url`, `secret` (gerada pelo sistema), `customer_id`, `active`, lista de status/eventos desejados
- [ ] Tabela `webhook_outbox`: status (`pending`, `processing`, `failed`, `delivered`), `created_at`, payload snapshot, índices em `status` e `created_at`
- [ ] Tabela `webhook_dead_letter`: payload, motivo da falha, timestamp
- [ ] Tabela/registros de entregas (`webhook_deliveries` ou equivalente) para histórico

### 2. CRUD de configuração de webhooks

Autenticação JWT normal; **`customer_id` vem do body/path** (não do JWT — JWT é do operador/admin).

| Endpoint | Descrição |
|----------|-----------|
| `POST /webhooks` | Cadastrar webhook: `url`, lista de status desejados, `customer_id`. Secret **gerada pelo sistema** e devolvida na criação. |
| `PATCH /webhooks/:id` | Editar configuração (incl. filtro de eventos) |
| `DELETE /webhooks/:id` | Remover webhook |
| `GET /webhooks` | Listar webhooks de um `customer_id` |
| `GET /webhooks/:id/deliveries` | Histórico das últimas entregas (~100): sucesso/falha, payload, response, tempo de resposta |

- [ ] Filtro de eventos por status: avaliar **na inserção na outbox** — se nenhum webhook do customer quer aquele status, **não inserir** linha
- [ ] Endpoint de rotação de secret (secret nova + antiga válida por 24 h)
- [ ] CRUD acessível a **qualquer role autenticada** (por enquanto)

### 3. Integração com pedidos

Ponto crítico mapeado em `achados.md` → `src/modules/orders/order.service.ts` → `changeStatus()`.

- [ ] Dentro da transação Prisma existente, chamar `publishWebhookEvent(tx, order, fromStatus, toStatus)` após update de status e histórico
- [ ] Função recebe o **tx client** da transação (não injetar repository inteiro no `OrderService`)
- [ ] Se inserção na outbox falhar → **rollback** da mudança de status
- [ ] Respeitar máquina de estados em `order.status.ts` — eventos nas transições de status

### 4. Worker de entrega

- [ ] `src/worker.ts` — bootstrap do processo worker
- [ ] Lógica em `src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts`
- [ ] Loop de polling: buscar pendentes mais antigos em **batch pequeno**, processar, marcar status
- [ ] HTTP timeout: **10 segundos** por chamada; falha → retry conforme D4
- [ ] Em sucesso: marcar como `delivered` e registrar entrega no histórico

### 5. Formato do evento HTTP

**Payload JSON (enxuto — sem items):**

```json
{
  "event_id": "<uuid>",
  "event_type": "order.status_changed",
  "timestamp": "<ISO 8601>",
  "order_id": "<uuid>",
  "order_number": "ORD-000001",
  "from_status": "PENDING",
  "to_status": "PAID",
  "customer_id": "<uuid>",
  "total_cents": 12345
}
```

**Headers do request:**

| Header | Conteúdo |
|--------|----------|
| `X-Event-Id` | UUID do evento (dedup no cliente) |
| `X-Signature` | HMAC-SHA256 do corpo |
| `X-Timestamp` | Timestamp do envio (anti-replay no cliente) |
| `X-Webhook-Id` | ID do endpoint webhook cadastrado |
| `Content-Type` | `application/json` |

### 6. Validações e requisitos não funcionais

- [ ] URL do webhook deve ser **HTTPS** — rejeitar `http` com erro de validação Zod (`WEBHOOK_INVALID_URL`)
- [ ] Limite de payload: **64 KB** — se ultrapassar, **erro** (não truncar)
- [ ] Códigos de erro com prefixo `WEBHOOK_`: ex. `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`
- [ ] Logs estruturados com Pino no worker (sem dados sensíveis — secret nunca em log)

### 7. Testes

- [ ] Testes de integração ponta a ponta (padrão Vitest + Supertest + banco real, conforme `achados.md`)
- [ ] Cobrir: CRUD, inserção na outbox na transação de status, retry, DLQ, replay ADMIN, HMAC, filtro por status

### 8. Wiring da aplicação

Conforme padrão em `src/app.ts` e `src/routes/index.ts`:

- [ ] Registrar módulo `webhooks` em `buildControllers()` e `buildApiRouter()`
- [ ] Adicionar script `"worker"` no `package.json`

---

## Referência rápida — onde alterar no código existente

| O quê | Onde (`achados.md` / repositório OMS) |
|-------|---------------------------------------|
| Hook de mudança de status | `src/modules/orders/order.service.ts` → `changeStatus()` |
| Transições de status | `src/modules/orders/order.status.ts` |
| Novo módulo | `src/modules/webhooks/` |
| Schema do banco | `prisma/schema.prisma` + migration |
| Erros tipados | `src/shared/errors/http-errors.ts` |
| Autorização ADMIN | `src/middlewares/auth.middleware.ts` → `requireRole('ADMIN')` |
| Composição da app | `src/app.ts`, `src/routes/index.ts` |

**Repositório analisado:** `C:\_aulasMBA\mba-ia-desafio-design-docs-com-ia`

---

## Anotações para o futuro (fora do escopo desta fase)

### Explicitamente adiados na reunião

- [ ] **Notificação por e-mail** quando webhook falha repetidamente (ex.: após 3 falhas) — próxima fase, após medir impacto
- [ ] **Rate limiting de saída** — se cliente tiver muitas mudanças de status em curto intervalo, observar em produção e decidir depois
- [ ] **Dashboard visual** para o cliente — projeto separado do time de frontend; nesta fase apenas endpoints REST
- [ ] **Arquivamento** de linhas entregues na outbox após 30 dias

### Evoluções arquiteturais mencionadas

- [ ] **Escalar para múltiplos workers** — exigirá particionamento por `order_id` ou lock pessimista; ordering global não garantido
- [ ] **Endurecer autorização** do CRUD de webhooks (hoje qualquer autenticado; futuramente pode restringir por role)
- [ ] Documentação no portal de desenvolvedor (responsabilidade do Marcos): at-least-once, dedup por `X-Event-Id`, integração HMAC

### Processo

- [ ] Revisão de segurança da Sofia (**mín. 2 dias úteis**) antes do deploy — foco em HMAC e geração/rotação de secrets
- [ ] Design doc da feature a ser revisado por Bruno e Diego antes de codar
