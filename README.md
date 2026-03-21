# eSale

Multi-tenant SaaS e-commerce platform with Database-per-Tenant architecture.

## Stack

- **Backend**: ASP.NET Core (.NET 10), MediatR, FluentValidation, AutoMapper, EF Core, MySQL
- **Frontend**: Next.js
- **Infrastructure**: Docker Compose, Nginx reverse proxy, Redis, Hangfire

## Architecture

```
┌──────────┐     ┌──────────┐     ┌──────────────┐     ┌─────────────┐
│  Nginx   │────→│ Frontend │     │   Backend    │────→│   MySQL     │
│ (proxy)  │────→│ Next.js  │     │  ASP.NET     │     │ per-tenant  │
└──────────┘     └──────────┘     └──────────────┘     └─────────────┘
                                        │
                                        ├── esale_central (tenants + hangfire)
                                        ├── esale_tenant_acme (users + products)
                                        └── esale_tenant_globex (users + products)
```

Each tenant gets its own MySQL database with isolated Identity (users, roles) and business data (products).

## Contents

```
docker-compose.yml     → Full stack orchestration
nginx/                 → Reverse proxy config
eSale-api/             → Backend (.NET)
eSale-web/             → Frontend (Next.js)
.env.example           → Environment variables template
```

## Documentation

| File | Description |
|------|-------------|
| [RUNNING.md](RUNNING.md) | Local development commands |
| [DEPLOYMENT.md](DEPLOYMENT.md) | Production Ubuntu server setup |
| [eSale-api/PROJECT_CYCLE.md](eSale-api/PROJECT_CYCLE.md) | Deep request flow documentation |
| [eSale-api/DATABASE-PER-TENANT.md](eSale-api/DATABASE-PER-TENANT.md) | Multi-tenancy architecture |

## Quick Start

```bash
cp .env.example .env
# Edit .env with your values
docker compose up -d --build
```

- Frontend: `http://localhost`
- API: `http://localhost/api/products`
- Hangfire: `http://localhost/hangfire`
