# Atlassian Knowledge Base

Self-hosted RAG chat assistant for Atlassian apps (Jira, Confluence). Built by Optimizory.

---

## Prerequisites

| Requirement | Minimum | Notes |
|-------------|---------|-------|
| Docker | 24+ | |
| Docker Compose | v2 (plugin) | `docker compose`, not `docker-compose` |
| Python 3 | 3.8+ | For secret generation and license key management |
| RAM | 2 GB | 4 GB recommended |
| Disk | 10 GB free | Database and audio files |
| Outbound HTTPS | Required | Anthropic API, Google GenAI, Atlassian Cloud |
| Open port | 8100 | Or 80/443 behind nginx |

---

## Getting a License Key

License keys are time-bound HMAC tokens generated with your `LICENSE_HMAC_SECRET`. Generate one after you have created `.env`:

```bash
export LICENSE_HMAC_SECRET=$(grep LICENSE_HMAC_SECRET .env | cut -d= -f2)

python3 tools/generate_license.py \
  --email admin@yourcompany.com \
  --days 365 \
  --verify
```

Output:

```
OTPLLK-eyJlbWFpbC...

Validation: {'valid': True, 'status': 'active', 'email': 'admin@yourcompany.com', 'days_remaining': 365}
```

Copy the `OTPLLK-…` key. You will paste it in the Admin UI after first login.

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
python3 -c "import secrets; print(secrets.token_urlsafe(32))"

# JWT_SECRET
python3 -c "import secrets; print(secrets.token_urlsafe(64))"

# FERNET_KEY
python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"

# ADMIN_PASSWORD
python3 -c "import secrets; print(secrets.token_urlsafe(16))"

# LICENSE_HMAC_SECRET
python3 -c "import secrets; print(secrets.token_urlsafe(32))"
```

Or generate all at once:

```bash
python3 - <<'EOF'
import secrets
from cryptography.fernet import Fernet

print("POSTGRES_PASSWORD=   " + secrets.token_urlsafe(32))
print("JWT_SECRET=          " + secrets.token_urlsafe(64))
print("FERNET_KEY=          " + Fernet.generate_key().decode())
print("ADMIN_PASSWORD=      " + secrets.token_urlsafe(16))
print("LICENSE_HMAC_SECRET= " + secrets.token_urlsafe(32))
EOF
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
| `LICENSE_HMAC_SECRET` | Generated above — back up separately; losing it invalidates all license keys |

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
| Gemini API Key 1 | Your key from aistudio.google.com — used for embeddings |
| Confluence Base URL | `https://yourorg.atlassian.net/wiki` |
| Confluence Email | Service account email |
| Confluence API Token | Token from id.atlassian.com → Security → API tokens |

Click **Test Confluence** and **Test Gemini Embedding** to verify each connection.

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

Generate a new key with an updated `--days` value:

```bash
python3 tools/generate_license.py \
  --email admin@yourcompany.com \
  --days 365
```

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
| Embeddings 429 errors | Add more Gemini API keys in Admin → Settings → API Credentials (Key 2, Key 3, …) |
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

# Generate license key
python3 tools/generate_license.py --email admin@yourcompany.com --days 365
```
