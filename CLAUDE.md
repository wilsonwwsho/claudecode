# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**OSSIEM** is a containerized, lab-focused Open Source SIEM stack orchestrated via Docker Compose. It integrates Wazuh (HIDS/alerting), Graylog (log aggregation), Grafana (visualization), Velociraptor (DFIR), and CoPilot (SOCFortress unified platform). It is intended for lab/testing environments, not production.

## Deployment Commands

### Pre-Deployment (run once)

Generate Wazuh SSL certificates:
```bash
docker compose -f wazuh/generate-indexer-certs.yml run --rm generator
```

Build the custom Wazuh Manager image (from `wazuh/custom-wazuh-manager/`):
```bash
docker build -t socfortress/wazuh-manager:4.9.0 --build-arg WAZUH_VERSION=4.9.0 --build-arg WAZUH_TAG_REVISION=1 .
```

Fix Graylog file ownership before first start (required — Graylog runs as UID 1100):
```bash
# See graylog/README.md for the exact chown commands
```

### Deployment

```bash
docker compose up -d        # Start all services
docker compose down         # Stop all services
docker compose logs -f      # Follow logs for all services
docker compose logs -f <service>  # Follow logs for a specific service (e.g., graylog, wazuh.manager)
```

### Post-Deployment Integration Steps

Install SOCFortress Wazuh rules inside the manager container:
```bash
docker exec -it wazuh.manager /bin/bash
dnf install git -y
curl -so ~/wazuh_socfortress_rules.sh https://raw.githubusercontent.com/socfortress/OSSIEM/main/wazuh_socfortress_rules.sh && bash ~/wazuh_socfortress_rules.sh
```

Import Wazuh root CA into Graylog's Java keystore (required for Graylog ↔ Wazuh Indexer TLS):
```bash
docker exec -it graylog bash
cp /opt/java/openjdk/lib/security/cacerts /usr/share/graylog/data/config/
cd /usr/share/graylog/data/config/
keytool -importcert -keystore cacerts -storepass changeit -alias wazuh_root_ca -file root-ca.pem
```

Generate Velociraptor API config (for CoPilot integration):
```bash
docker exec -it velociraptor /bin/bash
./velociraptor --config server.config.yaml config api_client --name admin --role administrator,api api.config.yaml
```

Retrieve CoPilot initial admin password (only available on first startup):
```bash
docker logs "$(docker ps --filter ancestor=ghcr.io/socfortress/copilot-backend:latest --format "{{.ID}}")" 2>&1 | grep "Admin user password"
```

## Architecture

### Service Map

| Service | Image | Ports | Role |
|---|---|---|---|
| `wazuh.manager` | Custom (`socfortress/wazuh-manager:4.9.0`) | 55000 (API), 1514/1515 (agents) | HIDS agent manager & alerting |
| `wazuh.indexer` | Official Wazuh | 9200 | OpenSearch-based log indexing |
| `wazuh.dashboard` | Official Wazuh | 5601 | Security monitoring UI |
| `graylog` | `graylog/graylog:6.0.6` | 9000 (UI), 12201 (GELF), 514/2514 (syslog) | Log aggregation & enrichment |
| `mongodb` | `mongo:6.0.14` | internal | Graylog config/session store |
| `grafana` | Grafana Enterprise | 3000 | Advanced dashboards |
| `velociraptor` | Official | 8000, 8001, 8889 | Digital forensics & DFIR |
| `copilot-backend` | `ghcr.io/socfortress/copilot-backend:latest` | 5000 | REST API, orchestration |
| `copilot-frontend` | `ghcr.io/socfortress/copilot-frontend:latest` | 80, 443 | Web UI |
| `copilot-mysql` | `mysql:8.0.38` | internal | CoPilot database |
| `copilot-minio` | MinIO | internal | S3-compatible object storage |

### Log Flow

```
Wazuh Agents → wazuh.manager → Fluent Bit → Graylog (GELF TCP :5555)
                             ↓
                       wazuh.indexer ← Graylog (OpenSearch API :9200, TLS)
                             ↓
                      wazuh.dashboard
```

Fluent Bit replaces Filebeat in the custom Wazuh Manager image. Its config lives at `wazuh/custom-wazuh-manager/config/fluent-bit.conf`.

### Custom Wazuh Manager Image

The custom image (`wazuh/custom-wazuh-manager/Dockerfile`) is built on Amazon Linux 2023 and uses **s6-overlay** for process supervision. It adds Fluent Bit alongside the standard Wazuh Manager process to ship JSON alerts to Graylog instead of using Filebeat.

Key directories inside the image:
- `config/etc/services.d/` — s6 service definitions (fluent-bit, ossec-logs)
- `config/etc/cont-init.d/` — initialization scripts run at container start

### Configuration Files (templates — must be customized)

| File | Purpose |
|---|---|
| `.env` | All service credentials, API keys, hostnames — primary configuration point |
| `wazuh/config/wazuh_cluster/wazuh_manager.conf` | Wazuh Manager: rules, SCA, vulnerability detection, syscollector |
| `wazuh/config/wazuh_indexer/wazuh.indexer.yml` | OpenSearch: cluster, TLS, security plugin |
| `wazuh/config/wazuh_dashboard/wazuh.yml` | Dashboard → Manager API connection |
| `graylog/graylog.conf` | Graylog: MongoDB URI, password secret, GeoIP paths, indexer connection |
| `wazuh/config/wazuh_indexer/internal_users.yml` | OpenSearch user/password hashes |

### GeoIP Enrichment

Graylog includes `GeoLite2-City.mmdb` and `GeoLite2-ASN.mmdb` for IP geolocation. These are large binary files (~70 MB total) committed to the repo. Paths are configured in `graylog/graylog.conf`.

## Key Constraints

- **Certificates must exist before first `docker compose up`** — services will fail to start without them. Copy `root-ca.pem` into `graylog/` after cert generation.
- **Graylog file ownership** — Graylog runs as UID 1100. The `graylog/` subdirectory contents must be owned by that UID before the container starts (see `graylog/README.md`).
- **Wazuh Dashboard health check error** — The `wazuh-alerts` index doesn't exist until Graylog is integrated with the Wazuh Indexer. This is expected on first boot.
- **CoPilot admin password** — Only logged once on first startup. Retrieve it immediately after first launch.
