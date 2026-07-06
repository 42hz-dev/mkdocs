# Grafana + Prometheus + Node Exporter 설치 (Ubuntu 24.04)

> Ubuntu 24.04 LTS 환경에서 Docker 없이 직접 설치하는 방법입니다.

---

# 모니터링 확인

현재 구축된 Grafana 대시보드는 아래 URL에서 확인할 수 있습니다.

| 항목 | 내용 |
|------|------|
| URL | https://grafana.42hzs.com/d/rYdddlPWk/node-exporter-full?orgId=1&from=now-24h&to=now&timezone=browser&var-ds_prometheus=ffqdamu3n9zpce&var-job=node&var-nodename=fourtwo-290888&var-node=localhost:9100&refresh=1m |
| ID | admin |
| Password | admin |

> **현재는 단일 서버(Node Exporter 1대)만 연결되어 있습니다.**
>
> 추후 여러 서버를 추가할 경우 Prometheus의 `scrape_configs`에 Node Exporter를 등록하면 Grafana에서 동일한 Dashboard로 여러 서버를 선택하여 모니터링할 수 있습니다.

---

# 시스템 구성

```text
Monitoring Server

├── Grafana        (3000)
├── Prometheus     (9090)
└── Node Exporter  (9100)
```

---

# 1. 사용자 생성

보안을 위해 Prometheus와 Node Exporter를 별도의 계정으로 실행합니다.

```bash
sudo useradd --no-create-home --shell /bin/false prometheus
sudo useradd --no-create-home --shell /bin/false node_exporter
```

---

# 2. Prometheus 설치

## 2-1. 최신 버전 확인

```bash
cd /tmp

curl -s https://api.github.com/repos/prometheus/prometheus/releases/latest | grep tag_name
```

예시

```text
"tag_name": "v3.6.0"
```

---

## 2-2. 다운로드

> 확인한 버전에 맞게 다운로드합니다.

```bash
wget https://github.com/prometheus/prometheus/releases/download/v3.6.0/prometheus-3.6.0.linux-amd64.tar.gz
```

---

## 2-3. 압축 해제

```bash
tar -xzf prometheus-3.6.0.linux-amd64.tar.gz

cd prometheus-3.6.0.linux-amd64
```

---

## 2-4. 설치

```bash
sudo mkdir -p /etc/prometheus
sudo mkdir -p /var/lib/prometheus

sudo cp prometheus /usr/local/bin/
sudo cp promtool /usr/local/bin/

#sudo cp -r consoles /etc/prometheus/
#sudo cp -r console_libraries /etc/prometheus/
sudo cp prometheus.yml /etc/prometheus/
```

---

## 2-5. 권한 설정

```bash
sudo chown -R prometheus:prometheus /etc/prometheus
sudo chown -R prometheus:prometheus /var/lib/prometheus
```

---

# 3. Prometheus 서비스 등록

```bash
sudo vi /etc/systemd/system/prometheus.service
```

```ini
[Unit]
Description=Prometheus
After=network.target

[Service]
User=prometheus
Group=prometheus

ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus

Restart=always

[Install]
WantedBy=multi-user.target
```

---

## 서비스 실행

```bash
sudo systemctl daemon-reload

sudo systemctl enable prometheus

sudo systemctl start prometheus
```

---

## 상태 확인

```bash
systemctl status prometheus
```

---

## 접속

```
http://SERVER_IP:9090
```

---

# 4. Node Exporter 설치

## 4-1. 최신 버전 확인

```bash
curl -s https://api.github.com/repos/prometheus/node_exporter/releases/latest | grep tag_name
```

예시

```text
"tag_name": "v1.10.2"
```

---

## 4-2. 다운로드

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz
```

---

## 4-3. 압축 해제

```bash
tar -xzf node_exporter-1.10.2.linux-amd64.tar.gz

cd node_exporter-1.10.2.linux-amd64
```

---

## 4-4. 설치

```bash
sudo cp node_exporter /usr/local/bin/
```

---

## 4-5. 서비스 등록

```bash
sudo vi /etc/systemd/system/node_exporter.service
```

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter

ExecStart=/usr/local/bin/node_exporter

Restart=always

[Install]
WantedBy=multi-user.target
```

---

## 서비스 실행

```bash
sudo systemctl daemon-reload

sudo systemctl enable node_exporter

sudo systemctl start node_exporter
```

---

## 상태 확인

```bash
systemctl status node_exporter
```

---

## 접속

```
http://SERVER_IP:9100/metrics
```

---

# 5. Prometheus 설정

```bash
sudo vi /etc/prometheus/prometheus.yml
```

```yaml
# Global Config
global:
  scrape_interval: 15s
  evaluation_interval: 15s

# AlertManager
alerting:
  alertmanagers:
    - static_configs:
        - targets:

# Rule Files
rule_files:

# Scrape Targets
scrape_configs:

  # Prometheus
  - job_name: "prometheus"

    static_configs:
      - targets:
          - localhost:9090
        labels:
          app: "prometheus"

  # Node Exporter
  - job_name: "node"

    static_configs:
      - targets:
          - localhost:9100
          # - 192.168.0.10:9100
          # - 192.168.0.11:9100
        labels:
          app: "node-exporter"
```

---

## 설정 적용

```bash
sudo systemctl restart prometheus
```

---

## Target 확인

```
http://SERVER_IP:9090/targets
```

정상이라면 아래 두 항목이 모두 **UP** 상태로 표시됩니다.

* Prometheus
* Node Exporter

---

# 6. Grafana 설치

## 저장소 추가

```bash
sudo apt update

sudo apt install -y software-properties-common

wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -

echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt update
```

---

## 설치

```bash
sudo apt install grafana
```

---

## 서비스 실행

```bash
sudo systemctl enable grafana-server

sudo systemctl start grafana-server
```

---

## 상태 확인

```bash
systemctl status grafana-server
```

---

## 접속

```
http://SERVER_IP:3000
```

---

## 기본 계정

```
ID       : admin
Password : admin
```

최초 로그인 시 비밀번호 변경을 요구합니다.

---

# 7. Prometheus Data Source 등록

Grafana 로그인 후

```
Connections
    ↓
Data Sources
    ↓
Add data source
```

### 설정

| 항목   | 값                     |
| ---- | --------------------- |
| Type | Prometheus            |
| URL  | http://localhost:9090 |

설정 후 **Save & Test**를 클릭하여 연결이 성공하는지 확인합니다.

---

# 설치 완료

현재 구성은 다음과 같습니다.

```text
Ubuntu Server
│
├── Grafana
│     └── http://SERVER_IP:3000
│
├── Prometheus
│     └── http://SERVER_IP:9090
│
└── Node Exporter
      └── http://SERVER_IP:9100/metrics
```

다음 단계에서는 Grafana Dashboard를 추가하여 CPU, Memory, Disk, Network 등의 시스템 리소스를 시각화합니다.
