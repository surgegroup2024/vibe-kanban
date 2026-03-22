# Remote service

The `remote` crate contains the implementation of the Vibe Kanban hosted API.

## Prerequisites

Create a `.env.remote` file in `crates/remote/` (this matches `pnpm run remote:dev`):

```env
# Required — generate with: openssl rand -base64 48
VIBEKANBAN_REMOTE_JWT_SECRET=your_base64_encoded_secret

# Required — password for the electric_sync database role used by ElectricSQL
ELECTRIC_ROLE_PASSWORD=your_secure_password

# OAuth — at least one provider (GitHub or Google) must be configured
GITHUB_OAUTH_CLIENT_ID=your_github_web_app_client_id
GITHUB_OAUTH_CLIENT_SECRET=your_github_web_app_client_secret
GOOGLE_OAUTH_CLIENT_ID=
GOOGLE_OAUTH_CLIENT_SECRET=

# Relay (required for tunnel/relay features)
# For local HTTPS via Caddy on :3001:
VITE_RELAY_API_BASE_URL=https://relay.localhost:3001

# Optional — enables Virtuoso Message List license for remote web UI
VITE_PUBLIC_REACT_VIRTUOSO_LICENSE_KEY=

# Optional — leave empty to disable invitation emails
LOOPS_EMAIL_API_KEY=
```

Generate `VIBEKANBAN_REMOTE_JWT_SECRET` once using `openssl rand -base64 48` and copy the value into `.env.remote`.

## Run the stack locally

From the repo root:

```bash
pnpm run remote:dev
```

Equivalent manual command:

```bash
cd crates/remote
docker compose --env-file .env.remote -f docker-compose.yml up --build
```

This starts PostgreSQL, ElectricSQL, the Remote Server, and the Relay Server.

- Remote web UI/API: `https://localhost:3001` (via Caddy) or `http://localhost:3000` (direct)
- Relay API: `http://localhost:8082`
- Postgres: `postgres://remote:remote@localhost:5433/remote`

## Run Vibe Kanban

To connect the desktop client to your local remote server (without relay/tunnel):

```bash
export VK_SHARED_API_BASE=https://localhost:3001

pnpm run dev
```

## Local HTTPS with Caddy

