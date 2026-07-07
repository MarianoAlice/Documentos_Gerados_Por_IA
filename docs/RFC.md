# RFC — Webhooks de Notificação de Pedidos

| Campo | Valor |
|-------|-------|
| **Autor** | Larissa (Tech Lead) |
| **Status** | Proposta para revisão |
| **Data** | 07/07/2026 |
| **Revisores** | Marcos (PM), Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança) |

---

## Resumo executivo (TL;DR)

Propomos notificar clientes B2B sobre mudanças de status de pedidos via **webhooks outbound**, sem alterar o contrato existente de pedidos.

A solução se apoia em **sete decisões arquiteturais** já registradas:

| Decisão | ADR |
|---------|-----|
| Transactional Outbox no MySQL, acoplado a `changeStatus()` | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Worker em processo separado, polling a cada 2 s | [ADR-002](adrs/ADR-002-worker-polling-processo-separado.md) |
| Retry com backoff (5 tentativas) e DLQ com replay admin | [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md) |
| HMAC-SHA256, secret por endpoint, rotação com grace 24 h | [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) |
| Entrega at-least-once, dedup pelo cliente via `X-Event-Id` | [ADR-005](adrs/ADR-005-at-least-once-x-event-id.md) |
| Módulo `webhooks` seguindo padrões do OMS (`AppError`, Zod, Pino) | [ADR-006](adrs/ADR-006-reuso-padroes-modulo-webhooks.md) |
| Payload JSON como snapshot na inserção da outbox | [ADR-007](adrs/ADR-007-snapshot-payload-na-insercao.md) |

**Não entra nesta fase:** filas externas (Redis), disparo síncrono no service de pedidos, dashboard visual, notificação por e-mail.

Detalhamento de endpoints, contratos HTTP e integração com arquivos do código: **FDD** (`FDD.md`).

---

## Contexto e problema

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — consultam periodicamente `GET /orders` para detectar mudanças de status. A abordagem é lenta, cara e não atende a expectativa de integração moderna. Para eles, "tempo real" significa entrega em **menos de 10 segundos** ([09:02] Marcos).

O OMS já possui máquina de estados, transações Prisma e auditoria em `order_status_history`, mas **nenhum mecanismo de notificação externa** — lacuna intencional que esta feature preenche. Os webhooks são **somente outbound**: a plataforma envia para o cliente, não o contrário ([09:02] Sofia).

---

## Proposta técnica

### Visão geral

```
┌─────────────┐     transação atômica   ┌──────────────────┐
│ OrderService│ ─────────────────────►  │ orders           │
│ changeStatus│                         │ order_status_hist│
└──────┬──────┘                         │ stock            │
       │                                │ webhook_outbox   │◄─ snapshot (ADR-007)
       │                                └────────┬─────────┘
       │                                         │
       │                                polling 2s (ADR-002)
       │                                         ▼
       │                                ┌──────────────────┐
       │                                │ Worker (proc.)   │
       │                                │ HMAC (ADR-004)   │
       │                                │ retry/DLQ(ADR-003)│
       │                                └────────┬─────────┘
       │                                         │
       ▼                                         ▼
  API REST (ADR-006)                    Endpoint cliente (HTTPS)
  CRUD + replay admin
```

### Pilares da solução

**1. Consistência transacional — [ADR-001](adrs/ADR-001-outbox-no-mysql.md)**

Quando o status de um pedido muda, o evento é inserido na `webhook_outbox` **na mesma transação** que persiste `orders`, `order_status_history` e ajustes de estoque. Se a transação der rollback, o evento some junto. A entrega HTTP fica a cargo de um worker assíncrono — nunca síncrona no `OrderService`.

**2. Entrega assíncrona — [ADR-002](adrs/ADR-002-worker-polling-processo-separado.md)**

Processo separado (`src/worker.ts`), mesmo banco, `PrismaClient` próprio. Polling a cada **2 segundos** atende o SLA de < 10 s. Single-worker na v1 garante ordenação por `order_id` via `created_at`.

