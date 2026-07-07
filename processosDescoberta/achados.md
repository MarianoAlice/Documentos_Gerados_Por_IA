# Achados — Order Management System (OMS)

**Projeto analisado:** `C:\_aulasMBA\Documentos_Gerados_Por_IA\mba-ia-desafio-design-docs-com-ia`  
**Data da análise:** 07/07/2026  
**Propósito deste documento:** servir como referência de arquitetura e funcionalidades para qualquer alteração futura na aplicação.

---

## 1. Visão Geral

O repositório contém uma **API REST funcional** em Node.js/TypeScript que implementa um **Order Management System (OMS)** — sistema de gestão de pedidos. A aplicação é o boilerplate de contexto para o desafio acadêmico *"Da Reunião ao Documento: Design Docs Gerados por IA"*, cujo objetivo é documentar uma futura feature de **webhooks de notificação de pedidos**, sem alterar o código existente.

**Capacidades atuais:**
- Autenticação JWT com controle de papéis (`ADMIN`, `OPERATOR`)
- CRUD de clientes, produtos e pedidos
- Máquina de estados para ciclo de vida do pedido
- Controle transacional de estoque
- Auditoria imutável de mudanças de status

**Lacunas intencionais (não implementadas):**
- Webhooks, eventos de domínio, filas ou workers
- Notificações externas
- Frontend
- OpenAPI/Swagger
- Refresh token ou blacklist de JWT

---

## 2. Stack Tecnológica

| Camada | Tecnologia | Versão |
|--------|------------|--------|
| Runtime | Node.js (ESM) | ≥ 20 |
| Linguagem | TypeScript (strict) | 5.6 |
| Framework HTTP | Express | 4.21 |
| ORM / Banco | Prisma + MySQL | 5.22 / 8.0 |
| Validação | Zod | 3.23 |
| Autenticação | JWT + bcrypt | 9.0 / 5.1 |
| Logging | Pino | 9.5 |
| Testes | Vitest + Supertest | 2.1 / 7.0 |
| Dev tooling | tsx, ESLint, Prettier | — |
| Infra local | Docker Compose | MySQL 8 |

---

## 3. Estrutura do Projeto

```
mba-ia-desafio-design-docs-com-ia/
├── src/
│   ├── server.ts                 # Entry point — bootstrap e graceful shutdown
│   ├── app.ts                    # Composição da aplicação Express
│   ├── config/                   # Env (Zod) e Prisma singleton
│   ├── routes/index.ts           # Agregador de rotas /api/v1
│   ├── middlewares/              # Auth, validação, erros, logging
│   ├── modules/                  # Módulos de domínio
│   │   ├── auth/
│   │   ├── users/
│   │   ├── customers/
│   │   ├── products/
│   │   └── orders/               # Núcleo do sistema
│   └── shared/                   # Erros, logger, helpers HTTP
├── prisma/
│   ├── schema.prisma
│   ├── seed.ts
│   └── migrations/
├── tests/                        # Testes de integração
├── docs/                         # Templates do desafio (PRD, RFC, FDD, ADRs)
├── docker-compose.yml
├── package.json
└── .env.example
```

---

## 4. Arquitetura

### 4.1 Padrão: monólito modular em camadas

Cada módulo de domínio segue a mesma estrutura:

```
routes → controller → service → repository → Prisma
         ↑ schemas (Zod)
```

| Camada | Responsabilidade |
|--------|------------------|
| **Routes** | Define endpoints; aplica middlewares (`authenticate`, `validate`, `requireRole`) |
| **Controller** | Adapta HTTP ↔ service; delega erros via `next(err)` |
| **Service** | Regras de negócio, orquestração, transações Prisma |
| **Repository** | Acesso a dados via Prisma |
| **Schemas** | Contratos de entrada (body, query, params) com Zod |

### 4.2 Composição e injeção de dependências

