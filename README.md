# n8n + Docker Compose + Virtualmin + Cloudflare
Full Reverse‑Proxy Setup with WebSockets, OAuth & Telegram Support  
*Last updated: December 2025 — by *Hamid Jamali***

---

# n8n Production Setup — Docker Compose + Apache Reverse Proxy + Cloudflare  
A secure, scalable, and upgrade-safe deployment of n8n using Docker Compose, Apache, and Cloudflare.

---

## 📌 Overview
This repository provides a production-ready setup for hosting **n8n** behind:

- Docker Compose  
- Apache Reverse Proxy  
- Cloudflare Proxy (optional but recommended)

It includes correct handling for:

- WebSockets  
- OAuth Redirect URLs  
- Telegram Webhooks  
- HTTPS Enforcement  
- Persistent data volumes  

---

## 📁 Folder Structure

````

/opt/n8n/
│── docker-compose.yml
│── .env
│── data/              # persistent n8n database + config
└── local-files/       # binary uploads, attachments, temp files

````

---

## ✅ Prerequisites

| Component | Notes |
|----------|-------|
| Docker Engine | `sudo apt install docker.io -y` |
| Docker Compose v2+ | Included with modern Docker |
| Apache ≥ 2.4.5 | Required for reverse proxy |
| Cloudflare (optional) | SSL termination + protection |
| Ports | `22`, `80`, `443` must be open |

⚠️ **Do NOT expose port 5678 publicly**.  
n8n must always run behind Apache or another reverse proxy.

---

## 🚀 Installation

```bash
sudo mkdir -p /opt/n8n/{data,local-files}
sudo chown -R 1000:1000 /opt/n8n
cd /opt/n8n
docker compose up -d
```

Visit:

```
https://your-domain.com
```

---

## 🔧 Environment Configuration

Environment variables are defined in `.env`.

Key variables explained:

| Variable               | Purpose                                  |
| ---------------------- | ---------------------------------------- |
| `N8N_HOST`             | Public domain (no port)                  |
| `N8N_PROTOCOL`         | Must be `https` when using reverse proxy |
| `WEBHOOK_URL`          | External webhook address                 |
| `GENERIC_TIMEZONE`     | Local timezone                           |
| `N8N_BINARY_DATA_MODE` | Store uploads on filesystem              |
| `N8N_ENCRYPTION_KEY`   | Required for credential encryption       |

A `.env.example` file is provided in this repo.

---

## 🐳 Docker Compose Setup

* Automatically restarts on failure
* Persists workflows + database in `data/`
* Stores binary assets in `local-files/`

See `docker-compose.yml` in this repo.

---

## 🌐 Apache Reverse Proxy Configuration

Enable modules:

```bash
sudo a2enmod proxy proxy_http proxy_wstunnel headers rewrite
sudo systemctl restart apache2
```

Use the provided file `apache-n8n.conf`:

```apache
ProxyPreserveHost On
RequestHeader set Connection "upgrade"
RequestHeader set Upgrade "websocket"

# WebSocket route — MUST come first
ProxyPass "/rest/push" "ws://localhost:5678/rest/push" upgrade=websocket
ProxyPassReverse "/rest/push" "ws://localhost:5678/rest/push"

# Main traffic
ProxyPass "/" "http://localhost:5678/"
ProxyPassReverse "/" "http://localhost:5678/"

# Allow Let's Encrypt
ProxyPass "/.well-known/" "!"
```

HTTP → HTTPS redirect:

```apache
RewriteEngine On
RewriteRule ^/?(.*)$ https://%{HTTP_HOST}/$1 [R=301,L]
```

---

## ☁️ Cloudflare Configuration

| Setting              | Value                 |
| -------------------- | --------------------- |
| Proxy (Orange Cloud) | ON                    |
| SSL Mode             | Full or Full (Strict) |
| WebSockets           | Enabled               |
| Always Use HTTPS     | Enabled               |

⚠️ **Do NOT use Flexible SSL** → causes redirect loops.

---

## 🧪 Testing

### Check container:

```bash
docker compose ps
docker compose logs -f
```

### Check local service:

```bash
curl -I http://localhost:5678
```

### Check WebSocket:

Browser → DevTools → Network → WS → `/rest/push`

Status must be:

```
101 Switching Protocols
```

---

## 🔐 OAuth & Telegram Notes

### OAuth callback URL:

```
https://your-domain.com/rest/oauth2-credential/callback
```

### Telegram webhook ports allowed:

`443`, `80`, `88`, `8443`

This setup supports Telegram natively.

---

## 🔄 Updating n8n

```bash
cd /opt/n8n
docker compose pull
docker compose down
docker compose up -d
```

All workflows and data remain safe.

---

## 🛡 Maintenance Checklist

✔ Back up `/opt/n8n/data/` weekly
✔ Back up `.env`
✔ Run `apachectl configtest` before enabling config
✔ Never expose port `5678`
✔ Use Cloudflare Full SSL


---
## License
MIT © Hamid Jamali
