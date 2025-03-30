<h1 align="center">Grafana стек для мониторинга логов</h1>

<p align="center">
  <img src="https://github.com/user-attachments/assets/6a035047-623f-4920-8118-3be9238c2b30" width="500">
</p>

<h3 align="center">Логи → Alloy → Loki → Grafana</h3>

## Обзор

TL;DR Есть виртуальная машина с Windows, на которой установлено два приложения:
- App1 генерирует ежедневные файлы логов (`2025-01-01.log`).
- App2 пишет логи в один файл (`message.log`).

Стек состоит из [Grafana](https://github.com/grafana/grafana/), [Loki](https://github.com/grafana/loki/) и [Grafana Alloy](https://github.com/grafana/alloy/) для мониторинга этих лог-файлов.

Дополнительно добавлен [Prometheus](https://github.com/prometheus/prometheus/) с конфигурацией на случай,  
если вы захотите использовать [windows_exporter](https://github.com/prometheus-community/windows_exporter/), [node exporter](https://github.com/prometheus/node_exporter/) или [другой экспортер](https://prometheus.io/docs/instrumenting/exporters/) для мониторинга в Grafana.

<details>
  <summary>App1 пример лога (2025-01-01.log)</summary>

```log
2025-01-01 12:00:59.109 FATAL  ComponentName Message
2025-01-01 13:00:59.109 ERROR  ComponentName Message
2025-01-01 14:00:59.109 WARN   ComponentName Message
2025-01-01 15:00:59.109 INFO   ComponentName Message
```

</details>

<details>
  <summary>App2 пример лога (message.log)</summary>

```log
01.01.25 12:00:59 - Message
01.01.25 13:00:59 - Message
01.01.25 14:00:59 - Message
01.01.25 15:00:59 - Message
```

</details>

<details>
  <summary>Пример дэшборда в Grafana</summary>

![grafana-dashboard](https://github.com/user-attachments/assets/595742f5-5a53-4fef-876e-4f8b389a8596)

</details>

## Инструкции по установке

<details>
  <summary><strong>1. (Опционально) Установка WSL для тестирования на Windows 11</strong></summary>

#### Если вы хотите протестировать этот стек на локальной Windows, установите WSL (Windows Subsystem for Linux), открыв PowerShell в Windows Terminal:

#### 1.1 Проверьте, установлен ли WSL:
```powershell
wsl -v
```

#### 1.2 Установите WSL, если он не установлен:
```powershell
wsl --install
```

#### 1.3 Список доступных дистрибутивов:
```powershell
wsl --list --online
```
или сокращенно:
```powershell
wsl -l -o
```

#### 1.4 Список установленных дистрибутивов:
```powershell
wsl -l
```

#### 1.5 Установите Ubuntu или другой дистрибутив:
```powershell
wsl --install -d Ubuntu-24.04
```

#### 1.6 После установки обновите систему:
```sh
sudo apt update && sudo apt upgrade
```

#### 1.7 Установите Git:
```sh
sudo apt install git
```

**Официальная документация:**  
[Руководство по установке WSL](https://learn.microsoft.com/ru-ru/windows/wsl/install/)

</details>

<details>
  <summary><strong>2. Установка Docker</strong></summary>

#### Вам понадобится Docker для развертывания этого стека. Если у вас его нет, установите его, следуя инструкциям ниже:

#### 2.1 Добавьте официальный GPG-ключ Docker:
```sh
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

#### 2.2 Добавьте репозиторий Docker в источники apt:
```sh
echo \  
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \  
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \  
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

#### 2.3 Установите Docker:
```sh
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

#### 2.4 Проверьте установку:
```sh
sudo docker run hello-world
```

#### 2.5 Проверьте запущенные контейнеры:
```sh
sudo docker ps
```

#### 2.6 Проверьте статус Docker:
```sh
sudo systemctl status docker
```

#### 2.7 Добавьте пользователя в группу Docker (чтобы запускать без sudo):
```sh
sudo usermod -aG docker $USER
```

#### 2.8 Примените изменения группы без выхода из системы:
```sh
newgrp docker
```

**Официальная документация:**  
[Руководство по установке Docker](https://docs.docker.com/engine/install/)

</details>

<details>
  <summary><strong>3. (Опционально) Установка Portainer</strong></summary>

#### Portainer — это веб-интерфейс для управления контейнерами Docker.

#### 3.1 Создайте том для Portainer:
```sh
docker volume create portainer_data
```

#### 3.2 Установите Portainer:
```sh
docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data portainer/portainer-ce:lts
```

#### 3.3 Проверьте запущенные контейнеры:
```sh
docker ps
```

#### 3.4 Проверьте IP-адрес:
```sh
ifconfig
```

Если `ifconfig` недоступен, установите `net-tools`:
```sh
sudo apt install net-tools
```

#### 3.5 Доступ к Portainer:
Откройте веб-браузер и перейдите по адресу:
```
https://<IP>:9443
```

**Официальная документация:**  
[Руководство по установке Portainer](https://docs.portainer.io/start/install-ce/server/docker/linux/)
</details>

<details>
  <summary><strong>4. Установка и настройка Grafana Alloy на системе с логами</strong></summary>

#### 4.1 [Скачайте и установите Grafana Alloy.](https://grafana.com/docs/alloy/latest/set-up/install/windows/)
#### 4.2 Отредактируйте `config.alloy` в каталоге с установленной Alloy:
```powershell
ii "C:\Program Files\GrafanaLabs\Alloy\"
```

#### 📝 ПРИМЕЧАНИЕ
Отредактируйте путь к папкам с логами приложений:

  ```
  // Define the path to App1 log files using glob patterns to match all .log files in the specified directory
  path_targets = [{"__path__" = "C:/App1/Folder/logs/Default/*.log"}]
  ```

  ```
  // Define the path to App2's single log file; no glob pattern needed as it's a specific file
  path_targets = [{"__path__" = "C:/App2/message.log"}]
  ```

Отредактируйте регулярное выражение:

  ```
    // Parse log lines into timestamp, log level, component, and message using a regular expression
    // Example log format: "2023-10-01 12:00:00.000 INFO ComponentName This is a message"
    expression = `^(?P<timestamp>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}\.\d{3})\s+(?P<level>INFO|WARN|ERROR|FATAL)\s+(?P<component>\S+)\s+(?P<message>.*)`
  ```

Отредактируйте IP-адрес Loki:

  ```
    // Specify the Loki server URL; ensure this endpoint is reachable from Alloy’s network
    url = "http://<IP>:3100/loki/api/v1/push"
  ```

#### 4.3 Перезапустите службу Alloy:
```powershell
Restart-Service -Name Alloy
```
или вручную через:
```powershell
services.msc
```
#### 4.5 Проверьте журнал событий на наличие ошибок:
```powershell
Get-WinEvent -FilterHashtable @{LogName="Application"; ProviderName="Alloy"; Level=@(2,3)} | Sort-Object TimeCreated
```
или вручную через:
```powershell
eventvwr.msc
```
</details>

<details>
  <summary><strong>5. Развертывание стека</strong></summary>

#### 5.1 Клонируйте репозиторий:
```sh
git clone https://github.com/SeshTiliRest/grafana-stack.git
cd grafana-stack
```
#### ⚠️ ВАЖНО
Внимательно ознакомьтесь с файлами `loki\config.yaml`, `prometheus\prometheus.yml` и `compose.yml` перед развертыванием!

#### 5.2 Запуск стека:
```sh
docker compose up -d
```
</details>

<details>
  <summary><strong>6. Настройка Grafana</strong></summary>

#### 6.1 Доступ к Grafana:
Откройте веб-браузер и перейдите по адресу:

```
http://<IP>:3000
```

#### 6.2 Добавление Loki и Prometheus в качестве источников данных:
- Войдите в Grafana (учетные данные по умолчанию: `admin` / `admin`).
- Измените пароль.
- Перейдите в **Connections → Data Sources**.
- Нажмите **Add data source**.
- Выберите **Loki** и установите URL:

   ```
   http://loki:3100
   ```

- Нажмите **Save & Test**.
- Повторите шаги для **Prometheus**, установив URL:

   ```
   http://prometheus:9090
   ```

#### 6.3 Исследование данных:
- Перейдите в **Explore**.
- Выберите **loki**.
- Переключите запрос A с Builder на Code.
- Вставьте: ` {host="windows_vm", job="app1_logs"} |= `` `
- Нажмите **Run query**.

#### 📝 ПРИМЕЧАНИЕ
Вы увидите только новые строки логов, так как в `config.alloy` указано:

  ```
  // Start reading from the end of each file to capture only new log entries, ideal for real-time monitoring
  tail_from_end = true
  ```

#### 6.4 Импорт дэшборда Grafana:
- Перейдите в **Dashboards → New → Import**.
- Загрузите `dashboard.json` из каталога `grafana/`.
- Нажмите **Load**, выберите источник данных **Loki** и нажмите **Import**.

</details>