A montagem é **manual** em `buildApp()` / `buildControllers()` (`src/app.ts`). Não há container de DI — cada repository recebe `PrismaClient` no construtor, cada service recebe seu repository, e cada controller recebe seu service.

Isso facilita testes: `buildApp({ prisma })` aceita o cliente Prisma real ou de teste.

### 4.3 Pipeline de middlewares (ordem em `buildApp`)

1. `express.json({ limit: '1mb' })`
2. `requestLogger` — gera/propaga `X-Request-Id`, loga duração da requisição
3. `GET /health` — health check
4. Rotas `/api/v1`
5. Catch-all → `NotFoundError`
6. `errorMiddleware` — tratamento centralizado de erros

### 4.4 Convenções de código

- **ESM nativo** — imports com sufixo `.js` (resolução TypeScript → Node ESM)
- **Classes** para services, controllers e repositories
- **Handlers como arrow functions** nos controllers
- **Validação declarativa** — middleware `validate({ body, query, params })` com Zod
- **Erros tipados** — hierarquia `AppError` com `statusCode` + `errorCode` + `details`
- **Respostas de erro uniformes** — `{ error: { code, message, details? } }`
- **Paginação padronizada** — helper `paginated()` em todos os endpoints de listagem
- **Valores monetários em centavos** — inteiros (`priceCents`, `totalCents`, etc.)
- **IDs UUID v4** — `CHAR(36)` no MySQL

---

## 5. Módulos e Funcionalidades

### 5.1 Auth (`src/modules/auth/`)

| Endpoint | Auth | Descrição |
|----------|------|-----------|
| `POST /api/v1/auth/register` | Não | Cria usuário com role `ADMIN` ou `OPERATOR` |
| `POST /api/v1/auth/login` | Não | Retorna JWT Bearer |
| `GET /api/v1/auth/me` | Bearer | Perfil do usuário autenticado |

- Senhas hasheadas com bcrypt (10 rounds)
- JWT stateless, sem refresh token
- Payload JWT: `{ sub, email, role }`

### 5.2 Users (`src/modules/users/`)

| Endpoint | Auth | Descrição |
|----------|------|-----------|
| `GET /api/v1/users/:id` | Bearer + **ADMIN** | Consulta usuário por ID |

Único endpoint com restrição de role explícita via `requireRole('ADMIN')`.

### 5.3 Customers (`src/modules/customers/`)

CRUD completo, todas as rotas autenticadas:

| Endpoint | Descrição |
|----------|-----------|
| `GET /` | Listagem paginada com busca por nome, e-mail ou documento |
| `GET /:id` | Detalhe do cliente |
| `POST /` | Criação com endereço estruturado (JSON) |
| `PATCH /:id` | Atualização parcial |
| `DELETE /:id` | Exclusão (204) |

- E-mail único validado na criação/atualização
- Endereço: `{ street, number, city, state, zipCode }`

### 5.4 Products (`src/modules/products/`)

CRUD completo, todas as rotas autenticadas:

| Endpoint | Descrição |
|----------|-----------|
| `GET /` | Listagem paginada com filtro `active` e busca por nome/SKU |
| `GET /:id` | Detalhe do produto |
| `POST /` | Criação com SKU único, preço em centavos, estoque |
| `PATCH /:id` | Atualização parcial |
| `DELETE /:id` | Exclusão (204) |

- SKU único
- Flag `active` — produtos inativos não podem entrar em pedidos

### 5.5 Orders (`src/modules/orders/`) — núcleo do sistema

| Endpoint | Descrição |
|----------|-----------|
| `GET /` | Listagem com filtros: `status`, `customerId`, `from`, `to` |
| `GET /:id` | Pedido com items, history e customer |
| `POST /` | Criação de pedido |
| `PATCH /:id/status` | Mudança de status |
| `DELETE /:id` | Exclusão (apenas `PENDING` ou `CANCELLED`) |

#### Criação de pedido (`POST /orders`)