**3. Resiliência — [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md)**

Falhas de entrega disparam até **5 retries** com backoff (1m → 5m → 30m → 2h → 12h). Após esgotar, evento vai para `webhook_dead_letter`. Admin com role `ADMIN` pode fazer replay manual.

**4. Segurança — [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)**

Cada endpoint tem secret própria. Payload assinado com HMAC-SHA256 (`X-Signature`). URLs exclusivamente HTTPS. Rotação de secret com grace period de 24 h.

**5. Semântica de entrega — [ADR-005](adrs/ADR-005-at-least-once-x-event-id.md)**

Garantia **at-least-once**: o cliente pode receber duplicatas e deduplica pelo `X-Event-Id` (UUID gerado na inserção). Exactly-once foi descartado por complexidade desproporcional.

**6. Organização do código — [ADR-006](adrs/ADR-006-reuso-padroes-modulo-webhooks.md)**

Novo módulo `src/modules/webhooks/` com a mesma estrutura dos demais domínios. Erros com prefixo `WEBHOOK_`, logging via Pino, validação Zod, wiring em `buildApp()`.

**7. Imutabilidade do evento — [ADR-007](adrs/ADR-007-snapshot-payload-na-insercao.md)**

Payload JSON renderizado e persistido na inserção — o worker envia o snapshot sem reconsultar o pedido. Evento enxuto (sem itens); cliente usa `GET /orders/:id` para detalhes.

### O que fica fora deste RFC

Contratos de API, matriz de erros, fluxos passo a passo e pontos de integração com arquivos do repositório estão no **FDD** — este documento não duplica esse nível de detalhe.

---

## Alternativas consideradas

Cada alternativa abaixo foi discutida na reunião e descartada em favor das decisões registradas nos ADRs.

### 1. Disparo síncrono no `OrderService` → descartada ([ADR-001](adrs/ADR-001-outbox-no-mysql.md))

**Proposta:** chamar o endpoint do cliente dentro da transação de `changeStatus()`.

**Trade-off que motivou o descarte:** cliente lento ou offline bloquearia mudança de status para outros pedidos; rollback por falha HTTP é inaceitável para o domínio de pedidos ([09:04] Bruno, Larissa).

### 2. Redis Streams / fila externa → descartada ([ADR-001](adrs/ADR-001-outbox-no-mysql.md))

**Proposta:** publicar eventos em Redis após commit da transação.

**Trade-off que motivou o descarte:** time pequeno; subir Redis Cluster é overengineering quando o MySQL existente resolve com outbox ([09:07] Diego).

### 3. Trigger MySQL para acordar o worker → descartada ([ADR-002](adrs/ADR-002-worker-polling-processo-separado.md))

**Proposta:** trigger no insert da outbox para notificar o processo externo.

**Trade-off que motivou o descarte:** MySQL não notifica processos externos nativamente; workarounds (arquivo, endpoint interno) são frágeis. Polling de 2 s atende o SLA ([09:09] Diego).

### 4. Exactly-once delivery → descartada ([ADR-005](adrs/ADR-005-at-least-once-x-event-id.md))

**Proposta:** garantir que o cliente recebe cada evento uma única vez.

**Trade-off que motivou o descarte:** exige coordenação bilateral; Stripe e GitHub usam at-least-once com dedup no consumidor ([09:25] Diego).

### 5. Renderizar payload no envio (somente `order_id` na outbox) → descartada ([ADR-007](adrs/ADR-007-snapshot-payload-na-insercao.md))

**Proposta:** guardar referência na outbox e montar JSON na hora do POST.

**Trade-off que motivou o descarte:** se o pedido mudar após a transição, o evento refletiria estado incorreto — compromete auditoria e confiança do cliente ([09:52] Larissa, Diego).

---

## Questões em aberto

