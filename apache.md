# HTTPS VirtualHost (Reverse Proxy for n8n)
```
<VirtualHost *:443>
    ServerName your-domain.com

    ProxyPreserveHost On
    RequestHeader set Connection "upgrade"
    RequestHeader set Upgrade "websocket"

    # --- WebSocket endpoint MUST come first ---
    ProxyPass "/rest/push" "ws://localhost:5678/rest/push" upgrade=websocket
    ProxyPassReverse "/rest/push" "ws://localhost:5678/rest/push"

    # --- Main traffic ---
    ProxyPass "/" "http://localhost:5678/"
    ProxyPassReverse "/" "http://localhost:5678/"

    # Allow Let's Encrypt
    ProxyPass "/.well-known/" "!"
</VirtualHost>
```
# HTTP → HTTPS redirect
```
<VirtualHost *:80>
    ServerName your-domain.com
    RewriteEngine On
    RewriteRule ^/?(.*)$ https://%{HTTP_HOST}/$1 [R=301,L]
</VirtualHost>
```
