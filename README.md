# Hotspotplus

A microservices-based WiFi hotspot management platform for ISPs and businesses. It handles user authentication, captive portal login, billing, and deep network flow analytics.

## Architecture

| Service | Tech | Purpose |
|---|---|---|
| **API** | Node.js / LoopBack 3 | Core REST API — users, billing, hotspot config |
| **Dashboard** | AngularJS | Admin UI for businesses and ISPs |
| **Hotspot Portal** | AngularJS | Captive portal for WiFi users |
| **Log Worker** | TypeScript / Express | Consumes Kafka logs, generates DNS/NetFlow/proxy reports |
| **GoFlow** | Go | NetFlow v9 / IPFIX / sFlow collector |
| **ClickHouse Sinker** | Go | Kafka → ClickHouse data pipeline |
| **RADIUS** | FreeRADIUS 3.0.19 | Network authentication (802.1x / MAC auth) |
| **MongoDB** | — | Primary database |
| **Redis** | — | Caching layer |
| **Kafka + Zookeeper** | Bitnami 2.3.0 | Event streaming between services |
| **ClickHouse** | 19.17.5 | High-performance analytics for network flows |
| **Traefik** | v2.0 | Reverse proxy and SSL termination |

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
