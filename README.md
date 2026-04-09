# Hotspotplus

A microservices-based WiFi hotspot management platform for ISPs and businesses. It handles user authentication via a captive portal, RADIUS-based network access control, billing, and deep network flow analytics.

## How It Works

The system has three independent data pipelines running in parallel:

**1. Authentication pipeline** — a user opens a browser, gets redirected to the captive portal, logs in, and the API validates their quota before instructing RADIUS to grant or deny network access.

**2. Session pipeline** — every RADIUS accounting event (start/update/stop) is written to MongoDB, cached in Redis, and streamed to Kafka. The log-worker consumes those Kafka messages and inserts them into ClickHouse for analytics.

**3. Network flow pipeline** — routers send NetFlow/IPFIX/sFlow packets to GoFlow, which decodes them and publishes JSON messages to Kafka. ClickHouse Sinker consumes those messages and bulk-inserts them into ClickHouse for traffic analysis and reporting.

### High-level diagram

```
                         ┌─────────────┐
  WiFi User ─────────►  │   Hotspot   │  (captive portal)
                         │   Portal    │
                         └──────┬──────┘
                                │ REST
                                ▼
                         ┌─────────────┐     ┌─────────┐
                         │     API     │────►│ MongoDB │  (users, plans, billing)
                         │  (LoopBack) │     └─────────┘
                         └──┬──────┬───┘
                            │      │          ┌─────────┐
                RADIUS auth │      └─────────►│  Redis  │  (session cache, usage)
                            ▼                 └─────────┘
                    ┌──────────────┐
                    │    RADIUS    │  ◄──── NAS / Access Point
                    │ (FreeRADIUS) │
                    └──────────────┘

  Accounting events ──► API ──► Kafka ──► Log Worker ──► ClickHouse
                                    ▲
  NetFlow/IPFIX/sFlow ──► GoFlow ───┘          (analytics & reports)

  Admin / ISP ──► Dashboard ──► API
```

## Architecture

| Service | Tech | Purpose |
|---|---|---|
| **API** | Node.js / LoopBack 3 | Core REST API — users, billing, hotspot config, RADIUS adapter |
| **Dashboard** | AngularJS | Admin UI for businesses and ISPs |
| **Hotspot Portal** | AngularJS | Captive portal shown to WiFi users |
| **Log Worker** | TypeScript / Express | Consumes Kafka, inserts into ClickHouse, serves reports |
| **GoFlow** | Go | NetFlow v9 / IPFIX / sFlow collector |
| **ClickHouse Sinker** | Go | Kafka → ClickHouse bulk-insert pipeline |
| **RADIUS** | FreeRADIUS 3.0.19 | Network authentication and accounting |
| **MongoDB** | — | Transactional data: users, plans, invoices, NAS config |
| **Redis** | — | Session cache and real-time usage counters |
| **Kafka + Zookeeper** | Bitnami 2.3.0 | Message bus between API, GoFlow, and Log Worker |
| **ClickHouse** | 19.17.5 | Columnar analytics database for sessions and NetFlow |
| **Traefik** | v2.0 | Reverse proxy and SSL termination |

## Data Flows in Detail

### WiFi authentication

```
User browser
  │
  ├─► GET wifi.your_domain  →  Captive Portal (Hotspot)
  │
  └─► POST /login  →  API: Member.signIn()
        │
        ├─► Check business subscription (MongoDB)
        ├─► Look up member (MongoDB, cached in Redis)
        ├─► Check plan validity (subscription window)
        ├─► Calculate remaining data quota
        │     └─► Query ClickHouse Usage view
        │         + add Redis in-flight usage
        ├─► Check remaining time quota
        └─► Check for concurrent sessions (Redis)
              │
              ├── Denied ──► HTTP 4xx to portal
              └── Approved ──► RADIUS receives:
                                - session timeout (seconds remaining)
                                - bandwidth limits (kbps + burst)
                                - accounting update interval
```

The NAS (router/access point) talks directly to RADIUS using the standard RADIUS protocol. The API acts as a RADIUS back-end: FreeRADIUS forwards every `Authorize` and `Post-Auth` request to the API via HTTP, and the API returns the appropriate RADIUS reply attributes.

### Session tracking

Every accounting packet from the NAS (Start / Interim-Update / Stop) hits the RADIUS server, which forwards it to the API:

```
NAS  →  RADIUS  →  API: radiusAccounting()
                     │
                     ├─► Write session to MongoDB
                     ├─► Update usage counters in Redis  (HINCRBY)
                     └─► Publish session JSON to Kafka topic: hotspotplus_sessions
                               │
                               └─► Log Worker consumes  →  INSERT into ClickHouse Session table
```

Redis usage counters serve as an in-memory accumulator between accounting intervals, so the API doesn't need to query ClickHouse on every authentication check.

### Network flow analytics

