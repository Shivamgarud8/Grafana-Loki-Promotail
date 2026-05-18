# 📊 Grafana Monitoring Stack

> **Full Observability Stack** — Grafana + Loki + Promtail + Prometheus + cAdvisor

A complete, production-ready monitoring and observability solution for containerized environments. Collect metrics, aggregate logs, and visualize everything in one unified dashboard.

---

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        GRAFANA DASHBOARD                            │
│                     (Visualization Layer)                           │
│                      http://localhost:3000                          │
└───────────────────────────┬─────────────────────┬───────────────────┘
                            │                     │
              ┌─────────────▼──────┐   ┌──────────▼──────────┐
              │     PROMETHEUS     │   │        LOKI         │
              │  (Metrics Store)   │   │    (Log Store)      │
              │ http://localhost   │   │  http://localhost   │
              │      :9090         │   │      :3100          │
              └─────────┬──────────┘   └──────────┬──────────┘
                        │                         │
           ┌────────────▼──────────┐   ┌──────────▼──────────┐
           │       cADVISOR        │   │      PROMTAIL       │
           │  (Container Metrics)  │   │   (Log Collector)   │
           │ http://localhost:8080 │   │   (Sidecar Agent)   │
           └────────────┬──────────┘   └──────────┬──────────┘
                        │                         │
           ┌────────────▼──────────────────────────▼──────────┐
           │              DOCKER HOST / CONTAINERS             │
           │         Redis  •  App Containers  •  System       │
           └───────────────────────────────────────────────────┘
```

### Component Roles

| Component | Role | Port | Data Type |
|-----------|------|------|-----------|
| **Grafana** | Dashboard & visualization UI | `3000` | — |
| **Prometheus** | Time-series metrics database | `9090` | Metrics |
| **cAdvisor** | Container resource metrics exporter | `8080` | Metrics |
| **Loki** | Log aggregation & storage | `3100` | Logs |
| **Promtail** | Log collector & shipper agent | — | Logs |
| **Redis** | Demo workload (cAdvisor target) | `6379` | — |

---

## 📋 Prerequisites

- Ubuntu / Debian Linux (for Grafana native install)
- Docker & Docker Compose installed
- `wget` and `curl` available
- Ports `3000`, `3100`, `8080`, `9090`, `6379` free

---

## 🚀 Setup Guide

### Part 1 — Install Grafana (Native on Ubuntu/Debian)

```bash
# Install dependencies
sudo apt-get install -y apt-transport-https
sudo apt-get install -y software-properties-common wget

# Add Grafana GPG key
sudo wget -q -O /usr/share/keyrings/grafana.key https://apt.grafana.com/gpg.key

# Add the stable APT repository
echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://apt.grafana.com stable main" \
  | sudo tee -a /etc/apt/sources.list.d/grafana.list

# (Optional) Add the beta APT repository
# echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://apt.grafana.com beta main" \
#   | sudo tee -a /etc/apt/sources.list.d/grafana.list

# Update package list and install Grafana
sudo apt-get update
sudo apt-get install grafana

# Enable and start the Grafana server
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server