Pontos levantados na reunião e **não fechados** nesta proposta — permanecem para observação ou fases futuras.

| # | Questão | Status | Origem |
|---|---------|--------|--------|
| Q1 | **Rate limiting de saída** — muitos pedidos mudando em curto intervalo podem bombardear o endpoint do cliente | Observar em produção; decidir depois | [09:39] Diego, Larissa |
| Q2 | **Endurecer autorização** do CRUD de webhooks (hoje qualquer autenticado) | Adiado | [09:37] Sofia |
| Q3 | **Escalar para múltiplos workers** — perda de ordering global | Problema futuro; v1 single-worker ([ADR-002](adrs/ADR-002-worker-polling-processo-separado.md)) | [09:13] Diego |
| Q4 | **Arquivamento** de linhas entregues na outbox após 30 dias | Fora do escopo desta feature | [09:08] Diego |
| Q5 | **Notificação por e-mail** quando webhook falha repetidamente | Próxima fase | [09:37] Larissa |
| Q6 | **Dashboard visual** para o cliente gerenciar webhooks | Projeto separado do time de frontend | [09:40] Larissa |

---

## Impacto e riscos

| Área | Impacto |
|------|---------|
| **Banco** | Novas tabelas: configuração de webhook, outbox, DLQ, histórico de entregas |
| **Deploy** | Novo processo worker além da API HTTP ([ADR-002](adrs/ADR-002-worker-polling-processo-separado.md)) |
| **Pedidos** | Alteração em `changeStatus()` — ponto crítico de regressão ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)) |
| **Clientes** | Devem implementar verificação HMAC ([ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)) e dedup ([ADR-005](adrs/ADR-005-at-least-once-x-event-id.md)) |
| **Segurança** | Revisão dedicada da Sofia antes do deploy (HMAC, geração e rotação de secrets) |

| Risco | Probabilidade | Mitigação |
|-------|---------------|-----------|
| Regressão na transação de pedidos | Média | Outbox na mesma `$transaction`; testes de integração |
| Secret vazada pelo cliente | Média | Secret por endpoint; rotação com grace period de 24 h ([ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)) |
| Cliente não trata duplicatas | Média | Documentação no portal; `X-Event-Id` em header e corpo ([ADR-005](adrs/ADR-005-at-least-once-x-event-id.md)) |

**Prazo estimado:** 3 sprints, incluindo revisão de segurança  mais dois dias para revisão da Sofia ([09:46] Sofia, [09:47] Larissa).

---

## Decisões relacionadas

Registro completo das decisões arquiteturais desta proposta:

| ADR | Título | Decisão em uma linha |
|-----|--------|----------------------|
| [ADR-001](adrs/ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL | Evento na mesma transação de `changeStatus()`; entrega HTTP assíncrona |
| [ADR-002](adrs/ADR-002-worker-polling-processo-separado.md) | Worker em processo separado com polling | `src/worker.ts`, polling 2 s, PrismaClient próprio |
| [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md) | Retry com backoff e DLQ | 5 tentativas (1m/5m/30m/2h/12h); DLQ separada; replay ADMIN |
| [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint | Assinatura do corpo; secret única; HTTPS obrigatório; rotação 24 h |
| [ADR-005](adrs/ADR-005-at-least-once-x-event-id.md) | At-least-once com X-Event-Id | Cliente deduplica por UUID do evento |
| [ADR-006](adrs/ADR-006-reuso-padroes-modulo-webhooks.md) | Reuso dos padrões do projeto | Módulo `webhooks` alinhado a `AppError`, Zod, Pino, `requireRole` |
| [ADR-007](adrs/ADR-007-snapshot-payload-na-insercao.md) | Snapshot do payload na inserção | JSON renderizado na outbox; sem itens; limite 64 KB |

---

**Próximo passo:** revisão desta proposta por Bruno e Diego; implementação conforme FDD após aprovação; revisão de segurança da Sofia antes do deploy.
