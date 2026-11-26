# ChangeDetection.io

20261126
Приложение открывается по адресу http://ip-nas:8037.

Текст yaml:
```yaml
services:
  changedetectiondocker:
    healthcheck:
      test:
        - CMD-SHELL
        - 'python3 -c "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen(''http://127.0.0.1:5000/'',timeout=5).status==200 else 1)"'
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    image: 'dgtlmoon/changedetection.io:0.70.2'
    container_name: changedetectiondocker
    environment:
      - TZ=Europe/Moscow
      - HOME=/tmp
    volumes:
      - '/Volume1/DockerAppData/changedetectiondocker/datastore:/datastore'
    ports:
      - '8037:5000'
    restart: unless-stopped
    user: '1000:1000'
x-app-meta:
  web:
    port: 8037
    protocol: http
```