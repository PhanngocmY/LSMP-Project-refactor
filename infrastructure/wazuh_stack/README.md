# Wazuh SIEM Stack

This directory contains the Docker Compose configurations and environment files to deploy a single-node **Wazuh SIEM (Security Information and Event Management)** stack.

---

## 📂 Directory Structure

```text
infrastructure/wazuh_stack/
├── docker-compose.yml             # Main Wazuh stack (Manager, Indexer, Dashboard)
├── generate-indexer-certs.yml     # Cert generator service helper
├── .env.example                   # Environment variables template (NEW)
├── README.md                      # This documentation file
└── config/                        # Security and service configuration mounts
    ├── certs.yml                  # Nodes certificate definition
    ├── wazuh_cluster/             # Wazuh manager cluster settings
    ├── wazuh_dashboard/           # Dashboard SSL/TLS config
    └── wazuh_indexer/             # OpenSearch/Indexer security configurations
```

---

## 🏛 Component Architecture

The stack consists of three major components communicating over secure TLS connections:

| Component | Container Name | Default Ports | Description |
|---|---|---|---|
| **Wazuh Manager** | `wazuh.manager` | `1514/tcp`, `1515/tcp`, `514/udp`, `55000/tcp` (localhost only) | Core manager receiving agent alerts, running decoding/rule matching engines, and hosting the Wazuh API. |
| **Wazuh Indexer** | `wazuh.indexer` | `9200/tcp` (localhost only) | High-performance search and analytics engine (OpenSearch) storing alerts and system events. |
| **Wazuh Dashboard** | `wazuh.dashboard` | `443/tcp` | Web UI for threat detection analysis, visualization, and agent fleet monitoring. |

---

## 🔧 What Was Changed in This Branch (`wazuh_stack_fix`)

| Finding ID | File | Problem | Fix | Commit |
|---|---|---|---|---|
| WZH-01 | `docker-compose.yml`, `.env.example` | Hardcoded secrets (`SecretPassword`, `kibanaserver`, `MyS3cr37P450r`) | Moved to `${VAR:?error}` interpolation with `.env.example` | `fix(wazuh_stack): move secrets to env vars...` |
| WZH-02 | `docker-compose.yml` | Internal ports 55000 and 9200 exposed on 0.0.0.0 | Bound to `127.0.0.1` only | (included in WZH-01 commit) |
| WZH-03 | `docker-compose.yml` | Deprecated `links` directive | Removed entirely — Docker Compose auto-resolves hostnames | (included in WZH-01 commit) |
| WZH-04 | `docker-compose.yml` | Missing healthchecks, dashboard starts before indexer ready | Added healthchecks + `depends_on: condition: service_healthy` | (included in WZH-01 commit) |
| WZH-05 | `config/certs.yml` | Hostnames in `ip:` field instead of actual IPs | Replaced with `127.0.0.1` | `fix(wazuh_stack): use valid IP addresses...` |
| WZH-06 | `README.md` | Step numbering jumps from 3 to 5 | Fixed in this README rewrite | `docs(wazuh_stack): update README...` |

---

## 🚀 Step-by-Step Deployment Guide

Follow these steps to generate certificates and launch the SIEM environment.

### Step 1: Prerequisites — Host Tuning

Wazuh Indexer uses an embedded OpenSearch engine. You **must** increase the virtual memory mapping limit on your Linux host.

```bash
sudo sysctl -w vm.max_map_count=262144
```

**Expected result:**
```
vm.max_map_count = 262144
```

To make this permanent, add to `/etc/sysctl.conf`:
```
vm.max_map_count=262144
```

**Verify:**
```bash
sysctl vm.max_map_count
```
**Expected:** `vm.max_map_count = 262144`

---

### Step 2: Configure Environment Variables

```bash
cd infrastructure/wazuh_stack
cp .env.example .env
```

Edit `.env` and set **ALL** passwords to strong values:
```env
INDEXER_PASSWORD=<strong-password-1>
API_PASSWORD=<strong-password-2>
DASHBOARD_PASSWORD=<strong-password-3>
```

> ⚠️ **The stack will REFUSE to start** if `INDEXER_PASSWORD`, `API_PASSWORD`, or `DASHBOARD_PASSWORD` are not set. This is intentional to prevent deploying with default credentials.

**Expected result:** `.env` file created with custom passwords.

---

### Step 3: Generate TLS Certificates

