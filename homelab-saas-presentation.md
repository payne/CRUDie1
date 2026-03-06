---
marp: true
theme: default
paginate: true
backgroundColor: #0f172a
color: #e2e8f0
style: |
  section {
    font-family: 'Segoe UI', sans-serif;
    padding: 40px 60px;
  }
  h1 {
    color: #f97316;
    font-size: 2em;
    border-bottom: 3px solid #f97316;
    padding-bottom: 10px;
  }
  h2 {
    color: #fb923c;
    font-size: 1.4em;
  }
  code {
    background: #1e293b;
    color: #7dd3fc;
    padding: 2px 6px;
    border-radius: 4px;
  }
  pre {
    background: #1e293b;
    border-left: 4px solid #f97316;
    border-radius: 8px;
    padding: 20px;
  }
  pre code {
    background: transparent;
    padding: 0;
    color: #e2e8f0;
  }
  pre code .hljs-attr,
  pre code .hljs-bullet {
    color: #93c5fd;
  }
  pre code .hljs-string,
  pre code .hljs-literal {
    color: #86efac;
  }
  pre code .hljs-comment {
    color: #cbd5e1;
  }
  pre code .hljs-variable,
  pre code .hljs-template-variable {
    color: #fde68a;
  }
  pre code .hljs-number {
    color: #fde68a;
  }
  pre code .hljs-keyword {
    color: #f9a8d4;
  }
  .highlight {
    color: #f97316;
    font-weight: bold;
  }
  a {
    color: #60a5fa;
  }
  table {
    border-collapse: collapse;
    width: 100%;
    margin-top: 16px;
  }
  th {
    background: #f97316;
    color: #0f172a;
    font-weight: bold;
    padding: 10px 16px;
    text-align: left;
    border: 2px solid #f97316;
  }
  td {
    background: #1e293b;
    color: #f1f5f9;
    padding: 10px 16px;
    border: 1px solid #334155;
  }
  tr:nth-child(even) td {
    background: #263548;
  }
---

# SaaS in the Homelab
## Cloudflare Tunnels + OAuth2 + Docker Compose

**Securely exposing homelab services to the Internet**
without opening firewall ports

---

# The Problem

You have a cool self-hosted app running in your homelab.
You want to access it **from anywhere** — securely.

**Traditional options are painful:**
- Open a firewall port (attack surface)
- Set up a VPN (friction for every user)
- Use a reverse proxy with TLS cert management (complexity)

**The homelab tax: security vs. accessibility**

---

# The Solution Stack

```
Internet
   |
   v
1.AppFarms.org  (Cloudflare DNS)
   |
   v
Cloudflare Edge  (zero-trust tunnel)
   |
   v
cloudflared  (docker container, no inbound ports)
   |
   v
oauth2-proxy  (Google auth gate)
   |
   v
OpenWebRX+  (homelab SDR receiver)
```

Zero open firewall ports. Auth enforced before any app traffic.

---

# Component 1: Cloudflare Tunnel

**What it does:** Creates an outbound-only encrypted tunnel from your homelab to Cloudflare's edge.

- No inbound firewall rules needed
- `cloudflared` daemon initiates the connection
- Cloudflare routes `1.appfarms.org` traffic into your network

**The key insight:** Your homelab *calls out* to Cloudflare, not the other way around.

```yaml
# cloudflared-config.yml
tunnel: appfarms-tunnel
credentials-file: /etc/cloudflared/credentials.json

ingress:
  - hostname: 1.appfarms.org
    service: http://oauth2-proxy:4180
  - service: http_status:404
```

---

# Component 2: oauth2-proxy

**What it does:** Sits between Cloudflare and your app.
Forces Google OAuth login before forwarding any request.

- Only users in `authorized_gmails.txt` get through
- Everyone else: 403
- No app changes needed — pure middleware

```
GET https://1.appfarms.org/
  -> oauth2-proxy checks session cookie
  -> Not authenticated? Redirect to Google login
  -> Authenticated but not authorized? 403
  -> Authorized? Forward to upstream app
```

Callback URL: `https://1.appfarms.org/oauth2/callback`

