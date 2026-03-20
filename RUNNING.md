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

Warning: this removes MySQL data.

```powershell
cd E:\eSale
docker compose down -v --remove-orphans
docker compose up -d --build
```

## Notes

- The backend runs behind Nginx.
- The admin panel already uses the frontend proxy route, so product create/list works from the UI.
- Use the reset command only when you really want a fresh database.
