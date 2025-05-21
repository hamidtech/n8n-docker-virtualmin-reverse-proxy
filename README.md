# n8n + Docker + Virtualmin + Cloudflare
Full Reverse‑Proxy Setup with WebSockets, OAuth & Telegram Support  
*Last updated: May 2025 — by *Hamid Jamali***

---

## Table of Contents
1. [Prerequisites](#prerequisites)  
2. [Run n8n in Docker](#runn8nindocker)  
3. [Configure Apache (Virtualmin)](#configureapachevirtualmin)  
4. [Configure Cloudflare](#configurecloudflare)  
5. [Testing & Troubleshooting](#testing--troubleshooting)  
6. [Telegram & OAuth Notes](#telegram--oauth-notes)  
7. [Maintenance & Safety Tips](#maintenance--safety-tips)  
8. [License](#license)

---

## Prerequisites
| Item | Notes |
|------|-------|
| Ubuntu 22.04 / 24.04 | or any distro with **Apache ≥ 2.4.5** |
| Docker Engine | `sudo apt install docker.io -y` |
| Virtualmin / Webmin | to manage Apache vhosts |
| Cloudflare (optional but recommended) | proxy + free SSL |

> **Open firewall ports:** `22`, `80`, `443`. The internal n8n port (`5678`) should **not** be exposed to the Internet.

---

## Run n8n in Docker
```bash
docker run -d --name n8n \
  -p 5678:5678 \
  -e N8N_PORT=5678 \
  -e N8N_PROTOCOL=https \
  -e N8N_HOST=<your-domain> \
  -e WEBHOOK_URL=https://<your-domain>/ \
  -e WEBHOOK_TUNNEL_URL=https://<your-domain>/ \
  -e VUE_APP_URL_BASE_API=https://<your-domain> \
  docker.n8n.io/n8nio/n8n
```
### Why these ENV vars?
| Variable | Purpose |
|----------|---------|
| `N8N_PORT` | Container’s internal listening port |
| `N8N_PROTOCOL=https` | Forces https in generated URLs |
| `N8N_HOST` | Your public domain (no port) |
| `WEBHOOK_*` | Generates valid webhook URLs for Telegram etc. |
| `VUE_APP_URL_BASE_API` | Front‑end knows API base path |

---

## Configure Apache (Virtualmin)
File: `/etc/apache2/sites-enabled/<your-domain>.conf`

### 1 – enable required modules
```bash
sudo a2enmod proxy proxy_http proxy_wstunnel headers
sudo systemctl restart apache2
```

### 2 – HTTPS vhost (`:443`)
```apache
ProxyPreserveHost On
RequestHeader set Connection "upgrade"
RequestHeader set Upgrade "websocket"

# WebSocket route – MUST come first
ProxyPass "/rest/push" "ws://localhost:5678/rest/push" upgrade=websocket
ProxyPassReverse "/rest/push" "ws://localhost:5678/rest/push"

# All other traffic
ProxyPass / http://localhost:5678/
ProxyPassReverse / http://localhost:5678/

# Let’s Encrypt challenge
ProxyPass /.well-known !
```

### 3 – HTTP vhost (`:80`)
```apache
RewriteEngine On
RewriteRule ^/?(.*)$ https://%{HTTP_HOST}/$1 [R=301,L]
```

> **DO NOT** swap the order of `ProxyPass` lines — WebSocket must be above `/`, otherwise connections drop with `code 1006`.

---

## Configure Cloudflare
| Setting | Value |
|---------|-------|
| Proxy Status | ☁️ **Orange Cloud** (ON) |
| SSL/TLS Mode | **Full** or **Full (Strict)** |
| WebSockets | **Enabled** |

> **DO NOT** use **Flexible SSL** — it causes redirect loops.

---

## Testing & Troubleshooting
| Check | Command |
|-------|---------|
| Container running | `docker ps` / `docker logs n8n` |
| Local reachability | `curl -i http://localhost:5678` (*expect 200*) |
| Apache error log | `tail -f /var/log/virtualmin/<your-domain>_error_log` |
| WebSocket status | Browser DevTools → Network → **WS** → `/rest/push` should show **101 Switching Protocols** |

Typical errors & fixes  
* **Connection refused** → wrong port in Apache rules, or container down.  
* **code 1006** loop → WebSocket rule below `/`, or `proxy_wstunnel` not enabled.

---

## Telegram & OAuth Notes
* Telegram accepts webhooks **only on ports 80 / 443 / 88 / 8443**. Because Apache serves on 443, this setup is compliant.
* Use this single Redirect URL for any OAuth provider (Twitter/X, Google, …):
  ```
  https://<your-domain>/rest/oauth2-credential/callback
  ```
* Never include `:5678` (or any other port) in public URLs.

---

## Maintenance & Safety Tips
| Do | Don’t |
|----|-------|
| `docker pull docker.n8n.io/n8nio/n8n` then restart to update | Expose port 5678 to the Internet |
| Mount a volume for `/home/node/.n8n` to persist data | Change Apache config without `apachectl configtest` |
| Rotate/clean logs regularly | Enable Cloudflare *Flexible* SSL |
| Take snapshots/backups before major upgrades | Re‑order ProxyPass lines randomly |

---

## License
MIT © Hamid Jamali
