# PRD — Sistema de Webhooks de Notificação de Pedidos

**Feature:** Webhooks outbound de mudança de status de pedidos  
**Versão:** 1.1  
**Data:** 07/07/2026  
**Prazo alvo:** fim de novembro (3 sprints)  
**Documentos relacionados:** [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/)

---

## Resumo e contexto da feature

O OMS (Order Management System) permite que operadores gerenciem pedidos com máquina de estados e auditoria de mudanças de status. Hoje, clientes B2B integrados consultam periodicamente `GET /orders` para detectar alterações — abordagem lenta, cara e propensa a atrasos.

Esta feature introduz **webhooks outbound**: quando o status de um pedido muda, a plataforma notifica automaticamente os endpoints cadastrados pelo cliente. Para os clientes piloto, **"tempo real" significa entrega em menos de 10 segundos** — não é necessário push instantâneo, mas é inaceitável depender de polling manual.

A proposta técnica está consolidada no RFC e nos ADRs; este PRD define o **por quê** e o **o quê** do ponto de vista de produto e negócio.

---

## Problema e motivação

Na semana anterior à reunião técnica, três clientes B2B formalizaram o pedido:

| Cliente | Situação |
|---------|----------|
| **Atlas Comercial** | Ameaça migrar para concorrente se a feature não for entregue até o fim do trimestre |
| **MaxDistribuição** | Integração lenta por dependência de polling |
| **Nova Cargo** | Mesma dor de atualização manual de status |

**Dor atual:** clientes consultam `GET /orders` de tempos em tempos para verificar mudanças. Isso torna a integração lenta, cara e frágil.

**Oportunidade:** oferecer notificação push confiável, reduzir carga na API de consulta e atender expectativa mínima de mercado para integração B2B.

---

## Público-alvo e cenários de uso

| Persona | Necessidade |
|---------|-------------|
| **Integrador B2B** (time técnico do cliente) | Cadastrar URL de webhook, escolher status de interesse, validar assinatura HMAC, deduplicar eventos recebidos |
| **Operador da plataforma** | Configurar webhooks via API autenticada (JWT do OMS) em nome de um `customer_id` |
| **Admin da plataforma** | Reprocessar manualmente eventos que falharam permanentemente (DLQ) |
| **Product/CS (Marcos)** | Comunicar prazo aos clientes e publicar guia de integração no portal do desenvolvedor |

> O JWT identifica o **usuário da plataforma** (operador/admin), não o cliente B2B. O `customer_id` é informado explicitamente no cadastro do webhook.

### Cenários de uso

1. **Cadastro:** integrador autentica na API, cria webhook com URL HTTPS e lista de status desejados; recebe `secret` gerada pelo sistema (exibida uma única vez).
2. **Notificação:** pedido muda de `PAID` → `PROCESSING`; em até 10 segundos o cliente recebe POST no endpoint cadastrado com dados do pedido e assinatura HMAC.
3. **Falha temporária:** endpoint do cliente em manutenção; plataforma retenta automaticamente; após esgotar tentativas, evento vai para DLQ; admin faz replay manual.
4. **Rotação de secret:** cliente suspeita de vazamento; solicita nova secret via API; secret antiga permanece válida por 24 h para migração.
5. **Consulta operacional:** integrador lista histórico das últimas entregas (sucesso/falha, payload, resposta, tempo de resposta).

---

## Objetivos e métricas de sucesso

| # | Objetivo | Métrica | Meta |
|---|----------|---------|------|
| O1 | Eliminar dependência de polling | Clientes piloto com webhook ativo | **3/3** (Atlas, MaxDistribuição, Nova Cargo) |
| O2 | Entrega em tempo aceitável | Latência p95 entre mudança de status e primeira tentativa de entrega | **< 10 segundos** |
| O3 | Confiabilidade de entrega | Taxa de entrega bem-sucedida em 24 h (excluindo endpoints permanentemente inválidos) | **≥ 99%** |
| O4 | Retenção comercial | Atlas permanece na plataforma | Sem migração até fim do trimestre |
| O5 | Adoção da integração | Clientes piloto validam HMAC e dedup em homologação | **3/3** aprovados antes de produção |

---

## Escopo

### Incluso (v1)

