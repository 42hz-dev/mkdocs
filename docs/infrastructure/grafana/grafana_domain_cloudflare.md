# Grafana를 Cloudflare 도메인으로 연결하기 (Ubuntu + Nginx)

## 구성

    Internet
        │
    Cloudflare DNS
        │
    grafana.42hzs.com
        │
    Nginx (80/443)
        │
    127.0.0.1:3000
        │
    Grafana

## 1. Cloudflare DNS

  Type   Name      Content
  ------ --------- --------------
  A      grafana   서버 공인 IP

SSL/TLS 모드: **Full (strict)** 권장.

## 2. Grafana 설정

파일:

``` bash
sudo nano /etc/grafana/grafana.ini
```

권장 설정:

``` ini
[server]
protocol = http
http_port = 3000
domain = grafana.42hzs.com
root_url = https://grafana.42hzs.com
```

재시작:

``` bash
sudo systemctl restart grafana-server
sudo systemctl status grafana-server
```

## 3. Nginx 설정

생성:

``` bash
sudo nano /etc/nginx/sites-available/grafana
```

내용:

``` nginx
server {
    listen 80;
    server_name grafana.42hzs.com;

    location / {
        proxy_pass http://127.0.0.1:3000;

        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

심볼릭 링크 생성:

``` bash
sudo ln -s /etc/nginx/sites-available/grafana /etc/nginx/sites-enabled/grafana
```

설정 확인 및 적용:

``` bash
sudo nginx -t
sudo systemctl reload nginx
```

## 4. 포트 확인

Grafana(3000):

``` bash
ss -tlnp | grep 3000
```

Nginx(80/443):

``` bash
ss -tlnp | grep -E '80|443'
```

## 5. 동작 확인

``` bash
curl -I http://127.0.0.1:3000
curl -I http://localhost:3000
```

## 6. 로그 확인

Nginx

``` bash
sudo tail -100 /var/log/nginx/error.log
sudo tail -100 /var/log/nginx/access.log
```

Grafana

``` bash
sudo journalctl -u grafana-server -n 100 --no-pager
sudo tail -100 /var/log/grafana/grafana.log
```

## 7. 서비스 관리

``` bash
sudo systemctl start grafana-server
sudo systemctl stop grafana-server
sudo systemctl restart grafana-server
sudo systemctl status grafana-server

sudo systemctl restart nginx
sudo systemctl reload nginx
sudo systemctl status nginx
```

## 8. 자주 발생하는 문제

### Cloudflare 502

-   Grafana 미실행
-   `protocol=https` 설정(일반적인 Reverse Proxy 환경에서는 `http` 사용)
-   Nginx `proxy_pass` 오류
-   방화벽 또는 SELinux 문제

### Connection reset by peer

Grafana가 연결을 강제로 종료한 경우로, `protocol=https`와 프록시 설정
불일치를 먼저 확인.

## 9. 체크리스트

-   [ ] Cloudflare DNS 추가
-   [ ] grafana.ini 수정
-   [ ] Grafana 재시작
-   [ ] Nginx 설정 작성
-   [ ] sites-enabled 심볼릭 링크 생성
-   [ ] nginx -t 성공
-   [ ] Nginx Reload
-   [ ] `ss -tlnp | grep 3000` 확인
-   [ ] `curl -I http://127.0.0.1:3000` 확인
-   [ ] https://grafana.42hzs.com 접속 확인
