# ADR-003: Política de Retry com Backoff Exponencial e DLQ

## Status

Aceito — [09:17] Larissa

## Contexto

Endpoints de webhook dos clientes podem estar temporariamente indisponíveis (manutenções de até 2 horas já ocorreram com clientes reais). O sistema precisa retentar entregas sem bloquear o processamento de outros eventos, e encerrar tentativas de forma previsível quando a falha é permanente.

## Decisão

### Retry

- **5 tentativas** após a falha inicial.
- **Backoff exponencial:** 1 min → 5 min → 30 min → 2 h → 12 h (~15 h de janela total).
- Timeout HTTP por tentativa: **10 segundos** ([09:42] Diego).

### Dead Letter Queue (DLQ)

- Após esgotar as 5 tentativas, mover o evento para tabela separada **`webhook_dead_letter`** com payload, motivo da falha e timestamp.
- A tabela `webhook_outbox` principal permanece enxuta para leitura operacional.

### Replay manual

- Endpoint `POST /admin/webhooks/dead-letter/:id/replay` recoloca o evento na outbox como `pending`.
- Exige role **ADMIN** via `requireRole('ADMIN')` (`src/middlewares/auth.middleware.ts`).
- Registrar **quem** executou o replay para auditoria.

## Alternativas Consideradas

### 3 tentativas (proposta de Bruno)

**Descartada.** Diego argumentou que 3 retries em ~30 minutos matariam eventos durante manutenções planejadas de manhã (até 2 h).

### Retry indefinido com backoff

**Descartada.** Eventos ficariam pendurados indefinidamente se o cliente desaparecesse; poluiria a outbox e dificultaria operação.

### Marcar como `failed` na própria outbox (sem tabela DLQ)

**Descartada.** Diego preferiu tabela separada para evidência de debug e reprocessamento, mantendo a outbox principal limpa.

## Consequências

### Positivas

- Cobre janelas realistas de indisponibilidade do cliente.
- DLQ separada facilita suporte e reprocessamento controlado.
- Replay admin com auditoria atende cenários de recuperação manual.

### Negativas

- Cliente offline por mais de ~15 h perde entrega automática (precisa replay manual ou consulta via API de pedidos).
- **Trade-off explícito:** entrega persistente com limite finito de esforço, em troca de não manter fila infinita de eventos zumbis.
