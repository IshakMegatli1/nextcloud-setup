# Nextcloud Self-Hosting Documentation
**Author:** Ishak  
**Server:** ThinkCentre
**OS:** Ubuntu (Linux)  
**Date:** March 13, 2026  
**Domain:** nextcloud.cloud-9.app  
**TECHNOLOGIES / NETWORKING CONCEPTS USED** : Docker & Containerization, DNS, Port Forwarding & NAT, Cloudflare Tunnel, Reverse Proxy & TLS Termination, Self-Hosting Architecture, Network Security — IP hiding, trusted domains, HTTPS enforcement

---

## Table of Contents
1. [Overview](#overview)
2. [Initial Problem](#initial-problem)
3. [Why Port Forwarding Failed](#why-port-forwarding-failed)
4. [Why Cloudflare Tunnel Was Used](#why-cloudflare-tunnel-was-used)
5. [Architecture Diagram](#architecture-diagram)
6. [Step-by-Step Setup](#step-by-step-setup)
   - [Step 1: DNS Setup on name.com](#step-1-dns-setup-on-namecom)
   - [Step 2: Cloudflare Setup](#step-2-cloudflare-setup)
   - [Step 3: Install cloudflared on ThinkCentre](#step-3-install-cloudflared-on-thinkcentre)
   - [Step 4: Create and Configure the Tunnel](#step-4-create-and-configure-the-tunnel)
   - [Step 5: Reconfigure Nextcloud AIO](#step-5-reconfigure-nextcloud-aio)
   - [Step 6: Install Tunnel as a Permanent Service](#step-6-install-tunnel-as-a-permanent-service)
7. [Difficulties Encountered](#difficulties-encountered)
8. [Final Configuration Files](#final-configuration-files)

---

## Overview

This document describes the complete setup process to make a self-hosted Nextcloud instance accessible from the internet via the domain `nextcloud.cloud-9.app`, running on a Lenovo ThinkCentre M710q home server behind a Virgin Canada residential ISP connection.

**Stack used:**
- Nextcloud AIO (All-in-One) via Docker
- Caddy (built into Nextcloud AIO apache container)
- Cloudflare Tunnel (`cloudflared`) for ingress
- Cloudflare DNS for domain management
- name.com as domain registrar

---

## Initial Problem

All Docker containers were running and healthy, but visiting `https://nextcloud.cloud-9.app` resulted in an infinite loading screen. The containers confirmed healthy included:

- `nextcloud-aio-apache` → ports 443, 80
- `nextcloud-aio-nextcloud` → port 9000
- `nextcloud-aio-mastercontainer` → ports 80, 8080, 8443
- `nextcloud-aio-database`, `nextcloud-aio-redis`, `nextcloud-aio-onlyoffice`, etc.

---

## Why Port Forwarding Failed

Port forwarding rules were correctly set on the Virgin router:

| Rule Name    | Protocol | External Port | Internal Port | Device                  |
|-------------|----------|---------------|---------------|-------------------------|
| Nextcloud80  | Both     | 80            | 80            | ThinkCentre-M710q |
| Nextcloud443 | Both     | 443           | 443           | ThinkCentre-M710q |

Despite correct rules, `canyouseeme.org` confirmed:
> **Error: I could not see your service on **.***.**.*** on port 80. Reason: Connection timed out.**

**Root cause:** Virgin Canada blocks inbound ports 80 and 443 on residential internet plans at the ISP level — before traffic even reaches the router. This is a deliberate policy to prevent customers from running servers on residential plans.

Additionally, even if port 80 were reachable, Cloudflare's proxy intercepts TLS connections, making Let's Encrypt's TLS-ALPN-01 challenge impossible:

```
Error: Cannot negotiate ALPN protocol "acme-tls/1" for tls-alpn-01 challenge
HTTP 403 urn:ietf:params:acme:error:unauthorized
```

---

## Why Cloudflare Tunnel Was Used

### Port Forwarding vs. Cloudflare Tunnel

| Feature                  | Port Forwarding         | Cloudflare Tunnel       |
|--------------------------|------------------------|------------------------|
| Connection direction     | Inbound ←              | Outbound →             |
| Virgin can block it?     | ✅ Yes                 | ❌ No                  |
| Ports needed open        | 80, 443                | None                   |
| SSL certificate          | Caddy (failed)         | Cloudflare (handled)   |
| Public IP exposed        | ✅ Yes                 | ❌ No (hidden)         |
| Works on residential ISP | ❌ Often blocked       | ✅ Always works        |

### How Cloudflare Tunnel Works

```
ThinkCentre (cloudflared daemon)
        ↓
  Outbound QUIC connection (Virgin cannot block outbound)
        ↓
  Cloudflare's global network (yul01 Montreal, ewr07 New York)
        ↑
  Browser → https://nextcloud.cloud-9.app
```

`cloudflared` running on the ThinkCentre dials **outbound** to Cloudflare on startup.
When a user visits the domain, Cloudflare pipes the request through the existing
outbound connection — no inbound connection ever touches Virgin's network.

---

## Step-by-Step Setup

### Step 1: DNS Setup on name.com

The domain `cloud-9.app` was registered on **name.com**.

**Original DNS records on name.com (before migration):**

| Type | Host                    | Answer        | TTL  |
|------|------------------------|---------------|------|
| A    | cloud-9.app            | **.***.**.*** | 3600 |
| A    | nextcloud.cloud-9.app  | **.***.**.*** | 300  |

**Action required:** Change nameservers on name.com to point to Cloudflare.

1. Log into **name.com** → My Domains → click `cloud-9.app`
2. Click **Nameservers**
3. Delete existing name.com nameservers
4. Add both Cloudflare nameservers (obtained from Cloudflare dashboard):
   ```
   <assigned>.ns.cloudflare.com
   <assigned>.ns.cloudflare.com
   ```
5. Click **Save Changes**

> ⚠️ Do NOT edit DNS records on name.com after this — Cloudflare takes full control.
> Cloudflare automatically imports all existing DNS records during setup.

**Propagation time:** Minutes to a few hours (up to 48h maximum).

---

### Step 2: Cloudflare Setup

1. Create a free account at **cloudflare.com**
2. Click **Add a domain** → enter `cloud-9.app` → select **Free plan**
3. Cloudflare scans and imports existing DNS records automatically
4. Copy the two assigned nameservers → paste into name.com (Step 1)
5. Click **"I updated my nameservers"** button on Cloudflare
6. Wait for activation email from Cloudflare

**Once active, delete the old A record for nextcloud:**
- Cloudflare Dashboard → `cloud-9.app` → DNS → Records
- Delete: `nextcloud.cloud-9.app → **.***.**.***`
- (This will be replaced by a CNAME pointing to the tunnel)

---

### Step 3: Install cloudflared on ThinkCentre

```bash
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflared.list

sudo apt update && sudo apt install cloudflared
```

---

### Step 4: Create and Configure the Tunnel

**Login to Cloudflare:**
```bash
cloudflared tunnel login
# Opens browser URL → select cloud-9.app → Authorize
# Credentials saved to /home/ishak/.cloudflared/cert.pem
```

**Create tunnel:**
```bash
cloudflared tunnel create nextcloud-tunnel
# Note the tunnel ID: c02240e1-0905-48d1-8801-3aa67538007b
```

**Create config file:**
```bash
nano ~/.cloudflared/config.yml
```

```yaml
tunnel: c02240e1-0905-48d1-8801-3aa67538007b
credentials-file: /home/ishak/.cloudflared/c02240e1-0905-48d1-8801-3aa67538007b.json

ingress:
  - hostname: nextcloud.cloud-9.app
    service: http://localhost:11000
  - service: http_status:404
```

**Route DNS (creates CNAME on Cloudflare automatically):**
```bash
cloudflared tunnel route dns nextcloud-tunnel nextcloud.cloud-9.app
# Output: Added CNAME nextcloud.cloud-9.app which will route to this tunnel
```

**Test run:**
```bash
cloudflared tunnel run nextcloud-tunnel
```

Expected output (tunnel healthy):
```
INF Registered tunnel connection connIndex=0 ... location=ewr07 protocol=quic
INF Registered tunnel connection connIndex=1 ... location=yul01 protocol=quic
INF Registered tunnel connection connIndex=2 ... location=yul01 protocol=quic
INF Registered tunnel connection connIndex=3 ... location=ewr12 protocol=quic
```

---

### Step 5: Reconfigure Nextcloud AIO

By default, Nextcloud AIO runs in **standalone mode** — Caddy handles TLS on port 443.
Behind a Cloudflare Tunnel, Cloudflare handles TLS, so Nextcloud AIO must switch to
**reverse proxy mode** — Apache listens on port 11000 (plain HTTP) and stops trying
to get its own Let's Encrypt certificate.

**Check current mastercontainer config:**
```bash
sudo docker inspect nextcloud-aio-mastercontainer | grep -A 20 "Env"
```

**Recreate mastercontainer with APACHE_PORT=11000:**

If using `docker run`:
```bash
sudo docker stop nextcloud-aio-mastercontainer
sudo docker rm nextcloud-aio-mastercontainer

sudo docker run \
--sig-proxy=false \
--name nextcloud-aio-mastercontainer \
--restart always \
--publish 80:80 \
--publish 8080:8080 \
--publish 8443:8443 \
--env SKIP_DOMAIN_VALIDATION=true \
--env APACHE_PORT=11000 \
--env APACHE_IP_BINDING=0.0.0.0 \
--volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config \
--volume /var/run/docker.sock:/var/run/docker.sock:ro \
nextcloud/all-in-one:latest
```

> ⚠️ The `--volume nextcloud_aio_mastercontainer` flag preserves all existing Nextcloud
> data. Nothing is lost when recreating the mastercontainer.

**Why port 11000:**

| Mode           | Who handles TLS? | Apache listens on |
|----------------|-----------------|-------------------|
| Standalone     | Caddy (AIO)     | Port 443 (HTTPS)  |
| Reverse proxy  | Cloudflare      | Port 11000 (HTTP) |

---

### Step 6: Install Tunnel as a Permanent Service

Copy config files to system path (required because the service runs as root):

```bash
sudo mkdir -p /etc/cloudflared

sudo cp ~/.cloudflared/config.yml /etc/cloudflared/config.yml

sudo cp ~/.cloudflared/c02240e1-0905-48d1-8801-3aa67538007b.json \
        /etc/cloudflared/c02240e1-0905-48d1-8801-3aa67538007b.json
```

Update credentials path in system config:
```bash
sudo nano /etc/cloudflared/config.yml
```

```yaml
tunnel: c02240e1-0905-48d1-8801-3aa67538007b
credentials-file: /etc/cloudflared/c02240e1-0905-48d1-8801-3aa67538007b.json

ingress:
  - hostname: nextcloud.cloud-9.app
    service: http://localhost:11000
  - service: http_status:404
```

Install and enable the service:
```bash
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```

Verify:
```bash
sudo systemctl status cloudflared
# Expected: active (running)
```

---

## Difficulties Encountered

### 1. Virgin ISP Blocks Ports 80 and 443
- **Symptom:** Site keeps loading, `canyouseeme.org` shows connection timeout
- **Diagnosis:** Port forwarding rules were correct on the router, but Virgin blocks
  inbound 80/443 at the ISP level on residential plans
- **Fix:** Switched to Cloudflare Tunnel (outbound connection, no ports needed)

### 2. Cloudflare CNAME Conflict with Existing A Record
- **Error:** `Failed to create record nextcloud.cloud-9.app: An A, AAAA, or CNAME record with that host already exists`
- **Cause:** Old A record `nextcloud.cloud-9.app → **.***.**.***` imported from name.com
- **Fix:** Manually deleted the old A record in Cloudflare DNS dashboard before re-running `cloudflared tunnel route dns`

### 3. Wrong credentials-file Path in config.yml
- **Error:** `Tunnel credentials file '/root/.cloudflared/YOUR-TUNNEL-ID.json' doesn't exist`
- **Cause 1:** Placeholder `YOUR-TUNNEL-ID` was never replaced with actual tunnel ID
- **Cause 2:** Tunnel was created as user `ishak`, but config pointed to `/root/`
- **Fix:** Used `cloudflared tunnel list` to get real ID, and corrected path to `/home/ishak/.cloudflared/<tunnel-id>.json`

### 4. TLS Handshake Error (https vs http on port 11000)
- **Error:** `tls: first record does not look like a TLS handshake`  
  `originService=https://localhost:11000`
- **Cause:** Config file had `https://localhost:11000` but port 11000 serves plain HTTP
- **Fix:** Changed `https` → `http` in `config.yml`

### 5. Let's Encrypt TLS-ALPN-01 Challenge Failing
- **Error:** `Cannot negotiate ALPN protocol "acme-tls/1" for tls-alpn-01 challenge`
- **Cause:** Cloudflare proxies port 443, so Let's Encrypt could never reach Caddy
  directly to complete the TLS challenge
- **Fix:** Switched Nextcloud AIO to reverse proxy mode (`APACHE_PORT=11000`).
  Caddy stops trying to get its own certificate. Cloudflare provides TLS instead.

### 6. Service Install Fails — Config Not Found
- **Error:** `Cannot determine default configuration path. No file [config.yml] in [~/.cloudflared ...]`
- **Cause:** `cloudflared service install` runs as root and looks in `/etc/cloudflared/`,
  not in `/home/ishak/.cloudflared/`
- **Fix:** Copied both `config.yml` and the tunnel credentials JSON to `/etc/cloudflared/`
  and updated the `credentials-file` path accordingly

---

## Final Configuration Files

### /etc/cloudflared/config.yml
```yaml
tunnel: c02240e1-0905-48d1-8801-3aa67538007b
credentials-file: /etc/cloudflared/c02240e1-0905-48d1-8801-3aa67538007b.json

ingress:
  - hostname: nextcloud.cloud-9.app
    service: http://localhost:11000
  - service: http_status:404
```

### Cloudflare DNS Records (after setup)
| Type  | Name                   | Content                              | Proxy |
|-------|------------------------|--------------------------------------|-------|
| A     | cloud-9.app            | **.***.**.***                       | ✅    |
| CNAME | nextcloud.cloud-9.app  | c02240e1-...cfargotunnel.com         | ✅    |

### Docker Containers (docker ps)
| Container                    | Ports                              |
|-----------------------------|-------------------------------------|
| nextcloud-aio-apache         | 0.0.0.0:443→443, 11000 (HTTP)      |
| nextcloud-aio-nextcloud      | 9000                                |
| nextcloud-aio-mastercontainer| 80, 8080, 8443                      |
| nextcloud-aio-database       | 5432                                |
| nextcloud-aio-redis          | 6379                                |
| nextcloud-aio-onlyoffice     | 80, 443                             |
| nextcloud-aio-imaginary      | —                                   |
| nextcloud-aio-notify-push    | —                                   |

---

## Result

✅ `https://nextcloud.cloud-9.app` — accessible globally with green lock  
✅ TLS handled by Cloudflare — no Let's Encrypt issues  
✅ Server IP hidden — Virgin/internet never sees ThinkCentre directly  
✅ Cloudflare Tunnel running as systemd service — auto-starts on reboot  
✅ Nextcloud AIO in reverse proxy mode — stable, no cert errors  