Wazuh components enforce TLS-only communications. Generate local certificates using the cert generator:

```bash
docker compose -f generate-indexer-certs.yml run --rm generator
```

**Expected result:**
```
Creating network "wazuh_stack_default" ...
Creating wazuh_stack_generator_run ...
The certificates have been generated. This container will exit now.
```

**Verify certificates were created:**
```bash
ls config/wazuh_indexer_ssl_certs/
```

**Expected output:**
```
admin-key.pem  admin.pem  root-ca.pem  root-ca-manager.pem
wazuh.dashboard-key.pem  wazuh.dashboard.pem
wazuh.indexer-key.pem    wazuh.indexer.pem
wazuh.manager-key.pem   wazuh.manager.pem
```

---

### Step 4: Start the Wazuh Stack

```bash
docker compose up -d
```

**Expected result:**
```
[+] Running 3/3
 ✔ Container wazuh.indexer    Started
 ✔ Container wazuh.manager    Started
 ✔ Container wazuh.dashboard  Started
```

> ℹ️ **Startup takes 1-2 minutes** on first run while the indexer initializes core index pattern schemas. The dashboard will wait until both the indexer and manager are healthy before starting (via `depends_on` healthcheck conditions).

---

### Step 5: Verify All Services Are Healthy

```bash
docker compose ps
```

**Expected result** (after ~2 minutes):
```
NAME              SERVICE          STATUS           PORTS
wazuh.indexer     wazuh.indexer    Up (healthy)     127.0.0.1:9200->9200/tcp
wazuh.manager     wazuh.manager    Up (healthy)     0.0.0.0:1514-1515->1514-1515/tcp, 0.0.0.0:514->514/udp, 127.0.0.1:55000->55000/tcp
wazuh.dashboard   wazuh.dashboard  Up               0.0.0.0:443->5601/tcp
```

Key things to verify:
- ✅ All containers show `Up (healthy)` or `Up`
- ✅ Ports `9200` and `55000` are bound to `127.0.0.1` (not `0.0.0.0`)
- ✅ Dashboard port `443` is on `0.0.0.0` (public-facing)

---

### Step 6: Verify Indexer Is Responding

```bash
curl -fks https://localhost:9200 -u admin:<your-INDEXER_PASSWORD>
```

**Expected result:** JSON response with cluster info:
```json
{
  "name" : "wazuh.indexer",
  "cluster_name" : "wazuh-cluster",
  "cluster_uuid" : "...",
  "version" : { ... },
  "tagline" : "The OpenSearch Project"
}
```

> ℹ️ This URL is only accessible from localhost due to the `127.0.0.1` binding (WZH-02 fix).

---

### Step 7: Verify Internal Ports Are NOT Exposed Externally

```bash
ss -tlnp | grep -E '9200|55000' | grep -v 127.0.0.1
```

**Expected result:** **NO output** — both ports are bound exclusively to `127.0.0.1`.

---

### Step 8: Access the Dashboard

Open your web browser and navigate to:

```
https://localhost
```

**Expected result:**
1. Browser shows a self-signed certificate warning — **this is normal**, accept it
2. Wazuh Dashboard login page appears
3. Login with:
   - **Username:** `admin`
   - **Password:** `<your-INDEXER_PASSWORD>` from `.env`
4. After login, you should see the Wazuh Overview page with agent status and security events

---

## 🛑 Teardown & Maintenance

Stop and preserve the volume states:
```bash
docker compose down
```

Stop, destroy containers, and wipe local Docker configurations:
```bash
docker compose down -v
```

---

## ⚠️ Impact and Risks

| Change | Impact | Risk |
|---|---|---|
| WZH-01: Secrets in .env | No more hardcoded passwords in compose file | Stack won't start without `.env` — intentional |
| WZH-02: Localhost binding | Indexer and Manager API only accessible from the host machine | External monitoring tools using 9200/55000 will break — use SSH tunnel instead |
| WZH-03: No links | No functional change — Docker Compose DNS handles resolution | None |
| WZH-04: Healthchecks | Dashboard waits for indexer+manager before starting | First startup may take slightly longer (~30s) |
| WZH-05: Valid cert IPs | Certificates now have correct x509 IP SAN extensions | Must regenerate certs with Step 3 if existing certs were generated with old config |

---

## 📋 Known Issues / Not Fixed

No findings were rejected. All 6 findings were implemented.