```
Router / Switch
  │  NetFlow v9 / IPFIX / sFlow (UDP)
  ▼
GoFlow (Go)
  │  Decodes binary flow records
  │  Converts to JSON FlowMessage (src/dst IP, ports, bytes, proto, timestamps)
  ▼
Kafka  (netflow topic)
  │
  ├─► ClickHouse Sinker  →  bulk INSERT into ClickHouse NetflowReport table
  │
  └─► Log Worker  →  serves query API for traffic reports
```

GoFlow supports NetFlow v9, IPFIX, and sFlow v5. It serialises decoded flow records to Protobuf internally and then emits JSON to Kafka. ClickHouse Sinker handles batching and retries so flow data survives transient ClickHouse downtime.

### Analytics data model

ClickHouse holds four tables used for reporting:

| Table | Populated by | Used for |
|---|---|---|
| `Session` | Log Worker (from Kafka) | Session history, per-user totals |
| `Usage` | Materialized view over `Session` | Fast quota lookup during auth |
| `NetflowReport` | ClickHouse Sinker (from GoFlow) | Traffic analysis, IP-level reports |
| `DnsReport` | Log Worker | DNS query logs per user |
| `WebProxy` | Log Worker | Web access logs (joined to Session by IP) |

Reports (JSON, CSV, Excel) are generated by the Log Worker by querying these tables and rendering results through jsreport templates. The API proxies report requests from the Dashboard to the Log Worker.

## Prerequisites

- Docker and Docker Compose — [install guide](https://docs.docker.com/engine/install/) / [post-install](https://docs.docker.com/engine/install/linux-postinstall/)
- Four DNS A records pointing to your server IP:

| Description | Domain |
|---|---|
| Main domain | `your_domain` |
| Hotspot portal | `wifi.your_domain` |
| Admin dashboard | `my.your_domain` |
| API | `api.your_domain` |

## Production Deployment

### 1. Server setup

SSH into your server as root and create a dedicated user:

```bash
ssh root@your_server_ip
adduser hotspotplus
usermod -aG sudo hotspotplus
su hotspotplus
sudo ufw disable
```

Increase kernel limits by adding the following to `/etc/sysctl.conf`:

```text
vm.max_map_count = 300000
fs.file-max = 70000
```

### 2. Clone and initialize

```bash
cd ~
git clone https://github.com/parmenides/hotspotplus.git
docker swarm init
```

### 3. Set up Portainer

```bash
docker volume create portainer_data
docker run -d \
  -p 8001:8000 -p 9001:9000 \
  --name=portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce
```

Open `http://your_server_ip:9001`, create an admin account, and select **Docker**.

### 4. Deploy the stack

1. Go to **Stacks** → **Add Stack** → **Web Editor**
2. Open [`config/docker-compose-swarm.yml`](https://github.com/parmenides/hotspotplus/blob/master/config/docker-compose-swarm.yml), replace all occurrences of `your_domain` with your real domain, and paste the contents into the editor
3. Add the following environment variables:

| Variable | Required | Description |
|---|---|---|
| `project_dir` | Yes | Absolute path to the cloned repo (e.g. `/home/hotspotplus/hotspotplus`) |
| `admin_username` | Yes | Administrator username |
| `admin_password` | Yes | Administrator password |
| `encryption_key` | Yes | Strong random string used for encryption |
| `panel_address` | Yes | Dashboard URL (e.g. `http://my.your_domain`) |
| `your_domain` | Yes | Main domain URL (e.g. `http://your_domain`) |
| `radius_shred_secret` | Yes | Shared secret for RADIUS communication |
| `radius_ip` | Yes | Server IP address used by RADIUS |
| `payment_api_key` | Optional | PayPing OAuth2.0 API key |
| `payping_client_id` | Optional | PayPing OAuth2.0 client ID |
| `payping_app_token` | Optional | PayPing OAuth2.0 app token |
| `sms_api_token` | Optional | Kavehnegar SMS API token |
| `sms_signature` | Optional | SMS sender signature |

4. Click **Deploy the stack**

### 5. SMS templates (optional)

To enable SMS notifications, create a [Kavehnegar](https://kavenegar.com/) account and add the following [Verification Patterns](https://panel.kavenegar.com/client/Verification) using the templates from [`config/smsTemplates`](https://github.com/parmenides/hotspotplus/blob/master/config/smsTemplates):

| Pattern Name |
|---|
| `businessSmsChargePurchaseConfirmed` |
| `hotspotPlusHotspotCredentials` |
| `hotspotPlusRegistrationSMS` |
| `passwordReset` |
| `sendVerificationCodeCallOnly` |
| `sendVerificationCodeThenCall` |

## Development

```bash
# First-time only: create the shared Docker network
docker network create hotspotplusgate

cd hotspotplus
npm install
npm start
# or: docker-compose down && docker-compose up
```

## Access

| Interface | URL |
|---|---|
| Admin dashboard | `http://my.your_domain/src/#/access/signin` |
| Hotspot portal | `http://wifi.your_domain` |
| API explorer | `http://api.your_domain/explorer` |
