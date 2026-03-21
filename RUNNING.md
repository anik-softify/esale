# Running eSale Locally

## Start the full stack

```powershell
cd E:\eSale
docker compose up -d
```

## Rebuild and start

Use this after code changes when containers need a fresh build.

```powershell
cd E:\eSale
docker compose up -d --build
```

## Stop the stack

```powershell
cd E:\eSale
docker compose down
```

## Open in browser

- Frontend: `http://localhost`
- Admin panel: `http://localhost/admin/products`
- Hangfire dashboard: `http://localhost/hangfire`

## Tenant Workflow

### 1. Provision a tenant

```bash
curl -X POST http://localhost/api/tenants/provision \
  -H "Content-Type: application/json" \
  -d '{"name": "Acme Corp"}'
# Returns: "a1b2c3d4-..."
```

### 2. Register a user in that tenant

```bash
curl -X POST http://localhost/api/account/register \
  -H "Content-Type: application/json" \
  -H "X-Tenant-Id: a1b2c3d4-..." \
  -d '{"firstName":"Anik","lastName":"Das","email":"anik@example.com","password":"MyPassword1"}'
# Returns: JWT token
```

### 3. Login

```bash
curl -X POST http://localhost/api/account/login \
  -H "Content-Type: application/json" \
  -H "X-Tenant-Id: a1b2c3d4-..." \
  -d '{"email":"anik@example.com","password":"MyPassword1"}'
# Returns: JWT token
```

### 4. Use the API

```bash
# Create a product
curl -X POST http://localhost/api/products \
  -H "Content-Type: application/json" \
  -H "X-Tenant-Id: a1b2c3d4-..." \
  -d '{"name":"Widget","sku":"WDG-001","price":29.99,"stockQuantity":100}'

# List products
curl http://localhost/api/products \
  -H "X-Tenant-Id: a1b2c3d4-..."
```

## Update only frontend

```powershell
cd E:\eSale
docker compose build frontend
docker compose up -d --no-deps frontend
```

## Update only backend

```powershell
cd E:\eSale
docker compose build backend
docker compose up -d --no-deps backend
```

## Check running containers

```powershell
cd E:\eSale
docker compose ps
```

## View logs

```powershell
cd E:\eSale
docker compose logs -f
```

## Reset everything including database data

Warning: this removes all MySQL data including tenant databases.

```powershell
cd E:\eSale
docker compose down -v --remove-orphans
docker compose up -d --build
```

## Notes

- The backend runs behind Nginx.
- Every API request (except tenant provisioning) requires an `X-Tenant-Id` header or a JWT with a `tenantId` claim.
- Each tenant has its own MySQL database with isolated users and data.
- Use the reset command only when you really want a fresh database.
