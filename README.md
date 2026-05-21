# AppFlowy Cloud — Unraid Community Apps Templates

Unraid Community Apps templates for self-hosting [AppFlowy Cloud](https://github.com/AppFlowy-IO/AppFlowy-Cloud) — the open-source, privacy-first collaborative workspace (Notion alternative).

## Services

| Container | Image | Role |
|---|---|---|
| **AppFlowy-Postgres** | `pgvector/pgvector:pg16` | Primary database |
| **AppFlowy-Redis** | `redis:7` | Cache & pub/sub |
| **AppFlowy-MinIO** | `minio/minio` | S3-compatible object storage |
| **AppFlowy-GoTrue** | `appflowyinc/gotrue` | Authentication (JWT, OAuth, email) |
| **AppFlowy-Cloud** | `appflowyinc/appflowy_cloud` | Core backend API + WebSocket |
| **AppFlowy-Admin** | `appflowyinc/admin_frontend` | Admin web console (`/console`) |
| **AppFlowy-Web** | `appflowyinc/appflowy_web` | Browser client (`/`) |
| **AppFlowy-Worker** | `appflowyinc/appflowy_worker` | Background job processor |
| **AppFlowy-Search** | `appflowyinc/appflowy_search` | Full-text & semantic search |
| **AppFlowy-AI** _(optional)_ | `appflowyinc/appflowy_ai` | LLM chat & embeddings |
| **AppFlowy-Nginx** | `nginx:stable-alpine` | Reverse proxy (public entry point) |

## Prerequisites

### 1. Create the `appflowy` Docker network

All containers communicate on a shared custom bridge network.

**In Unraid UI:** Settings → Docker → Custom Networks → Add → name it `appflowy`.

**Or via SSH:**
```bash
docker network create appflowy
```

### 2. Download the nginx config

Before starting **AppFlowy-Nginx**, place the custom config at:

```
/mnt/user/appdata/AppFlowy-Nginx/nginx.conf
```

```bash
mkdir -p /mnt/user/appdata/AppFlowy-Nginx/ssl
curl -o /mnt/user/appdata/AppFlowy-Nginx/nginx.conf \
  https://raw.githubusercontent.com/vmalinics0/unraid-appflowy/main/nginx/nginx.conf
```

The nginx.conf in this repo uses Unraid container names (`AppFlowy-Cloud`, `AppFlowy-GoTrue`, etc.) instead of the default compose service names.

## Deploy Order

Deploy containers **in this order** (each group can be deployed in parallel):

```
1. AppFlowy-Postgres
2. AppFlowy-Redis
3. AppFlowy-MinIO
4. AppFlowy-GoTrue        (waits for Postgres)
5. AppFlowy-Cloud         (waits for GoTrue, Postgres, Redis, MinIO)
6. AppFlowy-Admin         \
   AppFlowy-Web            | (wait for AppFlowy-Cloud)
   AppFlowy-Worker         |
   AppFlowy-Search         |
   AppFlowy-AI (optional)  /
7. AppFlowy-Nginx          (deploy last)
```

## Required Configuration

Set these values consistently across all templates before starting:

| Setting | Where to set | Notes |
|---|---|---|
| `YOUR_SERVER_IP` | All templates | Your Unraid server's LAN IP or domain |
| Postgres Password | AppFlowy-Postgres, AppFlowy-GoTrue, AppFlowy-Cloud, AppFlowy-Worker, AppFlowy-Search, AppFlowy-AI | Must be identical everywhere |
| JWT Secret | AppFlowy-GoTrue, AppFlowy-Cloud, AppFlowy-Search, AppFlowy-AI | Min 32 chars; must be identical everywhere |
| MinIO credentials | AppFlowy-MinIO, AppFlowy-Cloud, AppFlowy-Worker, AppFlowy-Search, AppFlowy-AI | Access key + secret key |
| Admin email/password | AppFlowy-GoTrue | Login for `/console` |

## Accessing AppFlowy

| URL | Service |
|---|---|
| `http://YOUR_SERVER_IP/` | AppFlowy Web (browser client) |
| `http://YOUR_SERVER_IP/console` | Admin console |
| `http://YOUR_SERVER_IP/minio/` | MinIO web UI |

Native desktop/mobile clients: point them to `http://YOUR_SERVER_IP` as the server URL.

## TLS / HTTPS

1. Place your certificate and key in `/mnt/user/appdata/AppFlowy-Nginx/ssl/`:
   - `certificate.crt`
   - `private_key.key`
2. Edit `nginx.conf` to uncomment the `ssl_certificate` lines and the `listen 443 ssl;` line, and comment out `listen 80;`.
3. Update all `http://` URLs in every template to `https://`.
4. Update `APPFLOWY_WS_BASE_URL` in **AppFlowy-Web** from `ws://` to `wss://`.
5. Restart all containers.

## Submitting to Community Apps

1. Fork / create a public GitHub repository.
2. Push all files from this directory into the repo root.
3. Submit the repository URL at [https://ca.unraid.net/submit](https://ca.unraid.net/submit).
4. Run **Validate** and **Scan** in the CA submission flow.

## Upstream

- [AppFlowy-Cloud GitHub](https://github.com/AppFlowy-IO/AppFlowy-Cloud)
- [AppFlowy Cloud Discussions](https://github.com/AppFlowy-IO/AppFlowy-Cloud/discussions)
