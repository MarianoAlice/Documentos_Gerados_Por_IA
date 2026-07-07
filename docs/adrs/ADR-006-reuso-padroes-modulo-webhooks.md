# ADR-006: Reuso dos Padrões Existentes do Projeto

## Status

Aceito — [09:30] Larissa

## Contexto

O OMS já possui convenções estabelecidas em `src/modules/` (auth, customers, products, orders). A composição da aplicação é manual em `src/app.ts` via `buildApp()` e `buildControllers()`. Erros seguem hierarquia `AppError` em `src/shared/errors/http-errors.ts`, tratados por `src/middlewares/error.middleware.ts`. Validação usa Zod via `src/middlewares/validate.middleware.ts`. Autorização usa `authenticate` e `requireRole` em `src/middlewares/auth.middleware.ts`.

Introduzir padrões divergentes no módulo de webhooks aumentaria curva de aprendizado e inconsistência operacional.

## Decisão

O módulo de webhooks seguirá **exatamente** a estrutura dos demais domínios:

```
src/modules/webhooks/
├── webhook.routes.ts
├── webhook.controller.ts
├── webhook.service.ts
├── webhook.repository.ts
├── webhook.schemas.ts
└── webhook.worker.ts   # ou webhook.processor.ts
```

Demais diretrizes:

| Aspecto | Padrão existente a reutilizar |
|---------|-------------------------------|
| Erros | Classes estendendo `AppError`; códigos com prefixo **`WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`) |
| Logging | Pino (`src/shared/logger/index.ts`); secrets nunca em log |
| Validação | Schemas Zod + middleware `validate()` |
| Auth | `authenticate` no CRUD; `requireRole('ADMIN')` no replay de DLQ |
| Wiring | Registrar em `buildControllers()` e `buildApiRouter()` (`src/app.ts`, `src/routes/index.ts`) |
| Testes | Vitest + Supertest + banco real (`tests/`), factories em `tests/helpers/factories.ts` |
| IDs | UUID v4, consistente com `prisma/schema.prisma` |

Worker: entry point `src/worker.ts`, lógica no módulo webhooks.

## Alternativas Consideradas

### Módulo isolado com framework/padrões próprios

**Descartada.** Bruno defendeu seguir o padrão claro já existente na codebase.

### Biblioteca externa de webhooks (SDK pronto)

**Não discutida; descartada implicitamente.** O desafio exige integração nativa com transação Prisma do `OrderService`.

## Consequências

### Positivas

- Desenvolvedores do time já conhecem a estrutura (routes → controller → service → repository).
- `errorMiddleware` trata novos erros `WEBHOOK_*` sem alteração.
- Testes seguem o mesmo setup de `tests/orders.test.ts`.

### Negativas

- Injeção manual em `buildApp()` cresce com o novo módulo (já é limitação conhecida do projeto).
- **Trade-off explícito:** consistência e velocidade de desenvolvimento em troca de não introduzir abstrações mais sofisticadas (DI container, event bus).
