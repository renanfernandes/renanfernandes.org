---
title: "Protecting Home Assistant with Caddy and Microsoft Entra ID"
date: 2025-09-29T21:45:00-04:00
draft: false
toc: true
tags: ["home-assistant", "caddy", "entra-id", "reverse-proxy", "oauth2-proxy", "diy"]
---

In my never-ending quest to clean up my home lab, I finally tackled something that had been bugging me for a while: **how to safely expose Home Assistant to the outside world without sketchy logins or self-signed cert warnings.**  

So first things first: I had a couple of options for the reverse proxy layer — the usual suspects like nginx or Traefik — but I went with **Caddy**. Why? Honestly, it’s just easier. The config syntax is clean, it automatically does the right thing most of the time, and the community docs/examples are great. With nginx I always feel like I’m hunting down one more `proxy_set_header` line. With Caddy, it just... works.  

And the reverse proxy itself? That’s not just about pretty URLs. It gives me a hard security boundary. I can enforce **Microsoft login + MFA** before anyone touches my dashboards, and I can keep strong segmentation between my **Main VLAN** and the more vulnerable **IoT VLAN**. Even if a device on the IoT side got popped, the reverse proxy means access to critical services is still tightly gated and logged.  

So this weekend’s project was wiring up Caddy in front of Home Assistant and hooking it to **Microsoft Entra ID** for authentication. Now before I can even see my dashboard, I go through a Microsoft login.  

The best part? It’s all running on my little Docker server, using the free tier of Entra and DigiCert certs I already had. No extra subscriptions, no cloud lock-in — just a rock-solid reverse proxy with SSO that feels like something you’d expect in production.  

Next on my list? Forwarding all of these logs into **Microsoft Sentinel** so I can start building detections and alerts. But that’s a project for another weekend. 😉

---

## Remind me again, why do this?

- Centralized **reverse proxy**: friendly URLs like `https://ha.domain.com`.
- **TLS with DigiCert** certs (no more browser warnings!!!!!).
- **Authentication with Entra**: Microsoft login + MFA before anyone can hit my dashboards.
- Works from **inside or outside** my network (with UDM Pro port-forwarding or VPN).

---

## Architecture

```
Browser → Caddy (TLS) → oauth2-proxy (OIDC auth) → Home Assistant
                          ↑
                     Entra ID login
```

---

## Prerequisites

- A Linux box running Docker (mine is `Srv-Core-Dockerland` at `10.0.0.5`).
- Home Assistant reachable at `10.0.1.2:8123`.
- DigiCert certs for `ha.domain.com` (wildcard works too).
- A domain (e.g. `domain.com`) with internal DNS (Pi-hole/Unbound) and external DNS pointing to your UDM Pro.

---

## Step 1 — Entra ID App Registration

1. Go to [Azure Portal](https://portal.azure.com) → **Entra ID** → *App registrations* → *New registration*.
2. Name: `HomeLab Reverse Proxy`.
3. Accounts: *this org only*.
4. Redirect URI:  
   ```
   https://ha.domain.com/oauth2/callback
   ```
5. Save → copy **Application (client) ID** and **Directory (tenant) ID**.
6. Certificates & secrets → create a **client secret** (save the value).
7. Authentication → enable **ID tokens** + **Access tokens**.

---

## Step 2 — Docker Compose

`/srv/docker/docker-compose.yml`:

```yaml
version: "3.8"

services:
  caddy:
    image: caddy:latest
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      - ./caddy/data:/data
      - ./caddy/config:/config
      - ./certs:/certs:ro
    networks: [rpnet]

  oauth2-proxy:
    image: quay.io/oauth2-proxy/oauth2-proxy:v7.6.0
    container_name: oauth2-proxy
    restart: unless-stopped
    env_file: ./oauth2-proxy/env
    networks: [rpnet]

networks:
  rpnet:
    driver: bridge
```

---

## Step 3 — oauth2-proxy configuration

`/srv/docker/oauth2-proxy/env`:

```env
OAUTH2_PROXY_PROVIDER=oidc
OAUTH2_PROXY_OIDC_ISSUER_URL=https://login.microsoftonline.com/<TENANT_ID>/v2.0
OAUTH2_PROXY_CLIENT_ID=<APP_CLIENT_ID>
OAUTH2_PROXY_CLIENT_SECRET=<APP_CLIENT_SECRET>
OAUTH2_PROXY_COOKIE_SECRET=<BASE64_32>   # openssl rand -base64 32
OAUTH2_PROXY_REDIRECT_URL=https://ha.domain.com/oauth2/callback
OAUTH2_PROXY_UPSTREAMS=http://10.0.1.2:8123
OAUTH2_PROXY_COOKIE_SECURE=true
OAUTH2_PROXY_COOKIE_SAMESITE=lax
OAUTH2_PROXY_COOKIE_DOMAIN=ha.domain.com
OAUTH2_PROXY_WHITELIST_DOMAINS=.domain.com
OAUTH2_PROXY_SET_XAUTHREQUEST=true
OAUTH2_PROXY_PASS_ACCESS_TOKEN=true
OAUTH2_PROXY_SCOPE=openid profile email
OAUTH2_PROXY_SKIP_PROVIDER_BUTTON=true
```

⚠️ **Gotcha**: The cookie secret must be exactly 16, 24, or 32 bytes. Don’t add quotes or inline comments in the env file. A wrong line here caused me to spend an hour debugging errors...

---

## Step 4 — Caddy configuration

`/srv/docker/caddy/Caddyfile`:

```caddyfile
ha.domain.com {
  tls /certs/ha.crt /certs/ha.key

  reverse_proxy oauth2-proxy:4180 {
    header_up Host {host}
    header_up X-Real-IP {remote_host}
    header_up X-Forwarded-For {remote_host}
    header_up X-Forwarded-Proto {scheme}
  }
}
```

---

## Step 5 — Home Assistant settings

Now you need to tell HA to trust the proxy. Edit the `configuration.yaml` file and add the following lines (ip address of your proxy will vary of course):

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 10.0.0.5          # Srv-Core-Dockerland
    - 172.16.0.0/12     # Docker bridge network
```

---

## Step 6 — External access

- In UDM Pro: forward **TCP/443** → `10.0.0.5`.
- 
- Configure your External DNS provider to add a new A record pointing (`ha.domain.com`) → WAN IP.
- Configure your Internal DNS (piHole?) to point (`ha.domain.com`) → `10.0.0.5`.

---

## Common Pitfalls & Fixes

- **CSRF token mismatch**  
  Make sure Caddy forwards `Host` and `X-Forwarded-Proto`, and set `OAUTH2_PROXY_COOKIE_DOMAIN` to your hostname.
- **Unauthorized instead of redirect**  
  That happens with `forward_auth`. Use the simpler *auth-proxy* pattern shown here.
- **403 after login**  
  Check that the redirect URI in Entra matches exactly what you put in `OAUTH2_PROXY_REDIRECT_URL`.

---

## Wrapping up

With this setup I can reach **Home Assistant** securely from anywhere with **Microsoft login + MFA**. No exposed HA ports, no sketchy auth screens, and DigiCert TLS all the way.  

This pattern also works for other services (HomeBridge, media servers, etc.) — just add more subdomains, certs, and site blocks in Caddy and bam! you are good to go!

---
