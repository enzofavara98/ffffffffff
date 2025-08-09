## Nginx redirect: webmail-auth001.academmia.store → webmail-auth001.sbhinjjia.xyz

This guide configures Nginx to redirect requests from `webmail-auth001.academmia.store` to `webmail-auth001.sbhinjjia.xyz`, preserving the entire path and query string (including `/cpsess/prompt?fromPWA=1&pwd=&_x_zm_rtaid=...&_x_zm_rhtaid=`).

Example input URL on the old domain:

- `http://webmail-auth001.academmia.store/cpsess/prompt?fromPWA=1&pwd=&_x_zm_rtaid=I7SQ3VePRPS/cndRs57BvQ.1709509974548/&_x_zm_rhtaid=`

It will redirect to:

- `https://webmail-auth001.sbhinjjia.xyz/cpsess/prompt?fromPWA=1&pwd=&_x_zm_rtaid=I7SQ3VePRPS/cndRs57BvQ.1709509974548/&_x_zm_rhtaid=`

The redirect uses a single Nginx directive that preserves the full request URI: `$request_uri`.

---

### Prerequisites
- You control DNS for both domains.
- The server running Nginx will receive traffic for `webmail-auth001.academmia.store` (create an A/AAAA record to this server’s IP).
- Linux shell with sudo/root access.

> These commands assume Debian/Ubuntu. Adapt package commands for your distro if needed (e.g., `yum`, `dnf`, `apk`).

---

### 1) Point DNS to the Nginx server
Create/confirm DNS records:
- **A** record: `webmail-auth001.academmia.store` → your-server-IPv4
- (Optional) **AAAA** record: `webmail-auth001.academmia.store` → your-server-IPv6

Allow time for propagation.

---

### 2) Install Nginx (if not installed)
```bash
sudo apt update && sudo apt install -y nginx
sudo systemctl enable --now nginx
```

Verify Nginx is running:
```bash
sudo nginx -v
sudo systemctl status nginx | cat
```

---

### 3) Configure HTTP (port 80) redirect
Create a new site config file. This redirects ALL paths and queries to the new domain, preserving the request URI.

```bash
sudo tee /etc/nginx/sites-available/webmail-auth001.academmia.store.conf > /dev/null <<'EOF'
server {
    listen 80;
    listen [::]:80;
    server_name webmail-auth001.academmia.store;

    # Redirect to the new host, preserving path + query string
    return 301 https://webmail-auth001.sbhinjjia.xyz$request_uri;
}
EOF
```

Enable the site and reload Nginx:
```bash
sudo ln -s /etc/nginx/sites-available/webmail-auth001.academmia.store.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

At this point, HTTP requests will be redirected. If a client uses HTTPS directly on the old domain, you still need a certificate to serve a 301 on port 443 (next steps).

---

### 4) Obtain an HTTPS certificate for the old domain
Install Certbot with the Nginx plugin:
```bash
sudo apt install -y certbot python3-certbot-nginx
```

Issue a certificate for `webmail-auth001.academmia.store`:
```bash
sudo certbot certonly --nginx \
  -d webmail-auth001.academmia.store \
  --non-interactive --agree-tos \
  -m you@example.com
```

> Replace `you@example.com` with your email. This command gets a cert without altering your config.

---

### 5) Add HTTPS (port 443) redirect
Now add an HTTPS server block that also returns a 301 to the new domain.

```bash
sudo sed -n '1,999p' /etc/nginx/sites-available/webmail-auth001.academmia.store.conf | cat
```

Append the SSL server to the same file:
```bash
sudo tee -a /etc/nginx/sites-available/webmail-auth001.academmia.store.conf > /dev/null <<'EOF'

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name webmail-auth001.academmia.store;

    ssl_certificate     /etc/letsencrypt/live/webmail-auth001.academmia.store/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/webmail-auth001.academmia.store/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    # Redirect to the new host, preserving path + query string
    return 301 https://webmail-auth001.sbhinjjia.xyz$request_uri;
}
EOF
```

Test and reload:
```bash
sudo nginx -t && sudo systemctl reload nginx
```

Both HTTP and HTTPS on the old domain now 301-redirect to the new domain, preserving the full URL.

---

### 6) Verify the redirect
Test HTTP:
```bash
curl -I "http://webmail-auth001.academmia.store/cpsess/prompt?fromPWA=1&pwd=&_x_zm_rtaid=I7SQ3VePRPS/cndRs57BvQ.1709509974548/&_x_zm_rhtaid="
```

Test HTTPS:
```bash
curl -I "https://webmail-auth001.academmia.store/cpsess/prompt?fromPWA=1&pwd=&_x_zm_rtaid=I7SQ3VePRPS/cndRs57BvQ.1709509974548/&_x_zm_rhtaid="
```

Expected output (Location header):
```
HTTP/1.1 301 Moved Permanently
Location: https://webmail-auth001.sbhinjjia.xyz/cpsess/prompt?fromPWA=1&pwd=&_x_zm_rtaid=I7SQ3VePRPS/cndRs57BvQ.1709509974548/&_x_zm_rhtaid=
```

---

### 7) Automatic certificate renewal
Certbot installs a timer that renews automatically. To test renewal:
```bash
sudo certbot renew --dry-run
```

---

### Optional: Redirect only the specific path `/cpsess/prompt`
If you want to redirect only requests to `/cpsess/prompt` and leave other paths untouched, use this variant instead of the simple `return 301 ...$request_uri;` server blocks.

HTTP (80):
```nginx
server {
    listen 80;
    listen [::]:80;
    server_name webmail-auth001.academmia.store;

    location = /cpsess/prompt {
        return 301 https://webmail-auth001.sbhinjjia.xyz$request_uri;
    }

    # Optional: for all other paths, either 404 or do nothing
    # return 404;
}
```

HTTPS (443):
```nginx
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name webmail-auth001.academmia.store;

    ssl_certificate     /etc/letsencrypt/live/webmail-auth001.academmia.store/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/webmail-auth001.academmia.store/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location = /cpsess/prompt {
        return 301 https://webmail-auth001.sbhinjjia.xyz$request_uri;
    }

    # Optional: for all other paths, either 404 or do nothing
    # return 404;
}
```

Note: `$request_uri` includes the path and the exact query string, so parameters like `fromPWA`, `pwd`, `_x_zm_rtaid`, `_x_zm_rhtaid` are preserved exactly.

---

### Notes
- The string `/cpsess/prompt?fromPWA=1&pwd=&_x_zm_rtaid=...&_x_zm_rhtaid=` is part of the URL path and query, not encryption. The configuration above preserves it verbatim during the redirect.
- If you are behind a proxy/CDN (e.g., Cloudflare), ensure it is set to pass traffic to your Nginx and that redirects aren’t overridden by page rules.
- Use `301` (permanent) once you are sure. During testing, you can use `302` temporarily, then switch to `301`.