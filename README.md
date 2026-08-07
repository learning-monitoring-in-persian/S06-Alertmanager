[English](README.md) | [فارسی](README-persian.md)

# Set up Alertmanager

Alertmanager handles alerts sent by client applications such as the Prometheus server. It takes care of deduplicating, grouping, and routing them to the correct receiver integration such as email, Slack, or webhooks. It also takes care of silencing and inhibition of alerts.

> [!NOTE]
> While Prometheus is the most common source of alerts, it is not the only one! Other systems like **Grafana Loki** (which we will cover in future tutorials) can evaluate logs and send alerts directly to Alertmanager as well.

> [!NOTE]
> If you plan to install **Alertmanager** on a machine that already has **Docker**, or if you want to run tools like **Prometheus** alongside it, I recommend using the **Docker** version of Alertmanager. This keeps your setup more flexible and it makes the maintenance of Alertmanager easier in the future.
>
> However, if the machine will only run Alertmanager and doesn't have Docker, it's better to avoid extra overhead and install the Alertmanager binary directly.

---

## Download and extract

First, download the pre-compiled binary for Linux:

```bash
VERSION=$(curl -s https://api.github.com/repos/prometheus/alertmanager/releases/latest | grep '"tag_name"' | cut -d'"' -f4)
wget -O alertmanager.tar.gz https://github.com/prometheus/alertmanager/releases/download/${VERSION}/alertmanager-${VERSION#v}.linux-amd64.tar.gz
tar -xvf alertmanager.tar.gz
```

You can find a default `alertmanager.yml` config file inside the extracted folder.
If your machine restarts, the process will stop. To run Alertmanager as a background service, follow the steps below.

## Run Alertmanager as a systemd service

### 1) Create user & directories

For security, create a dedicated system user:
```bash
sudo useradd --no-create-home --shell /usr/sbin/nologin alertmanager
sudo mkdir -p /etc/alertmanager /var/lib/alertmanager
```

### 2) Move binary

Move the binaries to the system's bin path:
```bash
sudo mv alertmanager-*/alertmanager /usr/local/bin/
sudo mv alertmanager-*/amtool /usr/local/bin/
rm -rf alertmanager*
```

### 3) Create configuration file

Create `/etc/alertmanager/alertmanager.yml` with a basic configuration (this routes alerts to a local webhook as an example):
```yaml
route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h
  receiver: 'web.hook'

receivers:
  - name: 'web.hook'
    webhook_configs:
      - url: 'http://127.0.0.1:5001/'
        send_resolved: true
        username: Infrustructure Alerting
```

> [!NOTE]
> Alertmanager has a vast set of configuration options for different receivers (Slack, Email, PagerDuty, Webhooks, etc.), complex routing trees, and inhibition rules. You can find the complete list of configuration parameters and their descriptions in the official [Alertmanager Configuration Documentation](https://prometheus.io/docs/alerting/latest/configuration/).

Set the correct permissions:
```bash
sudo chown -R alertmanager:alertmanager /etc/alertmanager /var/lib/alertmanager
```

### 4) Create systemd service

Create a new file at `/etc/systemd/system/alertmanager.service`:
```ini
[Unit]
Description=Alertmanager
Wants=network-online.target
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager

Restart=always

[Install]
WantedBy=multi-user.target
```

Reload systemd and start the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable alertmanager
sudo systemctl start alertmanager
```

---

## Run Alertmanager as a Docker container (recommended)

If you prefer using Docker, you can easily deploy Alertmanager as a container. Create a `docker-compose.yml`:

```yaml
services:
  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    restart: unless-stopped
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
```

Make sure you have your `alertmanager.yml` in the same directory, then run:
```bash
docker compose up -d
```

---

## Configure Prometheus to send alerts

By default, Prometheus does not know where to send the alerts it fires. You must configure it to point to Alertmanager.

Edit your `prometheus.yml` and add the `alerting` block:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

# Add this block to point to the Alertmanager
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # Replace with your Alertmanager IP if it's on a different machine
          - localhost:9093

rule_files:
  - "rules/*.yml"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
```

Restart Prometheus to apply the changes.

---

## Integration with Grafana

Grafana can connect to Alertmanager as a data source. This allows you to view and manage your alerts and silences directly from the Grafana UI.

### Graphic Method (Via UI)
1. Open Grafana.
2. Go to **Connections -> Data sources**.
3. Click **Add data source** and select **Alertmanager**.
4. In the HTTP URL field, enter `http://localhost:9093` (or the IP of your Alertmanager instance).
5. Scroll down and click **Save & test**.

### File Provisioning Method (Docker/Automated)
If you are managing Grafana via files (e.g., in a Docker Compose setup), you can provision the Alertmanager data source automatically.

Create a file named `alertmanager.yaml` inside your Grafana provisioning directory (e.g., `./grafana/provisioning/datasources/`):

```yaml
apiVersion: 1

datasources:
  - name: Alertmanager
    type: alertmanager
    access: proxy
    url: http://alertmanager:9093 # Use ip_address:port if not in the same Docker network
    isDefault: false
    jsonData:
      implementation: prometheus
```

Restart Grafana to apply the provisioning.