1. Valida que há ao menos um item
2. Agrega itens duplicados por `productId`
3. Valida cliente existente
4. Valida produtos ativos (rejeita inativos com `INACTIVE_PRODUCT`)
5. Calcula preços no servidor (cliente **não** envia preço)
6. Valida que desconto não excede subtotal
7. Gera `orderNumber` sequencial atômico (`ORD-000001`)
8. Cria histórico inicial `null → PENDING`
9. Status inicial: `PENDING` (sem débito de estoque)
10. Tudo em transação Prisma

#### Mudança de status (`PATCH /orders/:id/status`)

1. Valida transição via máquina de estados
2. Rejeita transição para o mesmo status (`INVALID_STATUS_TRANSITION`)
3. Débito de estoque na transição `PENDING → PAID`
4. Reposição de estoque em `PAID/PROCESSING → CANCELLED`
5. Registra auditoria em `OrderStatusHistory`
6. Tudo em transação Prisma

**Arquivos centrais:** `order.service.ts`, `order.status.ts`

---

## 6. Máquina de Estados dos Pedidos

Definida em `src/modules/orders/order.status.ts`:

```
PENDING    → PAID, CANCELLED
PAID       → PROCESSING, CANCELLED
PROCESSING → SHIPPED, CANCELLED
SHIPPED    → DELIVERED
DELIVERED  → (terminal)
CANCELLED  → (terminal)
```

### Regras de estoque

| Transição | Ação |
|-----------|------|
| `PENDING → PAID` | Débito de estoque (valida disponibilidade) |
| `PAID → CANCELLED` | Reposição de estoque |
| `PROCESSING → CANCELLED` | Reposição de estoque |
| Demais transições | Sem alteração de estoque |

**Decisão de design:** estoque é debitado somente no pagamento, não na criação do pedido. Isso evita reserva física prematura.

---

## 7. Modelo de Dados (Prisma)

### Enums

- `UserRole`: `ADMIN`, `OPERATOR`
- `OrderStatus`: `PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`

### Entidades

| Modelo | Tabela | Campos-chave |
|--------|--------|--------------|
| **User** | `users` | UUID, email único, passwordHash, role |
| **Customer** | `customers` | UUID, email único, document, address (JSON) |
| **Product** | `products` | UUID, SKU único, priceCents, stockQuantity, active |
| **Order** | `orders` | UUID, orderNumber único, status, totais em centavos |
| **OrderItem** | `order_items` | Itens com preço congelado (`unitPriceCents`) |
| **OrderStatusHistory** | `order_status_history` | Auditoria: fromStatus, toStatus, changedBy, reason |
| **OrderNumberSequence** | `order_number_sequence` | Sequência atômica para `ORD-XXXXXX` |

### Relacionamentos

```
User 1──* Order (createdBy)
User 1──* OrderStatusHistory (changedBy)
Customer 1──* Order
Order 1──* OrderItem ──* Product
Order 1──* OrderStatusHistory
```

### Convenções

- Cascade delete em `OrderItem` e `OrderStatusHistory` ao excluir pedido
- Preço do item congelado no momento da criação do pedido (`unitPriceCents`)
- `OrderNumberSequence` com `upsert` atômico evita race conditions

### Seed (`prisma/seed.ts`)

Popula dados de demonstração:
- 2 usuários: `admin@oms.local` / `admin123`, `operador@oms.local` / `operator123`
- 10 clientes, 20 produtos
- 26 pedidos em todos os status, com histórico simulado

---

## 8. API REST — Referência Completa

**Base URL:** `/api/v1`  
**Health check:** `GET /health` → `{ "status": "ok" }`

