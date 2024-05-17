# Securing Prometheus

A guide for setting up a secure communication channel between Prometheus and its target (node exporter) using TLS certificate and basic authentication.

## Target Setup

### 1. Generate the Certificate and the private Key

Commands to generate a self-signed certificate and a private key for the node exporter:

```sh
sudo openssl genrsa -out /etc/node_exporter/node_exporter.key 2048
sudo openssl req -new -key /etc/node_exporter/node_exporter.key -out /etc/node_exporter/node_exporter.csr
sudo openssl x509 -req -days 365 -in /etc/node_exporter/node_exporter.csr -signkey /etc/node_exporter/node_exporter.key -out /etc/node_exporter/node_exporter.crt
```

### 2. Generate a Password for Basic Authentication

Install apache2-utils and create a hashed password for basic authentication:

```sh
sudo apt-get update && sudo apt install apache2-utils –y
htpasswd -nBC 12 "" | tr -d ':\n'
```

### 3. Create `web_config.yml` for Node Exporter

Create  `/etc/node_exporter/web_config.yml`:

```yaml
tls_server_config:
  cert_file: node_exporter.crt
  key_file: node_exporter.key
basic_auth_users:
  prometheus: $2y$12$YzsQ3SoKqeIOmPUZKPje5euzz9fK8qs/oXYX7wgJBlhczW9EZzXmW
```

### 4. Change ownership of the /etc/node_exporter directory

```sh
sudo chown -R node_exporter:node_exporter /etc/node_exporter
```

### 5. Update the systemd service of node_exporter

```sh
sudo vim /etc/systemd/system/node_exporter.service
```
```ini
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target


[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter --web.config.file="/etc/node_exporter/web_config.yml"


[Install]
WantedBy=multi-user.target
```

### 6. Reload Daemon and Restart the node_exporter service
```sh
sudo systemctl daemon-reload
sudo systemctl restart node_exporter
```

## Prometheus server Setup

### 1. copy node_exporter certificate to prometheus server

get the `node_exporter.crt` to the server via SFTP at '/etc/prometheus/node_exporter.crt'

### 1. Edit the Prometheus Configuration File

append to `/etc/prometheus/prometheus.yaml` the next configuration

```yaml
  - job_name: 'Linux Server'
    scheme: https
    tls_config:
      ca_file: /etc/prometheus/node_exporter.crt
      insecure_skip_verify: true
    basic_auth:
      username: prometheus
      password: rana
    static_configs:
      - targets: ['192.168.1.17:9100']
```

### 2. Restart Prometheus service

Restart the Prometheus service:

```sh
sudo systemctl restart prometheus
```

Then check that the target state is up