The stack defaults to `https://localhost:3001` as its public URL. Use [Caddy](https://caddyserver.com) as a reverse proxy to terminate TLS — it automatically provisions a locally-trusted certificate for `localhost`.

### 1. Install Caddy

```bash
# macOS
brew install caddy

# Debian/Ubuntu
sudo apt install caddy
```

### 2. Create a Caddyfile

Create a `Caddyfile` in the repository root:

```text
localhost:3001, relay.localhost:3001, *.relay.localhost:3001 {
    tls internal

    @relay host relay.localhost *.relay.localhost
    handle @relay {
        reverse_proxy 127.0.0.1:8082
    }

    @app expression `{http.request.host} == "localhost:3001" || {http.request.host} == "localhost"`
    handle @app {
        reverse_proxy 127.0.0.1:3000
    }

    respond "not found" 404
}
```

### 3. Update OAuth callback URLs

Update your OAuth application to use `https://localhost:3001`:

- **GitHub**: `https://localhost:3001/v1/oauth/github/callback`
- **Google**: `https://localhost:3001/v1/oauth/google/callback`

### 4. Start everything

Start Docker services as usual, then start Caddy in a separate terminal:

```bash
# Terminal 1 — start the stack
pnpm run remote:dev

# Terminal 2 — start Caddy (from repo root)
caddy run --config Caddyfile
```

The first time Caddy runs it installs a local CA certificate — you may be prompted for your password.

Open **https://localhost:3001** in your browser.

> **Tip:** To use plain HTTP instead (no Caddy), set `PUBLIC_BASE_URL=http://localhost:3000` in your `.env.remote`.

## Run desktop with relay tunnel (optional)

To test relay/tunnel mode end-to-end:

```bash
export VK_SHARED_API_BASE=https://localhost:3001
export VK_SHARED_RELAY_API_BASE=https://relay.localhost:3001

pnpm run dev
```

Quick checks:

```bash
curl -sk https://localhost:3001/v1/health
curl -sk https://relay.localhost:3001/health
```

If `https://relay.localhost:3001/health` returns the remote frontend HTML instead of `{"status":"ok"}`, your Caddy host routing is incorrect.

---

## Production Deployment on EC2 (Ubuntu) with a Custom Domain

This section covers deploying to an AWS EC2 Ubuntu instance behind a custom domain (example: `vibekanban.stockfortress.com`) with HTTPS via Nginx + Let's Encrypt.

### Architecture overview

```
Internet
   │
   ▼ 443/80 (HTTPS/HTTP)
 Nginx  ──────────────────────────────────────────
   │                                               │
   ├─ /          → remote-server  (127.0.0.1:3000) │  Docker Compose
   ├─ /relay     → relay-server   (127.0.0.1:8082) │  (all internal)
   └─ /blobs/    → azurite        (127.0.0.1:10000)│
```

All backend services bind to `127.0.0.1` only. Nginx is the single public entry point.

---

### Step 1 — DNS

In your domain registrar or Route 53, add an **A record**:

```
vibekanban.stockfortress.com  →  <EC2_PUBLIC_IP>   TTL 300
```

Wait for DNS propagation before proceeding to step 4.

---

### Step 2 — EC2 Security Group inbound rules

| Port | Source | Purpose |
|------|--------|---------|
| 22 | Your IP | SSH |
| 80 | 0.0.0.0/0 | HTTP (Nginx — redirects to HTTPS) |
| 443 | 0.0.0.0/0 | HTTPS (all app traffic via Nginx) |

> **Close / do not open** ports 3000, 8082, 5433, 10000 — Nginx proxies them internally.

---

### Step 3 — Install Nginx and Certbot

```bash
sudo apt update
sudo apt install -y nginx certbot python3-certbot-nginx
```

---

### Step 4 — Configure Nginx

```bash
sudo nano /etc/nginx/sites-available/vibekanban
```

Paste:

```nginx
server {
    listen 80;
    server_name vibekanban.stockfortress.com;

    # Main app
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Relay WebSocket (VITE_RELAY_API_BASE_URL must end with /relay)
    location /relay {
        proxy_pass http://127.0.0.1:8082;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }

    # Azurite blob storage (AZURE_STORAGE_PUBLIC_ENDPOINT_URL must end with /blobs)
    location /blobs/ {
        proxy_pass http://127.0.0.1:10000/devstoreaccount1/;
        proxy_set_header Host 127.0.0.1:10000;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Enable and test:

```bash
sudo ln -s /etc/nginx/sites-available/vibekanban /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

### Step 5 — Obtain SSL certificate (Let's Encrypt)

```bash
sudo certbot --nginx -d vibekanban.stockfortress.com
```

Certbot will automatically update the Nginx config to add HTTPS and redirect HTTP → HTTPS. Auto-renewal is configured by default (verify with `sudo systemctl status certbot.timer`).

---

### Step 6 — Configure `.env.remote`

Create or update `crates/remote/.env.remote`:

```env
# Required — generate with: openssl rand -base64 48
VIBEKANBAN_REMOTE_JWT_SECRET=<your-strong-random-secret>

# Optional — password for the electric_sync DB role
ELECTRIC_ROLE_PASSWORD=

# OAuth — configure at least one provider (GitHub or Google)
GITHUB_OAUTH_CLIENT_ID=<your-github-client-id>
GITHUB_OAUTH_CLIENT_SECRET=<your-github-client-secret>
GOOGLE_OAUTH_CLIENT_ID=
GOOGLE_OAUTH_CLIENT_SECRET=

# Relay — baked into the frontend at Docker build time; must be the public HTTPS URL
VITE_RELAY_API_BASE_URL=https://vibekanban.stockfortress.com/relay

# Optional — leave empty to disable invitation emails
LOOPS_EMAIL_API_KEY=

# Public URLs — Nginx proxies everything through port 443
PUBLIC_BASE_URL=https://vibekanban.stockfortress.com
AZURE_STORAGE_PUBLIC_ENDPOINT_URL=https://vibekanban.stockfortress.com/blobs
```

> **Important:** `VITE_RELAY_API_BASE_URL` is baked into the frontend during `docker build`. You must rebuild the image whenever you change it.

---

### Step 7 — Update OAuth callback URLs

| Provider | Callback URL |
|----------|-------------|
| GitHub | `https://vibekanban.stockfortress.com/v1/oauth/github/callback` |
| Google | `https://vibekanban.stockfortress.com/v1/oauth/google/callback` |

Update these in your GitHub OAuth App settings and/or Google Cloud Console.

---

### Step 8 — Build and start the stack

```bash
cd ~/vibe-kanban/crates/remote   # path to the repo on the EC2 instance

# Full rebuild required when VITE_RELAY_API_BASE_URL changes
docker compose --env-file .env.remote down
docker compose --env-file .env.remote build --no-cache remote-server
docker compose --env-file .env.remote up -d
```

For subsequent restarts without env changes:

```bash
docker compose --env-file .env.remote up -d
```

---

### Step 9 — Verify

```bash
# App health
curl -s https://vibekanban.stockfortress.com/v1/health

# Relay health
curl -s https://vibekanban.stockfortress.com/relay/health
```

Both should return `{"status":"ok"}` (or similar JSON).

---

### Updating / redeploying

```bash
cd ~/vibe-kanban
git pull
cd crates/remote
docker compose --env-file .env.remote down
docker compose --env-file .env.remote build --no-cache remote-server
docker compose --env-file .env.remote up -d
```

---

### docker-compose.yml key settings for EC2

| Setting | EC2 value | Reason |
|---------|-----------|--------|
| `remote-db` port | `127.0.0.1:5433:5432` | Block public Postgres access |
| `azurite` port | `0.0.0.0:10000:10000` | Nginx proxies it; bind all so Docker networking works |
| `REMOTE_SERVER_PORTS` | `0.0.0.0:3000:8081` | Nginx reaches it on localhost |
| `relay-server` port | `0.0.0.0:8082:8082` | Nginx reaches it on localhost |
| `SERVER_PUBLIC_BASE_URL` | `https://vibekanban.stockfortress.com` | Set via `PUBLIC_BASE_URL` |
