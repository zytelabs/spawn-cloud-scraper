# spawn-cloud-scraper

A single-file web app that generates ready-to-paste bootstrap configs for cloud VMs running a web scraping stack. Fill a form, click Copy (or Download), paste into your cloud provider's user-data field — done.

**Live:** https://zytelabs.github.io/spawn-cloud-scraper

![spawn-cloud-scraper](assets/screenshot.png)

---

## What

Two output modes:

| Mode | Output | Use with |
|---|---|---|
| **Ubuntu 24.04** | `#cloud-config` YAML | Any cloud provider — paste into user-data |
| **Flatcar Linux** | Butane YAML + Ignition JSON | Flatcar-capable providers — paste Ignition JSON |

Services you can include:

| Service | Ubuntu | Flatcar |
|---|---|---|
| Scrapy Shell | native pip | python:3.11-slim container |
| Playwright Python | native pip + chromium | mcr.microsoft.com/playwright/python |
| Puppeteer | native npm | ghcr.io/puppeteer/puppeteer |
| Splash (JS renderer) | — Docker only | scrapinghub/splash |
| Redis | apt | redis:7-alpine |
| PostgreSQL | apt | postgres:16-alpine |
| Tor Proxy | apt | dperson/torproxy |
| mitmproxy | native pip | mitmproxy/mitmproxy |

Everything is self-contained in `index.html` — no build step, no dependencies, no backend.

---

## Why

Spinning up a scraping VM by hand means installing tools one by one, writing systemd units, figuring out docker-compose syntax, and copy-pasting SSH keys. This tool collapses that into a 30-second form.

It also handles the subtle differences between cloud-init (Ubuntu) and Ignition (Flatcar) so you don't have to look up the spec every time.

---

## How it works

```
collectFormState()
  └─ { hostname, sshKeys[], envVars[], services[] }
       ├─ renderCloudConfig(state)  → #cloud-config YAML   (Ubuntu mode)
       ├─ renderButane(state)       → Butane YAML           (Flatcar mode)
       └─ renderIgnition(state)     → Ignition JSON 3.4.0   (Flatcar mode)
```

- **Ubuntu**: selected services collapse into a deduplicated `packages:` list and `runcmd:` list
- **Flatcar**: services become a single `docker-compose.yml` + a `scraper.service` systemd unit that pulls and starts all containers on boot
- Env vars land at `/etc/scraper/.env` on the VM in both modes

---

## Usage

### Web UI

1. Open the app (link above or `index.html` locally)
2. Choose **Ubuntu 24.04** or **Flatcar Linux**
3. Fill in hostname, SSH public keys, and any API keys / env vars
4. Check the services you want
5. Click **Copy** or **Download** — the filename is pre-set for the right CLI tool

---

### CLI deployment

#### DigitalOcean — Ubuntu cloud-init

```bash
# Install doctl — https://docs.digitalocean.com/reference/doctl/how-to/install/
# macOS:  brew install doctl
# Linux:  snap install doctl  (or download binary from GitHub releases)
# Windows: choco install doctl  (or download binary from GitHub releases)
doctl auth init

# Deploy
doctl compute droplet create my-scraper \
  --region nyc3 \
  --size s-2vcpu-4gb \
  --image ubuntu-24-04-x64 \
  --ssh-keys $(doctl compute ssh-key list --no-header --format FingerPrint) \
  --user-data-file ./cloud-config.yaml \
  --wait

# Verify
ssh ubuntu@<ip> cat /var/log/cloud-init-output.log
```

#### Vultr — Flatcar + Ignition

```bash
# Install vultr-cli — https://github.com/vultr/vultr-cli#installation
# macOS:  brew install vultr-cli
# Linux / Windows: download binary from GitHub releases
vultr-cli auth   # enter API key

# Find Flatcar OS ID
vultr-cli os list | grep -i flatcar   # note the ID (e.g. 426)

# Deploy
vultr-cli instance create \
  --region ewr \
  --plan vc2-1c-1gb \
  --os 426 \
  --userdata "$(cat ignition.json)" \
  --label my-scraper

# Verify
ssh core@<ip> docker ps
ssh core@<ip> systemctl status scraper
```

---

## Cloud provider support

### Ubuntu cloud-init

Works on any provider that passes user-data through to cloud-init:

- AWS EC2
- Google Cloud (GCE)
- Azure
- DigitalOcean
- Hetzner Cloud
- Linode / Akamai
- Vultr
- OVHcloud
- Scaleway
- Oracle Cloud (OCI)
- Any OpenStack-based provider

### Flatcar + Ignition

Requires a provider that offers Flatcar as an OS image:

| Provider | Support |
|---|---|
| AWS EC2 | Flatcar AMI in marketplace |
| Google Cloud | Flatcar image available |
| Azure | Flatcar in Marketplace |
| Hetzner Cloud | Flatcar snapshot available |
| Vultr | Built-in OS option |
| Equinix Metal | First-class Flatcar support |
| OpenStack | Works with custom Flatcar image |

---

## License

MIT — see [LICENSE](LICENSE)
