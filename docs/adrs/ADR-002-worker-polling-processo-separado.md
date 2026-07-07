# ADR-002: Worker em Processo Separado com Polling de 2 Segundos

## Status

Aceito — [09:10] Larissa

## Contexto

Com o padrão outbox definido ([ADR-001](ADR-001-outbox-no-mysql.md)), é necessário definir como os eventos pendentes serão consumidos e entregues. O requisito de negócio aceita entrega em menos de 10 segundos como "tempo real".

MySQL não oferece mecanismo nativo de notificação a processos externos (diferente de `NOTIFY/LISTEN` do PostgreSQL). Triggers no banco executam SQL, mas não avisam um processo Node.js de forma limpa.

A API atual roda em `src/server.ts` com graceful shutdown. Reiniciar a API não pode interromper o processamento de webhooks.

## Decisão

1. **Processo separado:** criar entry point `src/worker.ts` com script `npm run worker`. O worker **não** roda na mesma instância Node da API.
2. **Mesmo banco, PrismaClient próprio:** mesma `DATABASE_URL`, mas instância separada de `PrismaClient` (um client por processo).
3. **Polling a cada 2 segundos:** o worker busca eventos `pending` mais antigos em batch pequeno, processa e atualiza status.
4. **Single-worker na v1:** ordering implícito por `created_at`; garantia de ordem por `order_id` enquanto houver apenas um worker.

Latência mínima aceita: **~2 segundos** no pior caso.

## Alternativas Consideradas

### Trigger MySQL para reatividade

**Descartada.** Diego explicou que trigger não notifica processo externo; soluções improvisadas (escrever em arquivo, bater em endpoint interno) são frágeis e não justificam a complexidade.

### Worker embutido no processo da API

**Descartada.** Se a API reinicia, o worker para junto — perda de processamento contínuo.

### Múltiplos workers em paralelo (v1)

**Adiada.** Diego documentou que escalar exigiria particionamento por `order_id` ou lock pessimista; problema do futuro, não desta entrega.

## Consequências

### Positivas

- Atende o SLA de "< 10 segundos" com folga.
- Implementação simples, sem nova infraestrutura.
- Isolamento de falhas: crash ou deploy da API não derruba o worker.

### Negativas

- Polling gera leituras periódicas no banco mesmo sem eventos (custo aceitável no volume atual).
- **Trade-off explícito:** simplicidade e previsibilidade em troca de latência mínima fixa (~2 s) e ausência de garantia de ordering global com múltiplos workers futuros.
