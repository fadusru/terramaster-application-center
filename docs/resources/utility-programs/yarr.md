# Yarr

20261123
Пользовательский интерфейс открылся по адресу http://ip-nas:8161

Текст yaml
```yaml
services:
  yarrdocker:
    healthcheck:
      test:
        - CMD-SHELL
        - 'grep -q '':1B9E'' /proc/net/tcp6'
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    image: 'nkanaev/yarr:v2.9'
    container_name: yarrdocker
    environment:
      - TZ=Europe/Moscow
    volumes:
      - '/Volume1/DockerAppData/yarrdocker/data:/data'
    ports:
      - '8161:7070'
    restart: unless-stopped
    user: '1000:1000'
x-app-meta:
  web:
    port: 8161
    protocol: http
```