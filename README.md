# eSale — Multi-Tenant SaaS E-Commerce Platform

eSale is a **multi-tenant SaaS backend** built with ASP.NET Core (.NET 10) using **Clean Architecture**. Every tenant (business customer) gets their own isolated MySQL database. A Next.js frontend connects through an Nginx reverse proxy.

---

## Architecture Overview

```
                        ┌─────────────────────────────────────────────┐
  Browser / Client      │              Docker Environment              │
       │                │                                              │
       ▼                │  ┌─────────┐      ┌──────────────────────┐  │
  HTTP Request ─────────┼─►│  Nginx  │─────►│  Backend (ASP.NET)   │  │
                        │  │ (proxy) │      │     eSale.Api         │  │
                        │  └─────────┘      └──────────┬───────────┘  │
                        │       │                       │              │
                        │       ▼                       ▼              │
                        │  ┌─────────┐      ┌──────────────────────┐  │
                        │  │ Next.js │      │     MySQL Databases   │  │
                        │  │(Frontend│      │                       │  │
                        │  └─────────┘      │  esale_central        │  │
                        │                   │  esale_tenant_acme    │  │
                        │                   │  esale_tenant_globex  │  │
                        │                   └──────────────────────┘  │
                        │                                              │
                        │            Redis Cache    Hangfire Jobs      │
                        └─────────────────────────────────────────────┘
```

**Key principle**: Each tenant gets its own separate database. Tenant A can never see Tenant B's data.

---

## Project Structure

```
eSale/
├── docker-compose.yml          ← Full stack orchestration
├── nginx/                      ← Reverse proxy config
├── eSale-api/                  ← Backend (.NET 10)
│   ├── eSale.Domain/           ← Core business entities & interfaces
│   ├── eSale.Application/      ← Use cases, CQRS, validation
│   ├── eSale.Infrastructure/   ← Database, cache, jobs, auth
│   ├── eSale.Api/              ← HTTP controllers, middleware
│   └── eSale.Tests/            ← Unit tests
├── eSale-web/                  ← Frontend (Next.js)
└── .env.example                ← Environment variables template
```

---

## Clean Architecture (4 Layers)

The backend is split into 4 projects. Each layer only talks to the one below it.

```
┌─────────────────────────────────┐
│         eSale.Api               │  HTTP in/out, middleware, routing
├─────────────────────────────────┤
│      eSale.Application          │  Business rules, CQRS, validation
├─────────────────────────────────┤
│       eSale.Domain              │  Entities, repository interfaces
├─────────────────────────────────┤
│    eSale.Infrastructure         │  MySQL, Redis, Hangfire, JWT
└─────────────────────────────────┘
```

### Layer Responsibilities

| Layer | What it does | What it knows about |
|---|---|---|
| **Domain** | Business entities (`Product`, `Tenant`, `ApplicationUser`) + abstract interfaces | Nothing (zero dependencies) |
| **Application** | Commands, Queries, Validators, AutoMapper profiles | Domain only |
| **Infrastructure** | EF Core, repositories, JWT, Redis, Hangfire | Application interfaces |
| **Api** | Controllers, middleware, DI wiring | Application (via MediatR) |

---

## Multi-Tenancy — How It Works

This is the core feature of eSale. Here is the exact flow for every request:

```
1. Client sends:
   GET /api/products
   Headers:
     Authorization: Bearer <jwt_token>
     X-Tenant-Id: acme-corp-guid

2. TenantMiddleware runs:
   - Reads X-Tenant-Id header
   - Also reads tenantId claim from JWT
   - Both must match (prevents cross-tenant attacks)

3. TenantConnectionResolver queries central DB:
   SELECT * FROM Tenants WHERE Id = 'acme-corp-guid'
   → Returns: DatabaseName = "esale_tenant_acme_corp"

4. Builds connection string:
   Server=mysql;Database=esale_tenant_acme_corp;User=root;Password=...

5. AppDbContext opens connection to tenant's database

6. Repository queries Products table in THAT database only

7. Response returned — Tenant B never touched
```

### Two Database Types

**`esale_central`** (one, shared):
- `Tenants` table — registry of all customers
- Hangfire job tables

