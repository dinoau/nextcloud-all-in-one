# Nextcloud AIO with Cloudflare Tunnel & ACME DNS-01 (Cloudflare)

This guide deploys a production-ready **Nextcloud All-in-One** instance with:

- **Cloudflare Tunnel** — Public access without opening any inbound firewall ports
- **Caddy reverse proxy** — Local HTTPS with valid Let's Encrypt certificates via Cloudflare DNS-01 ACME challenge
- **LocalAI** — On-device AI assistant (via AIO community container)
- **Nextcloud Talk** — Video/audio calls with external TURN server configuration
- **Nextcloud Office (Collabora)** — Document editing
- **Full-text search, Imaginary, ClamAV** — And other AIO optional features

> **Important:** Please review the [Cloudflare Tunnel caveats](https://github.com/nextcloud/all-in-one#notes-on-cloudflare-proxytunnel) in the main AIO documentation before proceeding. Cloudflare Tunnel has known limitations including upload size restrictions, 100-second request timeout, and TLS termination at Cloudflare's edge.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Quick Start](#quick-start)
4. [Detailed Setup](#detailed-setup)
   - [Step 1: Cloudflare Configuration](#step-1-cloudflare-configuration)
   - [Step 2: Server Preparation](#step-2-server-preparation)
   - [Step 3: Deploy the Stack](#step-3-deploy-the-stack)
   - [Step 4: Initial Nextcloud Setup](#step-4-initial-nextcloud-setup)
5. [Enabling Features](#enabling-features)
   - [LocalAI (AI Assistant)](#localai-ai-assistant)
   - [Nextcloud Talk (Video Calls)](#nextcloud-talk-video-calls)
   - [Nextcloud Office (Collabora)](#nextcloud-office-collabora)
   - [Other Optional Features](#other-optional-features)
6. [Best Practices](#best-practices)
7. [Troubleshooting](#troubleshooting)
8. [File Reference](#file-reference)

---

## Architecture Overview

```
                    ┌─────────────────────────────┐
                    │       Cloudflare Edge        │
                    │   (TLS termination, CDN,     │
                    │    DDoS protection)          │
                    └──────────┬──────────────────┘
                               │ Outbound tunnel
                               │ (no inbound ports needed)
┌──────────────────────────────┼──────────────────────────────┐
│  Your Server                 │                              │
│                              ▼                              │
│  ┌─────────────────────────────────────────────────┐        │
│  │  cloudflared (Cloudflare Tunnel connector)      │        │
│  │  Routes: cloud.example.com → localhost:11000    │        │
│  └──────────────────────┬──────────────────────────┘        │
│                         │                                    │
│  ┌──────────────────────┼──────────────────────────┐        │
│  │  Caddy (optional, for local HTTPS)              │        │
│  │  ACME DNS-01 via Cloudflare → localhost:11000   │        │
│  └──────────────────────┼──────────────────────────┘        │
│                         │                                    │
│                         ▼                                    │
│  ┌─────────────────────────────────────────────────┐        │
│  │  Nextcloud AIO Master Container                 │        │
│  │  ├── Apache (port 11000)                        │        │
│  │  ├── Nextcloud                                  │        │
│  │  ├── PostgreSQL                                 │        │
│  │  ├── Redis                                      │        │
│  │  ├── Collabora (Office)                         │        │
│  │  ├── Talk (HPB + signaling)                     │        │
│  │  ├── LocalAI (community container)              │        │
│  │  ├── Imaginary, ClamAV, Fulltextsearch          │        │
│  │  └── BorgBackup                                 │        │
│  └─────────────────────────────────────────────────┘        │
│                                                              │
│  External TURN server (for Talk video/audio calls)  ◄───────┤
│  (e.g., coturn on a separate port/server)                    │
└──────────────────────────────────────────────────────────────┘
```

**Key points:**
- `cloudflared` handles **public internet access** via an outbound-only tunnel (no inbound ports required)
- `caddy` provides **local LAN access** with a valid TLS certificate via Cloudflare DNS-01 ACME
- Nextcloud AIO's Apache listens on `localhost:11000` — not exposed to the internet directly
- An **external TURN server** is required for Talk video/audio calls (the built-in one does not work behind Cloudflare Tunnel)

---

## Prerequisites

- **Server**: Linux host with Docker and Docker Compose v2 installed (minimum 4 GB RAM, 8+ GB recommended for LocalAI)
- **Domain**: A domain managed by Cloudflare DNS (e.g., `example.com`)
- **Cloudflare account**: Free tier is sufficient (but has upload/timeout limits)
- **Cloudflare API token**: For DNS-01 ACME certificate issuance
- **Cloudflare Tunnel**: Created via the Zero Trust dashboard

### Hardware Recommendations

| Feature Set | RAM | CPU | Storage |
|-------------|-----|-----|---------|
| Nextcloud only | 4 GB | 2 cores | 32 GB+ |
| + Talk + Office | 8 GB | 4 cores | 64 GB+ |
| + LocalAI | 16 GB+ | 4+ cores | 100 GB+ |

---

## Quick Start

```bash
# 1. Clone this repository (or copy the cloudflare-tunnel directory)
git clone https://github.com/nextcloud/all-in-one.git
cd all-in-one/cloudflare-tunnel

# 2. Create your .env file
cp .env.example .env
# Edit .env with your values (domain, tokens, etc.)

# 3. Start the stack
docker compose up -d

# 4. Access the AIO interface
# Open https://localhost:8080 in your browser
# (Accept the self-signed certificate warning)
# Note the initial admin password shown in the logs:
docker logs nextcloud-aio-mastercontainer
```

---

## Detailed Setup

### Step 1: Cloudflare Configuration

#### 1a. Create a Cloudflare API Token

1. Go to [Cloudflare API Tokens](https://dash.cloudflare.com/profile/api-tokens)
2. Click **Create Token**
3. Use the **Edit zone DNS** template, or create a custom token with:
   - **Permissions**: `Zone:DNS:Edit` and `Zone:Zone:Read`
   - **Zone Resources**: Select your specific zone (e.g., `example.com`)
4. Copy the token and save it as `CLOUDFLARE_API_TOKEN` in your `.env` file

#### 1b. Create a Cloudflare Tunnel

1. Go to [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/) → **Networks** → **Tunnels**
2. Click **Create a tunnel** → Choose **Cloudflared**
3. Name your tunnel (e.g., `nextcloud`)
4. On the **Install connector** step, copy the **tunnel token** (starts with `eyJ...`)
5. Save the token as `CLOUDFLARE_TUNNEL_TOKEN` in your `.env` file
6. Configure the **Public Hostname**:
   - **Subdomain**: `cloud` (or your preferred subdomain)
   - **Domain**: `example.com` (your domain)
   - **Service**: `http://localhost:11000`
7. Under **Additional application settings** → **HTTP Settings**:
   - Enable **No TLS Verify** (since AIO Apache uses HTTP internally)
   - Set **HTTP Host Header** to `cloud.example.com`

#### 1c. Configure Cloudflare DNS and Security Settings

1. In the Cloudflare dashboard for your domain:
   - **Speed** → **Optimization** → Disable **Rocket Loader** (required — Nextcloud's login page will not render otherwise)
   - **SSL/TLS** → Set mode to **Full** (not Full Strict, since the origin uses HTTP behind the tunnel)
   - **Rules** → **Page Rules** (or WAF Custom Rules) → Add a rule for `cloud.example.com/*`:
     - Disable **Email Address Obfuscation**
     - Disable **Auto Minify** (optional but recommended)
2. **HSTS**: Under **SSL/TLS** → **Edge Certificates**, enable **Always Use HTTPS** and **HSTS** with `max-age=31536000; includeSubDomains; preload`

### Step 2: Server Preparation

```bash
# Install Docker (if not already installed)
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# Recommended: Set up a firewall (e.g., ufw)
# No inbound ports need to be opened for Cloudflare Tunnel!
# Only allow SSH and the AIO admin interface (8080) from trusted IPs:
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow from 192.168.0.0/16 to any port 8080  # AIO admin (LAN only)
sudo ufw allow from 192.168.0.0/16 to any port 443    # Local Caddy HTTPS (LAN only)
sudo ufw enable
```

### Step 3: Deploy the Stack

```bash
cd cloudflare-tunnel

# Copy and configure environment
cp .env.example .env
nano .env  # Fill in your values

# Deploy
docker compose up -d

# Check logs
docker logs -f nextcloud-aio-mastercontainer
docker logs -f nextcloud-aio-cloudflared
docker logs -f nextcloud-aio-caddy
```

### Step 4: Initial Nextcloud Setup

1. **Access the AIO interface**: Open `https://localhost:8080` (or `https://<server-lan-ip>:8080`) in your browser
2. **Note the password**: The initial admin password is shown in the mastercontainer logs
3. **Enter your domain**: Type `cloud.example.com` (your configured domain) — domain validation will be skipped automatically since `SKIP_DOMAIN_VALIDATION=true` is set
4. **Select optional containers**: Enable the features you want (Talk, Collabora, ClamAV, Fulltextsearch, Imaginary, etc.)
5. **Start the containers**: Click the start button and wait for all containers to come up
6. **Access Nextcloud**: Visit `https://cloud.example.com` in your browser

---

## Enabling Features

### LocalAI (AI Assistant)

LocalAI provides on-device AI capabilities for the Nextcloud Assistant, Smart Inbox, and other AI-powered features — without sending data to external services.

#### Setup

1. In the AIO interface (`https://localhost:8080`), go to the optional containers section
2. Under **Community Containers**, add `local-ai` by typing `local-ai` in the community containers field
3. Restart the AIO containers
4. The following apps are automatically installed and configured:
   - `integration_openai` — Connects Nextcloud to LocalAI
   - `assistant` — Provides the AI assistant interface

#### Hardware Requirements

- **CPU-only**: Works but slow. Minimum 8 GB RAM for small models
- **GPU (Vulkan)**: Recommended. Add your GPU device to the mastercontainer:
  ```yaml
  # In docker-compose.yml, add under nextcloud-aio-mastercontainer:
  devices:
    - /dev/dri  # For Intel/AMD GPU (Vulkan)
  environment:
    - NEXTCLOUD_ENABLE_NVIDIA_GPU=true  # For NVIDIA GPU
  ```

#### Best Practices for LocalAI

- Start with small models and increase size as needed
- Monitor memory usage — LocalAI can consume significant RAM
- For production use, consider a dedicated GPU
- Increase the PHP memory limit if you encounter timeouts:
  ```yaml
  environment:
    - NEXTCLOUD_MEMORY_LIMIT=1024M
  ```

### Nextcloud Talk (Video Calls)

> ⚠️ **The built-in TURN server does NOT work behind Cloudflare Tunnel.** Cloudflare Tunnel only supports HTTP/HTTPS traffic on ports 80/443. The TURN server requires a separate UDP/TCP port (default 3478) which Cloudflare Tunnel cannot forward.

#### Option A: Use a Self-Hosted TURN Server (Recommended)

Deploy a [coturn](https://github.com/coturn/coturn) server on a machine with a public IP (can be a small VPS):

```bash
# On a separate server with public IP, install coturn:
sudo apt install coturn

# Configure /etc/turnserver.conf:
cat << 'EOF' | sudo tee /etc/turnserver.conf
listening-port=3478
tls-listening-port=5349
realm=turn.example.com
server-name=turn.example.com
# Use static-auth-secret for shared secret authentication
use-auth-secret
static-auth-secret=REPLACE_WITH_A_STRONG_RANDOM_SECRET
total-quota=100
stale-nonce=600
# Certificates (use certbot or similar)
# cert=/etc/letsencrypt/live/turn.example.com/fullchain.pem
# pkey=/etc/letsencrypt/live/turn.example.com/privkey.pem
no-multicast-peers
fingerprint
EOF

# Enable and start
sudo systemctl enable coturn
sudo systemctl start coturn
```

Then configure in Nextcloud:

1. Go to **Administration** → **Talk** settings
2. Add your TURN server:
   - **TURN server URL**: `turn:turn.example.com:3478` and `turns:turn.example.com:5349`
   - **TURN secret**: Your shared secret
   - **Protocol**: UDP and TCP
3. Add your STUN server:
   - **STUN server URL**: `stun:turn.example.com:3478`

#### Option B: Use a Public TURN Server

For testing or small deployments, you can use public TURN servers:

1. Go to Nextcloud **Administration** → **Talk** settings
2. Add STUN server: `stun:stun.nextcloud.com:443`
3. For TURN, you will need your own or a provider's server (public free TURN servers are rare due to bandwidth costs)

### Nextcloud Office (Collabora)

Collabora (Nextcloud Office) requires adding Cloudflare's IP ranges to the WOPI allowlist:

1. Enable Collabora in the AIO interface
2. After Nextcloud is running, go to **Administration** → **Nextcloud Office**
3. Add all [Cloudflare IP ranges](https://www.cloudflare.com/ips/) to the **Allow list for WOPI requests**:
   ```
   173.245.48.0/20,103.21.244.0/22,103.22.200.0/22,103.31.4.0/22,141.101.64.0/18,108.162.192.0/18,190.93.240.0/20,188.114.96.0/20,197.234.240.0/22,198.41.128.0/17,162.158.0.0/15,104.16.0.0/13,104.24.0.0/14,172.64.0.0/13,131.0.72.0/22
   ```
4. Also add IPv6 ranges from the same page if needed

### Other Optional Features

Enable these in the AIO interface at `https://localhost:8080`:

| Feature | Description | Notes |
|---------|-------------|-------|
| **ClamAV** | Antivirus scanning | Adds ~1 GB RAM usage |
| **Fulltextsearch** | Full-text search with Elasticsearch | Adds ~1–2 GB RAM usage |
| **Imaginary** | Preview generation for HEIC, PDF, SVG, TIFF, etc. | Lightweight |
| **Whiteboard** | Collaborative whiteboard | Lightweight |
| **Docker Socket Proxy** | Required for Nextcloud AppAPI (ExApps) | Lightweight |
| **Talk Recording** | Server-side recording of Talk calls | Requires additional resources |

---

## Best Practices

### Security

- [ ] **Firewall**: Keep all inbound ports closed. Only allow SSH and AIO admin (8080) from trusted networks
- [ ] **Disable Rocket Loader**: In Cloudflare dashboard — otherwise the login page breaks
- [ ] **Enable 2FA**: Set up two-factor authentication for all admin accounts
- [ ] **Regular backups**: Enable AIO's built-in BorgBackup from the admin interface
- [ ] **HSTS**: Enable in Cloudflare SSL/TLS settings
- [ ] **CSP and Headers**: Cloudflare adds security headers automatically, but verify in Nextcloud's admin overview

### Performance

- [ ] **Cloudflare caching**: Cloudflare will cache static assets automatically. Nextcloud sets appropriate cache headers
- [ ] **PHP memory**: Increase `NEXTCLOUD_MEMORY_LIMIT` to `1024M` or higher when using LocalAI
- [ ] **Redis**: Enabled by default in AIO — no action needed
- [ ] **Upload chunking**: Configure Nextcloud Desktop Client to use small chunk sizes (e.g., 5 MB) when behind Cloudflare's free plan (100 MB upload limit per request). See the [Nextcloud Desktop Client documentation](https://docs.nextcloud.com/desktop/latest/) for chunking configuration

### Reliability

- [ ] **Automatic restarts**: All containers use `restart: always`
- [ ] **Daily backups**: Enable and configure BorgBackup schedule in the AIO interface
- [ ] **Monitor disk space**: Ensure sufficient disk space for backups, LocalAI models, and Nextcloud data
- [ ] **Update strategy**: AIO handles updates automatically. Check the AIO interface periodically for available updates
- [ ] **Test your TURN server**: Use the Talk settings test to verify video calling works

### Cloudflare-Specific

- [ ] **Upload limits**: Cloudflare free plan limits uploads to 100 MB per request. Logged-in users using the web interface or desktop/mobile clients use chunking and are not affected. Public link uploads are limited
- [ ] **Request timeout**: Cloudflare has a 100-second request timeout that is not configurable. Very large file assembly operations may time out
- [ ] **Email obfuscation**: Disable for your Nextcloud domain to prevent email addresses in Nextcloud from being mangled

---

## Troubleshooting

### Cloudflare Tunnel not connecting

```bash
# Check cloudflared logs
docker logs nextcloud-aio-cloudflared

# Verify the tunnel token is correct
# Re-check in Cloudflare Zero Trust > Networks > Tunnels
```

### Domain validation errors

Domain validation is **expected to fail** behind Cloudflare Tunnel. This is why `SKIP_DOMAIN_VALIDATION=true` is set. If you see validation errors in the AIO interface, verify:

1. The domain is entered correctly
2. The tunnel is connected (check Cloudflare dashboard)
3. The public hostname route points to `http://localhost:11000`

### Nextcloud login page blank or broken

This is almost always caused by **Cloudflare Rocket Loader**:

1. Go to Cloudflare Dashboard → **Speed** → **Optimization**
2. Disable **Rocket Loader**
3. Clear your browser cache and retry

### Collabora (Office) not loading

Add all [Cloudflare IP ranges](https://www.cloudflare.com/ips/) to the WOPI allowlist. See the [Nextcloud Office section](#nextcloud-office-collabora) above.

### Talk video calls not working

The built-in TURN server does not work behind Cloudflare Tunnel. You must configure an external TURN server. See the [Talk section](#nextcloud-talk-video-calls) above.

### Certificate issues with Caddy

```bash
# Check Caddy logs for ACME errors
docker logs nextcloud-aio-caddy

# Verify your Cloudflare API token has the correct permissions:
# Zone:DNS:Edit and Zone:Zone:Read
# Verify the token is scoped to your zone
```

### Large file uploads fail (413 error)

This is a Cloudflare limitation. Options:
- Use the Nextcloud Desktop/Mobile clients (they use chunking)
- For public link uploads, files must be under 100 MB on Cloudflare free plan
- Consider upgrading to Cloudflare Pro for higher limits (up to 500 MB)

### AIO interface not accessible

```bash
# The AIO interface runs on port 8080 with a self-signed certificate
# Access via https://localhost:8080 or https://<server-ip>:8080
docker logs nextcloud-aio-mastercontainer

# If port 8080 is blocked by firewall:
sudo ufw allow from 192.168.0.0/16 to any port 8080
```

### LocalAI not responding or slow

```bash
# Check LocalAI container logs
docker logs nextcloud-aio-local-ai

# Check memory usage
docker stats nextcloud-aio-local-ai

# Ensure sufficient RAM (8+ GB recommended)
# Consider enabling GPU acceleration
```

---

## File Reference

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Main Compose file defining all services |
| `.env.example` | Template for environment variables — copy to `.env` |
| `Caddyfile` | Caddy reverse proxy config with Cloudflare DNS-01 ACME |
| `README.md` | This guide |

---

## Additional Resources

- [Nextcloud AIO Documentation](https://github.com/nextcloud/all-in-one)
- [AIO Reverse Proxy Guide](https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md)
- [AIO Community Containers](https://github.com/nextcloud/all-in-one/tree/main/community-containers)
- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [Caddy Cloudflare DNS Module](https://github.com/caddy-dns/cloudflare)
- [Nextcloud Talk TURN Server Guide](https://nextcloud-talk.readthedocs.io/en/latest/TURN/)
- [Cloudflare Caveats for Nextcloud](https://github.com/nextcloud/all-in-one#notes-on-cloudflare-proxytunnel)
