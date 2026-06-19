# Atlassian Knowledge Base

Self-hosted RAG chat assistant for Atlassian apps (Jira, Confluence). Built by Optimizory.

---

## Prerequisites

| Requirement | Minimum | Notes |
|-------------|---------|-------|
| Docker | 24+ | |
| Docker Compose | v2 (plugin) | `docker compose`, not `docker-compose` |
| RAM | 2 GB | 4 GB recommended |
| Disk | 10 GB free | Database and audio files |
| Outbound HTTPS | Required | Anthropic API, Google GenAI, Atlassian Cloud |
| Open port | 8100 | Or 80/443 behind nginx |

---

## Getting a License Key

License keys are issued by Optimizory. Contact [support@optimizory.com](mailto:support@optimizory.com) to obtain a key for your deployment.

Once you have a key (`OTPLLK-…`), paste it in **Admin → Settings → License** after first login.

---

## Setup

### 1. Clone the repo

```bash
git clone https://github.com/Optimizory/app-atlassian-knowledge-base /opt/kb-platform
cd /opt/kb-platform
```

### 2. Generate secrets

```bash
# POSTGRES_PASSWORD
openssl rand -base64 24 | tr -d '=\n'

# JWT_SECRET
openssl rand -base64 48 | tr -d '=\n'

# FERNET_KEY  (must be url-safe base64 of exactly 32 bytes)
openssl rand -base64 32 | tr '+/' '-_'

# ADMIN_PASSWORD
openssl rand -base64 12 | tr -d '=\n'
```

Or generate all at once:

```bash
printf "POSTGRES_PASSWORD=%s\nJWT_SECRET=%s\nFERNET_KEY=%s\nADMIN_PASSWORD=%s\n" \
  "$(openssl rand -base64 24 | tr -d '=\n')" \
  "$(openssl rand -base64 48 | tr -d '=\n')" \
  "$(openssl rand -base64 32 | tr '+/' '-_' | tr -d '\n')" \
  "$(openssl rand -base64 12 | tr -d '=\n')"
```

### 3. Create .env

```bash
cp .env.example .env
```

Required variables:

| Variable | Description |
|----------|-------------|
| `POSTGRES_PASSWORD` | Generated above |
| `JWT_SECRET` | Generated above |
| `FERNET_KEY` | Generated above |
| `ADMIN_EMAIL` | Your admin login email |
| `ADMIN_PASSWORD` | Generated above |
Optional variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `APP_ENV` | `development` | Set to `production` to enable production-mode settings |
| `APP_VERSION` | `latest` | Pin a specific image tag, e.g. `1.2.0` |

### 4. Pull images and start

```bash
docker compose pull
docker compose up -d
```

Watch startup logs:

```bash
docker compose logs -f app
```

First boot runs database migrations automatically. Startup is complete when logs show:

```
Knowledge Base API started
```

Verify:

```bash
curl http://localhost:8100/health
# {"status":"ok"}
```

### 5. Log in and configure

Open `http://your-server:8100` in a browser and log in with `ADMIN_EMAIL` / `ADMIN_PASSWORD` from `.env`.

Go to **Admin → Settings** and complete the following in order:

**LLM Providers**

| Setting | Value |
|---------|-------|
| Anthropic API Key | Your key from console.anthropic.com |
| Claude Model | `claude-sonnet-4-6` |
| Default Provider | `claude` |

Click **Test LLM** before saving.

**API Credentials**

| Setting | Value |
|---------|-------|
| Confluence Base URL | `https://yourorg.atlassian.net/wiki` |
| Confluence Email | Your email used to access Atlassian marketplace data |
| Confluence API Token | https://id.atlassian.com/manage-profile/security/api-tokens |

Click **Test Confluence** to verify the connection.

**Google**

| Setting | Value |
|---------|-------|
| Gemini API Key 1 | https://aistudio.google.com/api-keys — used for embeddings |

Click **Test Gemini Embedding** to verify.

**License**

Paste your `OTPLLK-…` key (from [Getting a License Key](#getting-a-license-key)) and click **Save License Key**.

After saving the license, go to **Admin → Apps** to create your first app, attach a Confluence space, and run an initial sync.

---

## nginx + SSL

### HTTP only (testing)

```nginx
server {
    listen 80;
    server_name kb.yourdomain.com;

    client_max_body_size 20m;

    location / {
        proxy_pass         http://127.0.0.1:8100;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 120s;
    }
}
```

### Add SSL with Let's Encrypt

```bash
apt install certbot python3-certbot-nginx
certbot --nginx -d kb.yourdomain.com
```

Certbot modifies the config in place. Confirm `proxy_read_timeout 120s` is present in the resulting HTTPS server block — LLM responses can take up to 60 seconds.

---

## Updating

```bash
cd /opt/kb-platform
docker compose pull
docker compose up -d
```

Volumes `kb_postgres_data` and `audio_data` are preserved. Database migrations run automatically on startup.

To pin a version: set `APP_VERSION=1.2.0` in `.env`, then run `docker compose up -d`.

To roll back: set `APP_VERSION` to the prior tag and run `docker compose up -d`. Migrations are append-only so rolling back the application is safe.

---

## License Renewal

Contact [support@optimizory.com](mailto:support@optimizory.com) to obtain a renewed key.

Paste the new key in **Admin → Settings → License**. The old key is overwritten immediately.

License states:

| Status | Effect |
|--------|--------|
| `active` | Full access |
| `grace` | Read-only for 3 days after expiry — data visible, write operations blocked |
| `expired` | Full lockout — renew to restore access |

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| Container exits immediately | `docker compose logs app` — missing env vars or DB not yet ready |
| DB not healthy | `docker compose ps` — wait 15 s and retry; confirm `POSTGRES_PASSWORD` is set in `.env` |
| `curl http://localhost:8100/health` fails | `docker compose ps` — confirm `app` is running; check logs for migration errors |
| App not loading in browser | Confirm nginx `proxy_pass` points to `127.0.0.1:8100`; check `proxy_read_timeout 120s` |
| "No license key configured" banner | Admin → Settings → License — save the key, then reload the page |
| Confluence sync fails | Admin → Settings → Test Confluence; verify the space key matches exactly in Confluence → Space Settings |
| Embeddings 429 errors | Add more Gemini API keys in Admin → Settings → Google (Key 2, Key 3, …) |
| Clock-skew license errors | `date` on server — sync with `timedatectl set-ntp true` |
| Forgot admin password | Run the reset command below |

**Reset admin password:**

```bash
docker compose exec app python3 -c "
import asyncio, bcrypt
from database import async_session
from sqlalchemy import text

async def reset():
    h = bcrypt.hashpw(b'NewPassword123!', bcrypt.gensalt()).decode()
    async with async_session() as s:
        await s.execute(text(\"UPDATE users SET password_hash=:h WHERE email='admin@yourcompany.com'\"), {'h': h})
        await s.commit()
    print('Done')

asyncio.run(reset())
"
```

Replace `admin@yourcompany.com` and `NewPassword123!` with your values.

---

## Quick Reference

```bash
# Start
docker compose up -d

# Stop
docker compose down

# Upgrade
docker compose pull && docker compose up -d

# View logs
docker compose logs -f app

# Health check
curl http://localhost:8100/health

# Backup database
docker compose exec db pg_dump -U kb knowledge_base | gzip > kb_backup_$(date +%Y%m%d).sql.gz

# Restore database
docker compose stop app
gunzip -c kb_backup_20260619.sql.gz | docker compose exec -T db psql -U kb knowledge_base
docker compose start app

```