**`esale_tenant_*`** (one per customer):
- `Products` — that tenant's product catalog
- `AspNetUsers`, `AspNetRoles` etc. — Identity tables (per-tenant user accounts)

### Provisioning a New Tenant

```http
POST /api/tenants/provision
{ "name": "Acme Corp" }
```

This:
1. Inserts a row into `esale_central.Tenants`
2. Auto-generates database name: `esale_tenant_acme_corp`
3. Creates the MySQL database
4. Applies EF Core schema (all tables) to that new database

---

## CQRS Pattern (MediatR)

All business logic goes through **Commands** (write) and **Queries** (read) via MediatR.

### Commands

| Command | What it does |
|---|---|
| `RegisterCommand` | Creates a new user in the tenant DB, returns JWT |
| `LoginCommand` | Validates credentials, returns JWT |
| `CreateProductCommand` | Adds a product, invalidates product list cache |
| `ProvisionTenantCommand` | Creates a new tenant + their database |

### Queries

| Query | Cached? | Cache TTL |
|---|---|---|
| `GetProductByIdQuery` | No | — |
| `GetProductListQuery` | Yes (per tenant) | 2 minutes |

### Pipeline Behaviors (automatic middleware for all requests)

```
Request comes in
      ↓
ValidationBehavior   ← runs FluentValidation, returns 400 if invalid
      ↓
CachingBehavior      ← returns cached result if available (for queries)
      ↓
Handler              ← actual business logic
```

---

## Authentication (JWT)

### Login / Register Flow

```
POST /api/account/login
Headers: X-Tenant-Id: <guid>
Body: { "email": "...", "password": "..." }

→ Response: {
    "token": "eyJhbGci...",
    "userId": "...",
    "email": "...",
    "firstName": "...",
    "lastName": "..."
  }
```

### JWT Token Contents

```json
{
  "sub": "user-guid",
  "email": "user@example.com",
  "firstName": "John",
  "lastName": "Doe",
  "tenantId": "acme-guid",
  "jti": "correlation-id",
  "exp": 1234567890
}
```

The `tenantId` inside the token lets the API resolve the correct database on every subsequent request without the client needing to pass the header again (but the header is also accepted and must match the token).

### Token Configuration

| Setting | Default |
|---|---|
| Algorithm | HMAC SHA256 |
| Expiry | 24 hours |
| Issuer | `eSale.Api` |
| Audience | `eSale.Web` |

---

## API Endpoints

### Account (public — no auth required)

| Method | Route | Description |
|---|---|---|
| POST | `/api/account/register` | Register a new user |
| POST | `/api/account/login` | Login and get JWT |

### Products (requires `Authorization: Bearer <token>` + `X-Tenant-Id`)

| Method | Route | Description |
|---|---|---|
| GET | `/api/products` | List all products (cached 2 min) |
| GET | `/api/products/{id}` | Get product by ID |
| POST | `/api/products` | Create a product |

### Tenants (admin — no tenant context needed)

| Method | Route | Description |
|---|---|---|
| POST | `/api/tenants/provision` | Provision a new tenant |

---

## Domain Entities

### BaseEntity (inherited by all entities)

```
Id          Guid        Primary key (auto-generated)
TenantId    Guid        Which tenant owns this record
CreatedAt   DateTime    Set automatically on insert
UpdatedAt   DateTime?   Set automatically on update
```

### Product

```
Name            string      Max 200 chars
Description     string?     Max 2000 chars
Sku             string      Max 50 chars, unique per tenant
Price           decimal     Precision 18,2
StockQuantity   int
IsActive        bool
```

### Tenant (in central DB)

```
Name            string      Max 200 chars, globally unique
DatabaseName    string      e.g. "esale_tenant_acme_corp"
ConnectionString string?    Optional override
IsActive        bool
```

### ApplicationUser (per-tenant, extends ASP.NET Identity)

```
FirstName   string
LastName    string
CreatedAt   DateTime
+ all standard Identity fields (Email, PasswordHash, etc.)
```

---

## Caching (Redis)

Product list queries are cached in Redis, scoped per tenant:

```
Cache key:  tenant:{tenantId}:products:list
TTL:        2 minutes
```

If Redis is unavailable, the system falls back to in-memory cache automatically — no downtime.

Invalidation: When a product is created, the `products:list` cache key for that tenant is deleted.