| Capacidade | Descrição resumida |
|------------|-------------------|
| CRUD de webhooks | Cadastro, edição, remoção e listagem por `customer_id` |
| Filtro por status | Cliente escolhe quais transições deseja receber |
| Secret por endpoint | Gerada pelo sistema; rotacionável com grace period de 24 h |
| Notificação automática | Disparo quando status de pedido muda (outbox transacional) |
| Resiliência | Retry com backoff (5 tentativas) e DLQ |
| Replay admin | Reprocessamento manual de eventos falhos (role ADMIN) |
| Histórico de entregas | Consulta das ~100 últimas entregas por webhook |
| Segurança | HMAC-SHA256, HTTPS obrigatório, headers padronizados |
| Semântica de entrega | At-least-once com `X-Event-Id` para dedup no cliente |

### Fora de escopo

Itens explicitamente **descartados ou adiados** na reunião técnica:

| Item | Decisão | Origem |
|------|---------|--------|
| **Notificação por e-mail** em falhas repetidas | Adiado — próxima fase, após medir impacto | [09:37] Larissa |
| **Dashboard visual** para o cliente | Fora — projeto do time de frontend; v1 é só API | [09:40] Larissa |
| **Webhooks inbound** (cliente envia para nós) | Fora — escopo é somente outbound | [09:02] Sofia |
| **Rate limiting de saída** | Adiado — observar em produção | [09:39] Larissa |
| **Arquivamento automático** da outbox após 30 dias | Fora desta feature | [09:08] Diego |
| **Disparo síncrono** no service de pedidos | Descartado | [09:04] Bruno |
| **Redis / filas externas** | Descartado — overengineering | [09:07] Diego |

---

## Requisitos funcionais

