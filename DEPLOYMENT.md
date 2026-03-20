# eSale Deployment

## Server Layout

```text
/opt/esale/
  .env
  docker-compose.yml
  nginx/
    default.conf
    certbot/
  eSale-api/
  eSale-web/
```

## Fresh Ubuntu Setup

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl ca-certificates
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
```

```bash
sudo mkdir -p /opt/esale/nginx/certbot/conf
sudo mkdir -p /opt/esale/nginx/certbot/www
sudo chown -R $USER:$USER /opt/esale
cd /opt/esale
```

```bash
git clone https://github.com/anik-softify/esale-api.git eSale-api
git clone https://github.com/anik-softify/esale-web.git eSale-web
```

Copy `.env.example` to `.env`, then set real values.

## Start

```bash
docker compose up -d --build
```

## Update Frontend Only

```bash
cd /opt/esale/eSale-web
git pull origin main
cd /opt/esale
docker compose build frontend
docker compose up -d --no-deps frontend
```

## Update Backend Only

```bash
cd /opt/esale/eSale-api
git pull origin main
cd /opt/esale
docker compose build backend
docker compose up -d --no-deps backend
```

## SSL Certificates

```bash
docker run --rm \
  -v /opt/esale/nginx/certbot/conf:/etc/letsencrypt \
  -v /opt/esale/nginx/certbot/www:/var/www/certbot \
  certbot/certbot certonly --webroot \
  -w /var/www/certbot \
  -d yourdomain.com -d www.yourdomain.com
```

```bash
docker run --rm \
  -v /opt/esale/nginx/certbot/conf:/etc/letsencrypt \
  -v /opt/esale/nginx/certbot/www:/var/www/certbot \
  certbot/certbot certonly --webroot \
  -w /var/www/certbot \
  -d api.yourdomain.com
```

```bash
docker compose restart nginx
```