### Formato de erro padrão

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Descrição legível",
    "details": {}
  }
}
```

### Formato de listagem paginada

```json
{
  "data": [],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "total": 100,
    "totalPages": 5
  }
}
```

### Códigos de erro do sistema

| Código | HTTP | Contexto |
|--------|------|----------|
| `VALIDATION_ERROR` | 400 | Falha de validação Zod ou regra de negócio |
| `UNAUTHORIZED` | 401 | Token ausente, inválido ou expirado |
| `FORBIDDEN` | 403 | Role insuficiente |
| `NOT_FOUND` | 404 | Recurso inexistente |
| `EMAIL_ALREADY_USED` | 409 | E-mail duplicado |
| `SKU_ALREADY_USED` | 409 | SKU duplicado |
| `INVALID_STATUS_TRANSITION` | 409 | Transição de status inválida |
| `INVALID_ORDER_STATE_FOR_DELETE` | 409 | Delete em status não permitido |
| `INACTIVE_PRODUCT` | 422 | Produto inativo no pedido |
| `INSUFFICIENT_STOCK` | 422 | Estoque insuficiente ao pagar |
| `INTERNAL_SERVER_ERROR` | 500 | Erro não tratado |

**Arquivos de referência:** `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`

---

## 9. Autenticação e Autorização

### Fluxo

1. Cliente envia `Authorization: Bearer <token>` no header
2. Middleware `authenticate` valida JWT e popula `req.user`
3. Middleware `requireRole(...)` verifica papel quando necessário

### Papéis

| Role | Permissões |
|------|------------|
| `ADMIN` | Acesso total, incluindo `GET /users/:id` |
| `OPERATOR` | Acesso a todas as rotas exceto consulta de usuários |

**Decisão de design:** a maioria dos endpoints exige apenas autenticação, não role específica. Apenas `GET /users/:id` requer `ADMIN`.

**Arquivos de referência:** `src/middlewares/auth.middleware.ts`

---

## 10. Configuração e Ambiente

### Variáveis de ambiente (validadas com Zod em `src/config/env.ts`)

| Variável | Obrigatória | Default | Descrição |
|----------|-------------|---------|-----------|
| `NODE_ENV` | Não | `development` | `development` \| `test` \| `production` |
| `PORT` | Não | `3000` | Porta HTTP |
| `LOG_LEVEL` | Não | `info` | Nível do Pino |
| `DATABASE_URL` | **Sim** | — | Connection string MySQL |
| `JWT_SECRET` | **Sim** (min 16 chars) | — | Segredo JWT |
| `JWT_EXPIRES_IN` | Não | `8h` | Expiração do token |
| `SHADOW_DATABASE_URL` | Para migrations | — | Usada pelo Prisma Migrate |

### Scripts npm

| Script | Comando | Uso |
|--------|---------|-----|
| `dev` | `tsx watch --env-file=.env src/server.ts` | Desenvolvimento com hot reload |
| `build` | `tsc -p tsconfig.build.json` | Compila para `dist/` |
| `start` | `node --env-file=.env dist/server.js` | Produção |
| `db:migrate` | `prisma migrate dev` | Migrations |
| `db:reset` | `prisma migrate reset --force` | Reset + seed |
| `db:seed` | `tsx --env-file=.env prisma/seed.ts` | Seed manual |
| `test` | `vitest run` | Testes de integração |
| `lint` / `format` | ESLint / Prettier | Qualidade de código |

### Setup local

```bash
cp .env.example .env
docker compose up -d
npm install
npm run db:migrate
npm run db:seed
npm run dev
```

---

## 11. Testes

### Abordagem

- **Vitest** com ambiente Node, execução sequencial (`fileParallelism: false`)
- **Supertest** para requisições HTTP reais contra `buildApp()`
- **Banco real** via Prisma (sem mocks de repository)
- Timeout: 30s por teste/hook

### Setup (`tests/setup.ts`)

- `beforeAll`: conecta Prisma
- `beforeEach`: limpa todas as tabelas (ordem respeitando FKs)
- `afterAll`: desconecta Prisma

### Cobertura atual

| Arquivo | Cobertura |
|---------|-----------|
| `tests/auth.test.ts` | Registro, login, `/me`, validações, conflitos |
| `tests/orders.test.ts` | Criação, totais, transições, estoque, filtros, delete |

### Lacunas de cobertura

Não há testes para: customers, products, users (além do fluxo auth), middlewares isolados.

### Factories (`tests/helpers/factories.ts`)

- `getTestApp()` — singleton do Express app
- `createTestUser()`, `createTestCustomer()`, `createTestProduct()`
- `loginAndGetToken()`, `bootstrapAuthenticatedUser()`

---

## 12. Observabilidade

- **Logger:** Pino com redação de dados sensíveis (senhas, tokens, hashes)
- **Request ID:** header `X-Request-Id` gerado/propagado para rastreio
- **Logs estruturados:** JSON em produção, pretty em desenvolvimento
- **Graceful shutdown:** `SIGINT`/`SIGTERM` fecham servidor HTTP e desconectam Prisma

**Arquivos de referência:** `src/shared/logger/index.ts`, `src/middlewares/request-logger.middleware.ts`

---

## 13. Pontos de Extensão para Alterações Futuras

Esta seção mapeia onde integrar novas funcionalidades, com base no contexto do desafio (feature de webhooks) e em padrões gerais de extensão.

### 13.1 Adicionar um novo módulo de domínio

1. Criar pasta em `src/modules/<nome>/` com: `routes`, `controller`, `service`, `repository`, `schemas`
2. Registrar em `buildControllers()` e `buildApiRouter()` em `src/app.ts` / `src/routes/index.ts`
3. Adicionar modelos no `prisma/schema.prisma` e rodar migration
4. Criar testes de integração em `tests/`

### 13.2 Emitir eventos em mudança de status (webhooks)

| Ponto de integração | Arquivo | Como |
|---------------------|---------|------|
| Hook de mudança de status | `src/modules/orders/order.service.ts` → `changeStatus()` | Após atualizar status e criar histórico, inserir registro na outbox |
| Definição de transições | `src/modules/orders/order.status.ts` | Determinar quais transições geram eventos |
| Novos erros | `src/shared/errors/http-errors.ts` | Criar classes `WEBHOOK_*` seguindo padrão existente |
| Autorização de config | `src/middlewares/auth.middleware.ts` → `requireRole` | Proteger endpoints de gestão de webhooks |
| Observabilidade do worker | `src/shared/logger/index.ts` | Logs estruturados do processamento |
| Novo modelo de dados | `prisma/schema.prisma` | Tabelas: `WebhookEndpoint`, `WebhookOutbox`, `WebhookDelivery` |

### 13.3 Alterar regras de pedido

| Alteração | Arquivo(s) |
|-----------|------------|
| Transições de status | `src/modules/orders/order.status.ts` |
| Lógica de criação/status/estoque | `src/modules/orders/order.service.ts` |
| Queries e persistência | `src/modules/orders/order.repository.ts` |
| Contratos de entrada | `src/modules/orders/order.schemas.ts` |
| Rotas e middlewares | `src/modules/orders/order.routes.ts` |

### 13.4 Adicionar novos endpoints

1. Definir schema Zod em `*.schemas.ts`
2. Adicionar handler no controller
3. Registrar rota com `validate()` e middlewares de auth
4. Criar/estender erros em `src/shared/errors/` se necessário

### 13.5 Adicionar novos erros

1. Estender `AppError` ou classes em `src/shared/errors/http-errors.ts`
2. O `errorMiddleware` trata automaticamente qualquer `AppError`
3. Usar `errorCode` semântico (padrão: `SCREAMING_SNAKE_CASE`)

---

## 14. Arquivos-Chave — Mapa de Referência Rápida

| Arquivo | Papel |
|---------|-------|
| `src/server.ts` | Entry point — bootstrap, graceful shutdown |
| `src/app.ts` | Factory `buildApp()` — wiring de todos os módulos |
| `src/routes/index.ts` | Monta router `/api/v1` |
| `src/config/env.ts` | Validação de ambiente na inicialização |
| `src/config/database.ts` | Singleton PrismaClient |
| `src/modules/orders/order.service.ts` | **Coração do negócio** — transações, estoque, status |
| `src/modules/orders/order.status.ts` | Máquina de estados e regras de estoque |
| `src/middlewares/error.middleware.ts` | Tratamento global de erros |
| `src/middlewares/auth.middleware.ts` | JWT + `requireRole` |
| `src/middlewares/validate.middleware.ts` | Validação Zod declarativa |
| `src/shared/errors/http-errors.ts` | Classes de erro tipadas |
| `src/shared/http/response.ts` | Helper de paginação |
| `prisma/schema.prisma` | Schema do banco de dados |
| `prisma/seed.ts` | Dados de demonstração |

---

## 15. Decisões Arquiteturais Relevantes

| Decisão | Implicação para alterações |
|---------|---------------------------|
| Preço calculado no servidor | Nunca confiar em preço enviado pelo cliente |
| Estoque debitado só em `PENDING→PAID` | Novas reservas de estoque precisam respeitar esse ponto |
| `OrderNumberSequence` separada | Padrão para gerar identificadores sequenciais atômicos |
| Transações Prisma em operações críticas | Qualquer operação multi-tabela deve usar `$transaction` |
| Histórico imutável de status | Nunca alterar registros de `OrderStatusHistory` |
| JWT stateless | Sem invalidação de token; logout é apenas client-side |
| Injeção manual de dependências | Novos módulos seguem o padrão de `buildControllers()` |
| Logger com redação | Dados sensíveis nunca aparecem nos logs |
| Validação com Zod em dois níveis | Env na inicialização; requests via middleware `validate()` |

---

## 16. O Que NÃO Existe (e pode precisar ser criado)

| Capacidade | Status | Relevância para webhooks |
|------------|--------|--------------------------|
| Webhooks / notificações externas | Ausente | Feature principal do desafio |
| Padrão Outbox | Ausente | Decisão arquitetural discutida na reunião |
| Workers / processos em background | Ausente | Worker de polling para entrega de webhooks |
| Filas (SQS, RabbitMQ, etc.) | Ausente | Alternativa descartada na reunião |
| Eventos de domínio / pub-sub | Ausente | — |
| API de relatórios/analytics | Ausente | — |
| Frontend | Ausente | — |
| OpenAPI/Swagger | Ausente | — |
| Refresh token / blacklist JWT | Ausente | — |
| Rate limiting | Ausente | — |
| CORS configurado | Ausente | — |

---

## 17. Guia Rápido para Modificações Comuns

### Alterar regra de transição de status
→ Editar `transitions` em `src/modules/orders/order.status.ts`  
→ Atualizar testes em `tests/orders.test.ts`

### Adicionar campo a um modelo
→ Editar `prisma/schema.prisma`  
→ Rodar `npm run db:migrate`  
→ Atualizar schemas Zod, repository, service e controller do módulo  
→ Atualizar seed se necessário

### Criar novo endpoint autenticado
→ Adicionar schema Zod  
→ Adicionar método no service  
→ Adicionar handler no controller  
→ Registrar rota com `authenticate` + `validate()`

### Adicionar novo código de erro
→ Criar classe em `src/shared/errors/http-errors.ts`  
→ Exportar em `src/shared/errors/index.ts`  
→ Lançar no service onde a regra de negócio se aplica

### Rodar testes
→ Garantir MySQL disponível (Docker Compose)  
→ `npm test` (limpa banco entre testes automaticamente)

---

*Documento gerado a partir da análise do código-fonte em `C:\_aulasMBA\Documentos_Gerados_Por_IA\mba-ia-desafio-design-docs-com-ia`. Para alterações na aplicação, consulte os arquivos referenciados diretamente no repositório.*