---

## Background Jobs (Hangfire)

Hangfire runs on the central database and handles async tasks:

- **Welcome Email** — queued when a user registers
- Dashboard available at `/hangfire`
- Poll interval: 15 seconds

---

## Validation

All inputs are validated with FluentValidation before the handler runs:

| Field | Rule |
|---|---|
| `FirstName` / `LastName` | Required, max 100 chars |
| `Email` | Required, valid email format |
| `Password` | Required, min 6 chars |
| Product `Name` | Max 200 chars |
| Product `Sku` | Max 50 chars |
| Product `Price` | Must be > 0 |
| Product `StockQuantity` | Must be >= 0 |

Invalid requests return `400 Bad Request` with a structured error body:

```json
{
  "statusCode": 400,
  "message": "Validation failed",
  "errors": {
    "Email": ["'Email' is not a valid email address."],
    "Price": ["'Price' must be greater than 0."]
  },
  "traceId": "..."
}
```

---

## Error Handling

All exceptions are caught by `GlobalExceptionMiddleware` and returned as structured JSON:

| Exception | HTTP Status |
|---|---|
| `ValidationException` | 400 Bad Request |
| `NotFoundException` | 404 Not Found |
| `UnauthorizedAccessException` | 401 Unauthorized |
| `BadHttpRequestException` | 400 Bad Request |
| Any other | 500 Internal Server Error |

---

## Logging (Serilog)

Structured logging with three sinks:

| Sink | Purpose |
|---|---|
| Console | Local development |
| File | Daily rolling file (`logs/log-.txt`) |
| Seq | Centralized log aggregation (optional) |

Every log entry is enriched with request context (TraceId, TenantId where available).

---

## Tech Stack

| Category | Technology |
|---|---|
| Backend framework | ASP.NET Core (.NET 10) |
| Database | MySQL 8 via Pomelo EF Core provider |
| ORM | Entity Framework Core 9 |
| CQRS | MediatR 14 |
| Validation | FluentValidation 11 |
| Object mapping | AutoMapper 16 |
| Authentication | ASP.NET Core Identity + JWT Bearer |
| Caching | Redis (StackExchange.Redis) + in-memory fallback |
| Background jobs | Hangfire 1.8 with MySQL storage |
| Logging | Serilog |
| Frontend | Next.js (React) |
| Reverse proxy | Nginx |
| Containerization | Docker Compose |
| Testing | xUnit + Moq |

---

## Running Locally

### Prerequisites

- Docker and Docker Compose
- Copy `.env.example` to `.env` and fill in values

### Start everything

```bash
docker-compose up -d
```

This starts:
- MySQL (central + will auto-create tenant DBs)
- Redis
- Hangfire (background job processor)
- ASP.NET Core API
- Next.js frontend
- Nginx reverse proxy

### First tenant

```bash
curl -X POST http://localhost/api/tenants/provision \
  -H "Content-Type: application/json" \
  -d '{ "name": "My Store" }'
```

Take the returned GUID — that is your `X-Tenant-Id` for all future requests.

### Register a user

```bash
curl -X POST http://localhost/api/account/register \
  -H "Content-Type: application/json" \
  -H "X-Tenant-Id: <tenant-guid>" \
  -d '{
    "firstName": "John",
    "lastName": "Doe",
    "email": "john@example.com",
    "password": "secret123"
  }'
```

---

## Environment Variables

| Variable | Description |
|---|---|
| `ConnectionStrings__DefaultConnection` | Central MySQL connection string |
| `JwtSettings__Key` | Secret key (min 32 characters) |
| `JwtSettings__Issuer` | Token issuer (default: `eSale.Api`) |
| `JwtSettings__Audience` | Token audience (default: `eSale.Web`) |
| `JwtSettings__ExpirationHours` | Token lifetime in hours (default: `24`) |
| `Redis__ConnectionString` | Redis connection (optional) |
| `Serilog__SeqUrl` | Seq log server URL (optional) |

---

## Contents

| Path | Description |
|---|---|
| `docker-compose.yml` | Full stack orchestration |
| `nginx/` | Reverse proxy config |
| `eSale-api/` | Backend (.NET 10) |
| `eSale-web/` | Frontend (Next.js) |
| `.env.example` | Environment variables template |