# Check status
sudo systemctl status grafana-server
```

**Access Grafana:** Open `http://localhost:3000`  
**Default credentials:** `admin` / `admin` (you'll be prompted to change on first login)

---

### Part 2 — Install Loki & Promtail (Docker)

```bash
# Download Loki configuration
wget https://raw.githubusercontent.com/grafana/loki/v2.8.0/cmd/loki/loki-local-config.yaml \
  -O loki-config.yaml

# Start Loki container
docker run -d \
  --name loki \
  -v $(pwd):/mnt/config \
  -p 3100:3100 \
  grafana/loki:2.8.0 \
  --config.file=/mnt/config/loki-config.yaml

# Download Promtail configuration
wget https://raw.githubusercontent.com/grafana/loki/v2.8.0/clients/cmd/promtail/promtail-docker-config.yaml \
  -O promtail-config.yaml

# Start Promtail container (reads /var/log and ships to Loki)
docker run -d \
  --name promtail \
  -v $(pwd):/mnt/config \
  -v /var/log:/var/log \
  --link loki \
  grafana/promtail:2.8.0 \
  --config.file=/mnt/config/promtail-config.yaml
```

> **What Promtail does:** Promtail tails log files from `/var/log` on the host and ships them to Loki in real time. It attaches labels (host, job, filename) so you can filter and query logs in Grafana.

---

### Part 3 — Install Prometheus + cAdvisor (Docker Compose)

#### 3a. Download Prometheus config

```bash
wget https://raw.githubusercontent.com/prometheus/prometheus/main/documentation/examples/prometheus.yml
```

#### 3b. Add cAdvisor scrape target to `prometheus.yml`

Append this block to the `scrape_configs` section:

```yaml
scrape_configs:
  - job_name: cadvisor
    scrape_interval: 5s
    static_configs:
      - targets:
          - cadvisor:8080
```

#### 3c. Create `docker-compose.yml`

```yaml
version: '3.2'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    command:
      - --config.file=/etc/prometheus/prometheus.yml
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    depends_on:
      - cadvisor

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    depends_on:
      - redis

  redis:
    image: redis:latest
    container_name: redis
    ports:
      - "6379:6379"
```

#### 3d. Start the stack

```bash
docker-compose up -d

# Verify all containers are running
docker-compose ps
```

---

## 🔌 Connect Data Sources in Grafana

1. Open **Grafana** at `http://localhost:3000`
2. Go to **Configuration → Data Sources → Add data source**

### Add Prometheus
- Type: `Prometheus`
- URL: `http://localhost:9090`
- Click **Save & Test**

### Add Loki
- Type: `Loki`
- URL: `http://localhost:3100`
- Click **Save & Test**

---

## 🔍 Test with PromQL Queries

Open Prometheus at `http://localhost:9090/graph` and try:

### CPU Usage Rate for Redis container
```promql
rate(container_cpu_usage_seconds_total{name="redis"}[1m])
```

### Memory Usage for Redis container
```promql
container_memory_usage_bytes{name="redis"}
```

### All running containers — CPU usage
```promql
rate(container_cpu_usage_seconds_total[1m])
```

### Container network received bytes
```promql
rate(container_network_receive_bytes_total[1m])
```

---

## 📁 Project File Structure

```
monitoring-stack/
├── docker-compose.yml        # Prometheus + cAdvisor + Redis
├── prometheus.yml            # Prometheus scrape config
├── loki-config.yaml          # Loki server config
├── promtail-config.yaml      # Promtail log shipper config
└── README.md                 # This file
```

---

## 🌐 Service URLs Summary

| Service | URL |
|---------|-----|
| Grafana UI | http://localhost:3000 |
| Prometheus UI | http://localhost:9090 |
| cAdvisor UI | http://localhost:8080 |
| Loki API | http://localhost:3100 |
| Redis | localhost:6379 |

---

## 🛠️ Useful Commands

```bash
# View all running containers
docker ps

# View logs for a specific container
docker logs loki
docker logs promtail

# Stop the Docker Compose stack
docker-compose down

# Stop and remove volumes
docker-compose down -v

# Restart Grafana service
sudo systemctl restart grafana-server

# Check Grafana logs
sudo journalctl -u grafana-server -f
```

---

---
👩‍🏫 **Guided and Supported by [Trupti Mane Ma’am](https://github.com/iamtruptimane)**  
---

👨‍💻 **Developed By:**  
**Shivam Garud**  
🧠 *DevOps & Cloud Engineer*  
💼 *DevOps Engineer | CI/CD | Docker | Kubernetes | Terraform | Ansible | AWS | Linux | Cloud Automation | Infrastructure as Code!*  
🌐 [GitHub Profile](https://github.com/Shivamgarud8)
🌐 [Medium blog](https://medium.com/@shivam.garud2011)
🌐 [linkedin](www.linkedin.com/in/shivam-garud)
🌐 [portfolio](https://shivam-garud.vercel.app/)