---

# Component 3: Docker Compose Network

Everything runs in one `docker compose` stack on a single host.

```yaml
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    depends_on: [oauth2-proxy]
    volumes:
      - ./cloudflared:/etc/cloudflared

  oauth2-proxy:
    image: quay.io/oauth2-proxy/oauth2-proxy:latest
    volumes:
      - ./authorized_gmails.txt:/authorized_gmails.txt
    environment:
      - OAUTH2_PROXY_CLIENT_ID=${GOOGLE_CLIENT_ID}
      - OAUTH2_PROXY_CLIENT_SECRET=${GOOGLE_CLIENT_SECRET}
      - OAUTH2_PROXY_COOKIE_SECRET=${COOKIE_SECRET}
      - OAUTH2_PROXY_UPSTREAM=http://openwebrx:8073

networks:
  default:
    name: proxy-net
```

---

# The Traffic Flow (End-to-End)

| Step | Where | What Happens |
|------|-------|--------------|
| 1 | Browser | User visits `1.appfarms.org` |
| 2 | Cloudflare Edge | TLS terminated, tunnel selected |
| 3 | `cloudflared` | Traffic delivered into docker network |
| 4 | `oauth2-proxy` | Session cookie checked |
| 5 | Google | OAuth2 login (if needed) |
| 6 | `oauth2-proxy` | Email checked against allowlist |
| 7 | OpenWebRX+ | App receives authenticated request |

**The app never sees an unauthenticated request.**

---

# The App: OpenWebRX+

A web-based **software-defined radio** receiver.
Lets you tune into radio signals from your browser.

- Runs on the homelab, not exposed directly
- Accessed only through the auth proxy chain
- URL stays internal: `http://openwebrx-host:8073`

**This pattern works for any homelab service:**
- Home Assistant
- Grafana
- Paperless-ngx
- Anything with a web UI

---

# Access Control: The Gmail Allowlist

Simple, auditable, file-based access control.

```
# authorized_gmails.txt
alice@gmail.com
bob@gmail.com
friend@gmail.com
```

**Audit who has accessed the app:**

```bash
docker inspect oauth2-proxy --format='{{.LogPath}}' \
  | xargs sudo cat \
  | jq 'select(.authorized == true)'
```

Add/remove users: edit the file, restart the container.

---

# What You Need to Set This Up

1. **Cloudflare account** — free tier works
   - Domain on Cloudflare DNS
   - Tunnel created via `cloudflared tunnel create`

2. **Google Cloud Console** — free
   - OAuth2 credentials (Client ID + Secret)
   - Authorized redirect URI: `https://your-domain/oauth2/callback`

3. **Docker + Docker Compose** — on any homelab machine

4. **The app** — running anywhere reachable from the compose host

**Reference implementation:** github.com/payne/appfarms-proxy

---

# Why This Architecture Works

**Security**
- Zero open ports on your router/firewall
- Authentication enforced at the edge of your stack
- App code untouched — no auth logic to implement

**Simplicity**
- One `docker compose up` deploys the whole proxy stack
- Secrets in environment variables (`.env` file)
- Cloudflare handles TLS, DDoS, and DNS

**Flexibility**
- Swap in any upstream app
- Add more tunnels/hostnames as needed
- Works behind CGNAT (no public IP required)

---

# Live Example

**`1.AppFarms.org`** — a working deployment of this pattern

Routing: `Cloudflare → cloudflared → oauth2-proxy → OpenWebRX+`

```
AppFarms.org
  |
  +-- 1.AppFarms.org  → OpenWebRX+ (SDR receiver)
  |                     [this presentation's example]
  |
  +-- *.AppFarms.org  → future homelab services
                        [same pattern, different upstreams]
```

Source code: **github.com/payne/appfarms-proxy**

---

# Summary

**The pattern:**

> Cloudflare Tunnel + oauth2-proxy + Docker Compose
> = Secure, authenticated public access to any homelab service

**No open ports. No VPN. No certificate headaches.**
Just Google login standing between the Internet and your homelab.

**Go build something and put it on the Internet safely.**
