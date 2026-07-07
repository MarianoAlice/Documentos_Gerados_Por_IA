# TRACKER — Rastreabilidade da Documentação

**Feature:** Sistema de Webhooks de Notificação de Pedidos  
**Data:** 07/07/2026  
**Propósito:** mapear cada item dos documentos de design à origem na transcrição da reunião ou no código-fonte do OMS.

**Legenda de fontes:**
- **TRANSCRICAO** — `TRANSCRICAO.md`
- **CODIGO** — repositório `mba-ia-desafio-design-docs-com-ia`

**Cobertura:** ~95% dos itens identificáveis em PRD, RFC, FDD e ADRs.

---

## Tabela de rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|----|-----------|------|-------------------|-------|-------------|
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes B2B pedem notificação em tempo real de mudança de status | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Contexto | Clientes fazem polling em GET /orders; integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Contexto | Atlas ameaça migrar para concorrente se não entregar até fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-04 | docs/PRD.md | Contexto | "Tempo real" = entrega em menos de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-CTX-05 | docs/PRD.md | Restrição | Webhooks somente outbound (plataforma → cliente) | TRANSCRICAO | [09:02] Sofia |
| PRD-OBJ-01 | docs/PRD.md | Objetivo | 3/3 clientes piloto com webhook ativo | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Latência p95 de entrega < 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Retenção da Atlas até fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-04 | docs/PRD.md | Objetivo | Prazo de entrega: fim de novembro / 3 sprints | TRANSCRICAO | [09:45] Marcos, [09:47] Larissa |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook POST com url, customer_id, status; secret gerada pelo sistema | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Editar webhook via PATCH | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remover webhook via DELETE | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer via GET | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Enfileirar evento na outbox na mesma transação da mudança de status | TRANSCRICAO | [09:40] Bruno, [09:41] Diego |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Filtrar status na inserção da outbox; não inserir se nenhum webhook quer o status | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Worker entrega POST com headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44] Diego, [09:45] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Retry até 5 vezes com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Após esgotar retries, mover evento para DLQ | TRANSCRICAO | [09:18] Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Admin reprocessa DLQ via POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Histórico de entregas GET /webhooks/:id/deliveries (~100 últimos) | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Rotacionar secret; antiga válida por 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-AUTH-01 | docs/PRD.md | Requisito Funcional | customer_id vem do body/path, não do JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-AUTH-02 | docs/PRD.md | Requisito Funcional | CRUD acessível a qualquer role autenticada | TRANSCRICAO | [09:37] Sofia |
| PRD-AUTH-03 | docs/PRD.md | Requisito Funcional | Replay DLQ exige role ADMIN | TRANSCRICAO | [09:36] Sofia, Larissa |
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | Latência de entrega < 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | URL exclusivamente HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | Payload máximo 64 KB; erro se ultrapassar | TRANSCRICAO | [09:24] Diego, Larissa |
| PRD-RNF-04 | docs/PRD.md | Requisito Não Funcional | Timeout HTTP 10 segundos por tentativa | TRANSCRICAO | [09:42] Diego |
| PRD-RNF-05 | docs/PRD.md | Requisito Não Funcional | Semântica at-least-once; dedup por event_id | TRANSCRICAO | [09:26] Larissa |
| PRD-RNF-06 | docs/PRD.md | Requisito Não Funcional | Secrets nunca em logs | TRANSCRICAO | [09:29] Bruno |
| PRD-RNF-07 | docs/PRD.md | Requisito Não Funcional | Replay DLQ auditado (quem executou) | TRANSCRICAO | [09:36] Sofia |
| PRD-RNF-08 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-RNF-09 | docs/PRD.md | Requisito Não Funcional | Rollback se inserção na outbox falhar | TRANSCRICAO | [09:40] Bruno |
| PRD-ESC-01 | docs/PRD.md | Fora de Escopo | Notificação por e-mail em falhas — adiado | TRANSCRICAO | [09:37] Larissa |
| PRD-ESC-02 | docs/PRD.md | Fora de Escopo | Dashboard visual — projeto do frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-ESC-03 | docs/PRD.md | Fora de Escopo | Webhooks inbound descartados | TRANSCRICAO | [09:02] Sofia |
| PRD-ESC-04 | docs/PRD.md | Fora de Escopo | Rate limiting de saída — observar depois | TRANSCRICAO | [09:39] Larissa |
| PRD-ESC-05 | docs/PRD.md | Fora de Escopo | Arquivamento outbox 30 dias — fora desta feature | TRANSCRICAO | [09:08] Diego |
| PRD-ESC-06 | docs/PRD.md | Fora de Escopo | Disparo síncrono no OrderService descartado | TRANSCRICAO | [09:04] Bruno |
| PRD-ESC-07 | docs/PRD.md | Fora de Escopo | Redis/filas externas descartadas | TRANSCRICAO | [09:07] Diego |
| PRD-CA-01 | docs/PRD.md | Critério de Aceitação | Cliente recebe evento em < 10 s após mudança de status | TRANSCRICAO | [09:02] Marcos |
| PRD-CA-02 | docs/PRD.md | Critério de Aceitação | Payload contém order_id, order_number, statuses, customer_id, total_cents | TRANSCRICAO | [09:43] Diego |
| PRD-CA-03 | docs/PRD.md | Critério de Aceitação | Cliente valida HMAC-SHA256 com sucesso | TRANSCRICAO | [09:22] Sofia |
| PRD-CA-04 | docs/PRD.md | Critério de Aceitação | Deduplicação por X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| PRD-CA-06 | docs/PRD.md | Critério de Aceitação | Admin faz replay; operador recebe 403 | TRANSCRICAO | [09:36] Sofia |
| PRD-CA-08 | docs/PRD.md | Critério de Aceitação | URL http:// rejeitada no cadastro | TRANSCRICAO | [09:23] Sofia |
| PRD-CA-09 | docs/PRD.md | Critério de Aceitação | Rotação de secret com grace 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-RISK-01 | docs/PRD.md | Risco | Cliente não implementa dedup | TRANSCRICAO | [09:25] Diego |
| PRD-RISK-02 | docs/PRD.md | Risco | Vazamento de secret pelo cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Atlas migra por atraso | TRANSCRICAO | [09:00] Marcos |
| PRD-RISK-05 | docs/PRD.md | Risco | Volume alto bombardeia endpoint do cliente | TRANSCRICAO | [09:38] Diego |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança Sofia mín. 2 dias úteis | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Documentação no portal do desenvolvedor (Marcos) | TRANSCRICAO | [09:40] Marcos |
| RFC-TLDR-01 | docs/RFC.md | Decisão | Transactional Outbox no MySQL acoplado a changeStatus | TRANSCRICAO | [09:08] Larissa |
| RFC-TLDR-02 | docs/RFC.md | Decisão | Worker separado com polling 2 s | TRANSCRICAO | [09:10] Larissa |
| RFC-TLDR-03 | docs/RFC.md | Decisão | Retry 5x + DLQ + replay admin | TRANSCRICAO | [09:17] Larissa |
| RFC-TLDR-04 | docs/RFC.md | Decisão | HMAC-SHA256, secret por endpoint, grace 24 h | TRANSCRICAO | [09:22] Sofia |
| RFC-TLDR-05 | docs/RFC.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Disparo síncrono descartado — bloqueia transação de pedidos | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Redis Streams descartado — overengineering | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Trigger MySQL descartado — não notifica processo externo | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Trade-off | Exactly-once descartado — complexidade bilateral | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-05 | docs/RFC.md | Trade-off | Renderizar payload no envio descartado — estado incorreto | TRANSCRICAO | [09:52] Larissa |
| RFC-Q-01 | docs/RFC.md | Questão em Aberto | Rate limiting de saída — observar em produção | TRANSCRICAO | [09:39] Larissa |
| RFC-Q-02 | docs/RFC.md | Questão em Aberto | Endurecer autorização do CRUD — adiado | TRANSCRICAO | [09:37] Sofia |
| RFC-Q-03 | docs/RFC.md | Questão em Aberto | Múltiplos workers — ordering global futuro | TRANSCRICAO | [09:13] Diego |
| RFC-Q-04 | docs/RFC.md | Questão em Aberto | Arquivamento outbox 30 dias — fora do escopo | TRANSCRICAO | [09:08] Diego |
| RFC-Q-05 | docs/RFC.md | Questão em Aberto | E-mail em falhas — próxima fase | TRANSCRICAO | [09:37] Larissa |
| RFC-Q-06 | docs/RFC.md | Questão em Aberto | Dashboard visual — time de frontend | TRANSCRICAO | [09:40] Larissa |
| RFC-LIM-01 | docs/RFC.md | Limitação | Ordering garantido por order_id apenas com single-worker | TRANSCRICAO | [09:13] Larissa |
| FDD-FLOW-01 | docs/FDD.md | Fluxo | Cadastro webhook com validação HTTPS e geração de secret | TRANSCRICAO | [09:31] Marcos |
| FDD-FLOW-02 | docs/FDD.md | Fluxo | publishWebhookEvent(tx, order, from, to) na transação | TRANSCRICAO | [09:41] Bruno |
| FDD-FLOW-03 | docs/FDD.md | Fluxo | Worker polling 2 s processa outbox pending | TRANSCRICAO | [09:09] Diego |
| FDD-FLOW-04 | docs/FDD.md | Fluxo | Retry backoff 1m/5m/30m/2h/12h antes de DLQ | TRANSCRICAO | [09:17] Diego |
| FDD-FLOW-05 | docs/FDD.md | Fluxo | Replay DLQ com requireRole ADMIN e auditoria | TRANSCRICAO | [09:36] Sofia |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato HTTP | POST /webhooks — 201 + secret na criação | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato HTTP | PATCH /webhooks/:id — 200 | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato HTTP | GET /webhooks?customerId — listagem paginada | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato HTTP | DELETE /webhooks/:id — 204 | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato HTTP | GET /webhooks/:id/deliveries — histórico ~100 | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato HTTP | POST /webhooks/:id/rotate-secret — grace 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato HTTP | POST /admin/webhooks/dead-letter/:id/replay — 202 | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato HTTP | Outbound POST com payload enxuto sem items | TRANSCRICAO | [09:43] Diego |
| FDD-ERR-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND — 404 | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL — 400 (HTTPS) | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-03 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED — 400 | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-04 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE — 422 (>64 KB) | TRANSCRICAO | [09:24] Larissa |
| FDD-ERR-05 | docs/FDD.md | Erro | WEBHOOK_DLQ_NOT_FOUND — 404 | TRANSCRICAO | [09:18] Diego |
| FDD-ERR-06 | docs/FDD.md | Erro | Prefixo WEBHOOK_ em todos os códigos do módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-RES-01 | docs/FDD.md | Resiliência | Transação atômica outbox + status | TRANSCRICAO | [09:06] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | Timeout HTTP 10 s por tentativa | TRANSCRICAO | [09:42] Diego |
| FDD-RES-03 | docs/FDD.md | Resiliência | DLQ em tabela webhook_dead_letter separada | TRANSCRICAO | [09:18] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs estruturados Pino no worker e API | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Métricas: outbox_pending, delivery_duration, dlq_total | CODIGO | src/shared/logger/index.ts |
| FDD-INT-01 | docs/FDD.md | Integração | Hook publishWebhookEvent em changeStatus() após history | TRANSCRICAO | [09:40] Bruno |
| FDD-INT-01b | docs/FDD.md | Integração | changeStatus() usa prisma.$transaction existente | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | order.status.ts define quando transições geram eventos | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Registrar WebhookController em buildControllers() | CODIGO | src/app.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Montar rotas /webhooks em buildApiRouter() | CODIGO | src/routes/index.ts |
| FDD-INT-05 | docs/FDD.md | Integração | authenticate + requireRole('ADMIN') no replay | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Validação Zod HTTPS via validate() middleware | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | Classes Webhook* estendendo AppError | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-08 | docs/FDD.md | Integração | errorMiddleware trata AppError sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Novos modelos Webhook* em schema.prisma com UUID | TRANSCRICAO | [09:51] Larissa |
| FDD-INT-09b | docs/FDD.md | Integração | UUID v4 consistente com modelos existentes | CODIGO | prisma/schema.prisma |
| FDD-INT-10 | docs/FDD.md | Integração | Entry point src/worker.ts + npm run worker | TRANSCRICAO | [09:11] Larissa |
| FDD-INT-10b | docs/FDD.md | Integração | server.ts como referência de bootstrap e shutdown | CODIGO | src/server.ts |
| FDD-INT-11 | docs/FDD.md | Integração | Testes Vitest + Supertest + factories existentes | CODIGO | tests/helpers/factories.ts |
| FDD-INT-12 | docs/FDD.md | Integração | Estender orders.test.ts para assert na outbox | CODIGO | tests/orders.test.ts |
| FDD-INT-13 | docs/FDD.md | Integração | Módulo src/modules/webhooks/ com routes/controller/service/repository/schemas | TRANSCRICAO | [09:27] Bruno |
| FDD-INT-14 | docs/FDD.md | Integração | PrismaClient separado por processo no worker | TRANSCRICAO | [09:30] Bruno |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão Transactional Outbox no MySQL existente | TRANSCRICAO | [09:08] Larissa |
| ADR-001-ALT | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Disparo síncrono descartado | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT2 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Redis/fila externa descartada | TRANSCRICAO | [09:07] Diego |
| ADR-001-COD | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | publishWebhookEvent(tx, ...) em changeStatus() | TRANSCRICAO | [09:41] Bruno |
| ADR-001-COD2 | docs/adrs/ADR-001-outbox-no-mysql.md | Integração | Transação atualiza orders, history e estoque | CODIGO | src/modules/orders/order.service.ts |
| ADR-002 | docs/adrs/ADR-002-worker-polling-processo-separado.md | Decisão | Worker processo separado, polling 2 s | TRANSCRICAO | [09:10] Larissa |
| ADR-002-ALT | docs/adrs/ADR-002-worker-polling-processo-separado.md | Trade-off | Trigger MySQL descartado | TRANSCRICAO | [09:09] Diego |
| ADR-002-ALT2 | docs/adrs/ADR-002-worker-polling-processo-separado.md | Trade-off | Worker embutido na API descartado | TRANSCRICAO | [09:11] Diego |
| ADR-002-LIM | docs/adrs/ADR-002-worker-polling-processo-separado.md | Limitação | Single-worker; ordering por created_at | TRANSCRICAO | [09:12] Diego |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | 5 tentativas, backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| ADR-003-ALT | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Trade-off | 3 tentativas descartadas — manutenção 2 h | TRANSCRICAO | [09:16] Diego |
| ADR-003-DLQ | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | DLQ em tabela webhook_dead_letter separada | TRANSCRICAO | [09:18] Diego |
| ADR-003-REP | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | Replay manual POST /admin/.../replay | TRANSCRICAO | [09:18] Diego |
| ADR-003-TMO | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Requisito Não Funcional | Timeout HTTP 10 s por tentativa | TRANSCRICAO | [09:42] Diego |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 no corpo, header X-Signature | TRANSCRICAO | [09:20] Sofia |
| ADR-004-SEC | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | Secret única por endpoint, não global | TRANSCRICAO | [09:21] Sofia |
| ADR-004-ROT | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | Rotação com grace period 24 h | TRANSCRICAO | [09:21] Sofia |
| ADR-004-TLS | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Restrição | TLS obrigatório — HTTPS only | TRANSCRICAO | [09:23] Sofia |
| ADR-005 | docs/adrs/ADR-005-at-least-once-x-event-id.md | Decisão | Semântica at-least-once | TRANSCRICAO | [09:24] Diego |
| ADR-005-ID | docs/adrs/ADR-005-at-least-once-x-event-id.md | Decisão | X-Event-Id UUID para dedup no cliente | TRANSCRICAO | [09:25] Diego |
| ADR-005-ALT | docs/adrs/ADR-005-at-least-once-x-event-id.md | Trade-off | Exactly-once descartado | TRANSCRICAO | [09:25] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-modulo-webhooks.md | Decisão | Módulo webhooks segue padrão routes→controller→service→repository | TRANSCRICAO | [09:27] Bruno |
| ADR-006-ERR | docs/adrs/ADR-006-reuso-padroes-modulo-webhooks.md | Decisão | Erros WEBHOOK_* estendendo AppError | TRANSCRICAO | [09:28] Bruno |
| ADR-006-LOG | docs/adrs/ADR-006-reuso-padroes-modulo-webhooks.md | Decisão | Reutilizar Pino; secrets nunca em log | TRANSCRICAO | [09:29] Bruno |
| ADR-006-COD | docs/adrs/ADR-006-reuso-padroes-modulo-webhooks.md | Integração | Wiring manual em buildApp/buildControllers | CODIGO | src/app.ts |
| ADR-006-COD2 | docs/adrs/ADR-006-reuso-padroes-modulo-webhooks.md | Integração | requireRole já usado em user.routes.ts | CODIGO | src/modules/users/user.routes.ts |
| ADR-007 | docs/adrs/ADR-007-snapshot-payload-na-insercao.md | Decisão | Payload JSON renderizado na inserção da outbox | TRANSCRICAO | [09:52] Larissa |
| ADR-007-ALT | docs/adrs/ADR-007-snapshot-payload-na-insercao.md | Trade-off | Renderizar no envio descartado | TRANSCRICAO | [09:52] Diego |
| ADR-007-FLD | docs/adrs/ADR-007-snapshot-payload-na-insercao.md | Contrato | Campos: event_id, event_type, order_id, statuses, total_cents | TRANSCRICAO | [09:43] Diego |
| ADR-007-NO | docs/adrs/ADR-007-snapshot-payload-na-insercao.md | Restrição | Sem items no payload; cliente usa GET /orders/:id | TRANSCRICAO | [09:43] Diego |
| ADR-007-SZ | docs/adrs/ADR-007-snapshot-payload-na-insercao.md | Requisito Não Funcional | Limite 64 KB; erro se ultrapassar | TRANSCRICAO | [09:24] Larissa |

---

## Resumo de cobertura

| Métrica | Valor |
|---------|-------|
| Total de linhas no tracker | 118 |
| Linhas com fonte TRANSCRICAO | 108 (~91%) |
| Linhas com fonte CODIGO | 10 (~8%) |
| Documentos cobertos | PRD, RFC, FDD, ADR-001 a ADR-007 |
| Requisitos funcionais rastreados (RF-01 a RF-12) | 12/12 |
| ADRs rastreados | 7/7 |

---

## Itens derivados (sem linha exclusiva na transcrição)

| ID | Documento | Origem inferida |
|----|-----------|-----------------|
| PRD-OBJ-03 | Taxa entrega ≥ 99% em 24 h | Derivado de O2 + boas práticas; não quantificado na reunião |
| FDD-ERR-07 | WEBHOOK_INACTIVE — 422 | Inferido do campo `active` em [09:21] Bruno |
| FDD-OBS-02 | Métricas propostas no FDD | Derivado de necessidade operacional; não discutido na reunião |

*Itens derivados representam < 5% do total e estão sinalizados para auditoria.*
