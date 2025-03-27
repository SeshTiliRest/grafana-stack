# Grafana Logs Monitoring Stack

### Logs → Alloy → Loki → Grafana

## Overview

TL;DR I have a Windows virtual machine with two applications: App1, which generates daily log files (e.g., `2025-01-01.log`), and App2, which writes logs to a single file (`message.log`).

The stack consists of [Grafana](https://github.com/grafana/grafana), [Loki](https://github.com/grafana/loki), and [Grafana Alloy](https://github.com/grafana/alloy) to monitor these log files.

Additionally, [Prometheus](https://github.com/prometheus/prometheus) is included with a configuration in case you want to use [windows_exporter](https://github.com/prometheus-community/windows_exporter) or [another exporter](https://prometheus.io/docs/instrumenting/exporters/) for monitoring in Grafana.

<details>
  <summary>App1 log example (2025-01-01.log)</summary>

```log
2025-01-01 12:00:59.109 FATAL  ComponentName Message
2025-01-01 13:00:59.109 ERROR  ComponentName Message
2025-01-01 14:00:59.109 WARN   ComponentName Message
2025-01-01 15:00:59.109 INFO   ComponentName Message
```

</details>

<details>
  <summary>App2 log example (message.log)</summary>

```log
01.01.25 12:00:59 - Message
01.01.25 13:00:59 - Message
01.01.25 14:00:59 - Message
01.01.25 15:00:59 - Message
```

</details>

## Setup Instructions

<details>
  <summary><strong>1. (Optional) Install WSL for testing on Windows 11</strong></summary>

#### In case you want to test this setup on your local Windows, install WSL (Windows Subsystem for Linux) by opening PowerShell in Windows Terminal:

#### Check if WSL is installed:
```powershell
wsl -v
```

#### Install WSL if not already installed:
```powershell
wsl --install
```

#### List available distributions:
```powershell
wsl --list --online
```
or for short:
```powershell
wsl -l -o
```

#### List installed distributions:
```powershell
wsl -l
```

#### Install Ubuntu or another distribution:
```powershell
wsl --install -d Ubuntu-24.04
```

#### After installation, update the system:
```sh
sudo apt update && sudo apt upgrade
```

#### Install Git:
```sh
sudo apt install git
```

**Official Documentation:**  
[WSL Installation Guide](https://learn.microsoft.com/ru-ru/windows/wsl/install)

</details>

<details>
  <summary><strong>2. Install Docker</strong></summary>

#### You need Docker to deploy this stack. If you don't have Docker already, install it by following the instructions below:

#### Add Docker’s official GPG key:
```sh
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

#### Add Docker repository to apt sources:
```sh
echo \  
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \  
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \  
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

#### Install Docker:
```sh
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

#### Verify installation:
```sh
sudo docker run hello-world
```

#### Check running containers:
```sh
sudo docker ps
```

#### Check Docker status:
```sh
sudo systemctl status docker
```

#### Add user to Docker group (to run without sudo):
```sh
sudo usermod -aG docker $USER
```

#### Apply group changes without logging out:
```sh
newgrp docker
```

**Official Documentation:**  
[Docker Installation Guide](https://docs.docker.com/engine/install/)

</details>

<details>
  <summary><strong>3. (Optional) Install Portainer</strong></summary>

#### Portainer is a web-based UI for managing Docker containers.

#### Create a volume for Portainer:
```sh
docker volume create portainer_data
```

#### Install Portainer:
```sh
docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data portainer/portainer-ce:lts
```

#### Check running containers:
```sh
docker ps
```

#### Check IP address:
```sh
ifconfig
```

If `ifconfig` is not available, install `net-tools`:
```sh
sudo apt install net-tools
```

#### Access Portainer:
Open a web browser and navigate to:
```
https://<IP>:9443
```

**Official Documentation:**  
[Portainer Installation Guide](https://docs.portainer.io/start/install-ce/server/docker/linux)
</details>

<details>
  <summary><strong>4. Install and configure Grafana Alloy on machine with logs</strong></summary>

#### [Download and install Grafana Alloy.](https://grafana.com/docs/alloy/latest/set-up/install/windows/)
#### Edit `config.alloy` in the Alloy configuration directory:
```powershell
ii "C:\Program Files\GrafanaLabs\Alloy\"
```

#### NOTE
Change path to logs folder as needed:

  ```
  // Define the path to App1 log files using glob patterns to match all .log files in the specified directory
  path_targets = [{"__path__" = "C:/App1/Folder/logs/Default/*.log"}]
  ```

Modify the regular expression as needed:

  ```
    // Parse log lines into timestamp, log level, component, and message using a regular expression
    // Example log format: "2023-10-01 12:00:00.000 INFO ComponentName This is a message"
    expression = `^(?P<timestamp>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}\.\d{3})\s+(?P<level>INFO|WARN|ERROR|FATAL)\s+(?P<component>\S+)\s+(?P<message>.*)`
  ```

Change Loki IP:

  ```
    // Specify the Loki server URL; ensure this endpoint is reachable from Alloy’s network
    url = "http://<IP>:3100/loki/api/v1/push"
  ```

#### Restart the Alloy service:
```powershell
Restart-Service -Name Alloy
```
or manually from:
```powershell
services.msc
```
#### Check the event log for errors:
```powershell
Get-WinEvent -FilterHashtable @{LogName="Application"; ProviderName="Alloy"; Level=@(2,3)} | Sort-Object TimeCreated
```
or manually from:
```powershell
eventvwr.msc
```
</details>

<details>
  <summary><strong>5. Deploy Grafana Monitoring Stack</strong></summary>

#### Clone this repository:
```sh
git clone https://github.com/SeshTiliRest/grafana-stack.git
cd grafana-stack
```

#### Start the monitoring stack:
```sh
docker compose up -d
```
</details>

<details>
  <summary><strong>6. Configure Grafana</strong></summary>

#### Access Grafana:
Open a web browser and navigate to:

```
http://<IP>:3000
```

#### Add Loki and Prometheus as data sources:
1. Log in to Grafana (default credentials: `admin` / `admin`).
2. Change password.
3. Go to **Connections → Data Sources**.
4. Click **Add data source**.
5. Select **Loki** and set the URL to:

   ```
   http://loki:3100
   ```

6. Click **Save & Test**.
7. Repeat the steps for **Prometheus**, setting the URL to:

   ```
   http://prometheus:9090
   ```

#### Explore data:
1. Navigate to **Explore**.
2. Select **loki**.
3. Switch A query from Builder to Code.
4. Paste: {host="windows_vm", job="app1_logs"} |= ``
5. Click **Run query**.

NOTE: You will see only new log lines because in config.alloy:

  ```
  // Start reading from the end of each file to capture only new log entries, ideal for real-time monitoring
  tail_from_end = true
  ```

#### Import the Grafana dashboard:
1. Navigate to **Dashboards → New → Import**.
2. Upload `dashboard.json` from the `grafana/` directory.
3. Load, select a Loki data source and click **Import**.

</details>