Requisitos derivados da reunião e consolidados no FDD. Detalhamento de endpoints: ver [FDD — Contratos públicos](FDD.md#contratos-públicos).

| ID | Requisito | Prioridade |
|----|-----------|------------|
| RF-01 | Cadastrar webhook (`POST`) com `url`, `customer_id` e lista de status; sistema gera e devolve `secret` | Must |
| RF-02 | Editar webhook (`PATCH`): URL, status subscritos, estado ativo | Must |
| RF-03 | Remover webhook (`DELETE`) | Must |
| RF-04 | Listar webhooks de um `customer_id` (`GET`) | Must |
| RF-05 | Ao mudar status de pedido, enfileirar evento na outbox **na mesma transação** da mudança | Must |
| RF-06 | Filtrar por status na inserção: se nenhum webhook do customer quer aquele `to_status`, não criar evento | Must |
| RF-07 | Worker entregar evento via HTTP POST com payload JSON e headers (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`) | Must |
| RF-08 | Em falha de entrega, retentar até 5 vezes com backoff (1m / 5m / 30m / 2h / 12h) | Must |
| RF-09 | Após esgotar retries, mover evento para DLQ | Must |
| RF-10 | Admin reprocessar item da DLQ via replay manual (`POST /admin/webhooks/dead-letter/:id/replay`) | Must |
| RF-11 | Consultar histórico de entregas (`GET /webhooks/:id/deliveries`, ~100 últimos) | Must |
| RF-12 | Rotacionar secret via API; secret antiga válida por 24 h | Must |

**Autorização (v1):** CRUD acessível a qualquer usuário autenticado; replay de DLQ exige role `ADMIN`.

---

## Requisitos não funcionais

| ID | Requisito | Meta |
|----|-----------|------|
| RNF-01 | Latência de entrega aceitável ao negócio | < 10 segundos |
| RNF-02 | URL de webhook exclusivamente HTTPS | Rejeitar `http://` |
| RNF-03 | Tamanho máximo do payload do evento | 64 KB (erro se ultrapassar) |
| RNF-04 | Timeout por tentativa de entrega HTTP | 10 segundos |
| RNF-05 | Semântica de entrega at-least-once | Cliente deduplica por `event_id` |
| RNF-06 | Secrets nunca expostas em logs | Redação no logger |
| RNF-07 | Replay de DLQ auditado | Registrar quem executou |
| RNF-08 | Disponibilidade do worker | Processo separado da API HTTP |
| RNF-09 | Consistência status ↔ evento | Rollback da mudança de status se outbox falhar |

---

## Decisões e trade-offs principais

Decisões arquiteturais registradas nos ADRs. O PRD resume o impacto de produto; detalhes técnicos no RFC.

| Decisão | ADR | Trade-off (produto) |
|---------|-----|---------------------|
| Outbox no MySQL, transação atômica | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) | Cliente sempre notificado se status mudou; entrega em segundos, não instantânea |
| Worker em processo separado, polling 2 s | [ADR-002](adrs/ADR-002-worker-polling-processo-separado.md) | Operação simples; latência mínima ~2 s |
| Retry 5x + DLQ + replay admin | [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md) | Resiliência a manutenções; após ~15 h, intervenção manual |
| HMAC-SHA256, secret por endpoint | [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) | Integração segura; cliente gerencia rotação de secrets |
| At-least-once com `X-Event-Id` | [ADR-005](adrs/ADR-005-at-least-once-x-event-id.md) | Nenhum evento perdido; cliente deve implementar dedup |
| Reuso dos padrões do OMS | [ADR-006](adrs/ADR-006-reuso-padroes-modulo-webhooks.md) | Entrega mais rápida; sem inovação desnecessária em stack |
| Snapshot do payload na inserção | [ADR-007](adrs/ADR-007-snapshot-payload-na-insercao.md) | Evento reflete estado histórico; payload enxuto (sem itens) |

---

## Dependências

| Dependência | Responsável | Impacto se atrasar |
|-------------|-------------|-------------------|
| OMS existente (módulo `orders`, Prisma, MySQL) | Time de Pedidos | Bloqueia integração transacional |
| Infraestrutura MySQL / deploy do worker | Time de Plataforma | Bloqueia entrega assíncrona |
| Revisão de segurança (mín. 2 dias úteis) | Sofia | Bloqueia deploy em produção |
| Documentação no portal do desenvolvedor | Marcos | Bloqueia adoção pelos 3 clientes piloto |
| Validação em homologação | Atlas, MaxDistribuição, Nova Cargo | Bloqueia go-live |

---

## Riscos e mitigação

| Risco | Prob. | Impacto | Mitigação |
|-------|-------|---------|-----------|
| Cliente não implementa dedup e processa eventos duplicados | Média | Alto | Documentação destacada no portal; `X-Event-Id` em header e corpo ([ADR-005](adrs/ADR-005-at-least-once-x-event-id.md)) |
| Vazamento de secret pelo cliente | Média | Alto | Secret por endpoint; rotação com grace 24 h; nunca logar secrets ([ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)) |
| Atlas migra por atraso na entrega | Média | Alto | Prazo de 3 sprints; priorizar clientes piloto em homologação cedo |
| Prazo de 3 sprints insuficiente | Média | Alto | MVP: CRUD + outbox + worker + HMAC na sprint 1–2; DLQ/replay na sprint 2 |
| Volume alto de eventos sobrecarrega endpoint do cliente | Baixa | Médio | Monitorar; rate limiting avaliado para fase futura (RFC Q1) |
| Regressão na mudança de status de pedidos | Média | Alto | Outbox na mesma transação; testes de integração orders + webhooks |

---

## Critérios de aceitação

Critérios verificáveis do ponto de vista de produto e integração:

- [ ] **CA-01:** Cliente piloto cadastra webhook HTTPS e recebe evento em < 10 s após mudança de status elegível
- [ ] **CA-02:** Evento contém `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents`
- [ ] **CA-03:** Cliente valida `X-Signature` HMAC-SHA256 com sucesso usando a secret recebida
- [ ] **CA-04:** Cliente deduplica eventos repetidos usando `X-Event-Id` sem efeito colateral no negócio
- [ ] **CA-05:** Falha temporária do endpoint dispara retries; após 5 tentativas, evento aparece na DLQ
- [ ] **CA-06:** Admin com role `ADMIN` consegue replay de item da DLQ; operador recebe `403`
- [ ] **CA-07:** `GET /webhooks/:id/deliveries` retorna histórico com sucesso/falha, payload, resposta e tempo
- [ ] **CA-08:** URL `http://` é rejeitada no cadastro
- [ ] **CA-09:** Rotação de secret funciona com grace period de 24 h (secret antiga ainda válida)
- [ ] **CA-10:** Os 3 clientes piloto aprovam integração em homologação

---

## Estratégia de testes e validação

| Fase | O quê | Como |
|------|-------|------|
| **Automatizado** | Fluxos críticos de API e outbox | Vitest + Supertest + MySQL real (`tests/webhooks.test.ts`) |
| **Segurança** | HMAC, geração e rotação de secrets | Revisão dedicada da Sofia (mín. 2 dias úteis antes do deploy) |
| **Homologação** | Integração ponta a ponta com clientes reais | Atlas, MaxDistribuição e Nova Cargo validam em ambiente de staging |
| **Produção** | Monitorar latência p95 e taxa de entrega | Métricas definidas no FDD; alertas se p95 > 10 s ou taxa < 99% |

**Cenários de teste obrigatórios:** CRUD webhooks, enfileiramento transacional, entrega worker, retry, DLQ, replay ADMIN, HMAC, filtro por status, rotação de secret.

**Critério de go-live:** CA-01 a CA-10 atendidos + aprovação da Sofia + confirmação dos 3 clientes piloto.
