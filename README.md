# Setting Up Grafana Cloud with Prometheus on AWS

**Prometheus Installation**

- Create a Prometheus VM instance
- Name: Prometheus
- AMI: Ubuntu 24.04
- Instance type: t2.micro
- Key pair: Select a keypair
- Security Group (Eit/Open): 9090, 9100, 80 and 22 to 0.0.0.0/0
- IAM instance profile: Select the AWS-EC2FullAccess-Role
- Launch Instance

**SSH into Prometheus**

- run this command : 

```bash
sudo useradd --system --no-create-home --shell /bin/false prometheus wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
  ```

Extract Prometheus files, move them, and create directories:

```bash
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
cd prometheus-2.47.1.linux-amd64/
sudo mkdir -p /data /etc/prometheus
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
```

Set ownership for directories:

```bash
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
```

Create a systemd unit configuration file for Prometheus:

```bash
sudo nano /etc/systemd/system/prometheus.service
```

Add the following content to the prometheus.service file:

```plaintext
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
```

Enable and start Prometheus:

 ```bash
sudo systemctl enable prometheus
sudo systemctl start prometheus
```

Verify Prometheus's status:

```bash
sudo systemctl status prometheus
 ```

You can access Prometheus in a web browser using your server's IP and port 9090:

 `http://<your-server-ip>:9090`

**Installing Node Exporter**

```bash
sudo useradd --system --no-create-home --shell /bin/false node_exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
 ```

Extract Node Exporter files, move the binary, and clean up:

```bash
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter*
 ```

Create a systemd unit configuration file for Node Exporter:

 ```bash
sudo nano /etc/systemd/system/node_exporter.service
 ```
 
Add the following content to the node_exporter.service file:

 ```plaintext
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
 ```

Enable and start Node Exporter:

 ```bash
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
 ```

Verify the Node Exporter's status:

 ```bash
sudo systemctl status node_exporter
 ```

You can access Node Exporter metrics in Prometheus.


**Prometheus Configuration**

To configure Prometheus to scrape metrics from Node Exporter.

 ```bash
cd /etc/prometheus
sudo nano prometheus.yml
```

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
```

Check the validity of the configuration file:

 ```bash
promtool check config /etc/prometheus/prometheus.yml
 ```  

Reload the Prometheus configuration without restarting:

 ```bash
  curl -X POST http://localhost:9090/-/reload
 ```

You can access Prometheus targets at:

`http://<your-prometheus-ip>:9090/targets`


**Sign Up for Grafana Cloud**

- Go to the Grafana Cloud website: https://grafana.com/cloud/
- Click on "Create Free Account" and follow the registration process.
- Verify your email address and log in to your Grafana Cloud account.


**Add Prometheus as a Data Source**

- Click on "Data Sources".
- Click on "Add data source".
- Select "Prometheus" as the data source type.
- Enter the URL of your Prometheus server (e.g. http://localhost:9090).
- ⁠Replace localhost with Prometheus public ip
- Click "Save & Test".


**Go to Dashboard**
- Click on New and Click Import.
- Paste the Node Exporter ID (1860), then click on Load.


**Install Stress in the Prometheus Server to Simulate stress on the Server**

```bash
sudo apt update
sudo apt install -y stress
```
```bash
stress --cpu 4 --timeout 60s
```
```bash
stress --vm 2 --vm-bytes 256M --timeout 60s
```
```bash
stress --io 4 --timeout 10s
```


After Installing Stress, Go back to your Grafana Dashboard to see the CPU utilization.
