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
| **AppFlowy-Certbot** _(optional)_ | `certbot/certbot` | Automated Let's Encrypt TLS certificates |

## Prerequisites

### 1. Create the `appflowy` Docker network

All containers communicate on a shared custom bridge network.

**Via SSH or the Unraid terminal:**
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

Deploy containers **in this order** (each group can be deployed in parallel).
The install step number is shown in each app's description in Community Apps.

```
Step 1 — AppFlowy-Postgres
Step 2 — AppFlowy-Redis
Step 3 — AppFlowy-MinIO
Step 4 — AppFlowy-GoTrue        (waits for Postgres)
Step 5 — AppFlowy-Cloud         (waits for GoTrue, Postgres, Redis, MinIO)
Step 6 — AppFlowy-Admin         \
          AppFlowy-Web            | (wait for AppFlowy-Cloud)
          AppFlowy-Worker         |
          AppFlowy-Search         |
          AppFlowy-AI (optional)  /
Step 7 — AppFlowy-Nginx          (deploy last)
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

Three nginx config files are provided; choose one:

| Config file | Use when |
|---|---|
| `nginx/nginx.conf` | HTTP only (default, no TLS) |
| `nginx/nginx-https-custom.conf` | You have your own TLS certificate |
| `nginx/nginx-letsencrypt.conf` | Automated Let's Encrypt certificates |

### Option A — User-provided certificate

1. Place your certificate and key in `/mnt/user/appdata/AppFlowy-Nginx/ssl/`:
   - `certificate.crt`
   - `private_key.key`
2. Point the AppFlowy-Nginx **nginx.conf** path to `nginx-https-custom.conf`.
3. Update all `http://` URLs in every template to `https://`.
4. Update `APPFLOWY_WS_BASE_URL` in **AppFlowy-Web** from `ws://` to `wss://`.
5. Restart all containers.

### Option B — Automated Let's Encrypt (AppFlowy-Certbot)

> Requires a public domain name with port 80 reachable from the internet.

1. Create the required host directories:
   ```bash
   mkdir -p /mnt/user/appdata/AppFlowy-Nginx/letsencrypt
   mkdir -p /mnt/user/appdata/AppFlowy-Nginx/certbot-webroot
   ```
2. Download `nginx-letsencrypt.conf` and edit it — replace `YOUR_DOMAIN` (two occurrences) with your actual domain:
   ```bash
   curl -o /mnt/user/appdata/AppFlowy-Nginx/nginx-letsencrypt.conf \
     https://raw.githubusercontent.com/vmalinics0/unraid-appflowy/main/nginx/nginx-letsencrypt.conf
   ```
3. Point the AppFlowy-Nginx **nginx.conf** path to `nginx-letsencrypt.conf` and restart it.
4. In Community Apps, install **AppFlowy-Certbot**. Edit its **Extra Parameters** field — replace `yourdomain.com` and `admin@yourdomain.com` with your real values. Start it; it will obtain the certificate and exit (this is normal).
5. Restart **AppFlowy-Nginx** to load the new certificate.
6. Update all `http://` URLs in every template to `https://` and `APPFLOWY_WS_BASE_URL` from `ws://` to `wss://`.
7. Schedule AppFlowy-Certbot to restart monthly (e.g. via the Unraid User Scripts plugin) for automatic renewal. After each renewal, restart AppFlowy-Nginx.

## Submitting to Community Apps

1. Fork / create a public GitHub repository.
2. Push all files from this directory into the repo root.
3. Submit the repository URL at [https://ca.unraid.net/submit](https://ca.unraid.net/submit).
4. Run **Validate** and **Scan** in the CA submission flow.

## Upstream

- [AppFlowy-Cloud GitHub](https://github.com/AppFlowy-IO/AppFlowy-Cloud)
- [AppFlowy Cloud Discussions](https://github.com/AppFlowy-IO/AppFlowy-Cloud/discussions)
