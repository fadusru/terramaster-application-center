# File Browser

20262226
Приложение открывается по адресу http://ip-nas:8128.

Текст yaml:
```yaml
services:
  filebrowserdocker:
    healthcheck:
      test:
        - CMD-SHELL
        - 'wget -q -O /dev/null http://127.0.0.1:8080/ || exit 1'
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 20s
    image: 'filebrowser/filebrowser:v2.32.0'
    container_name: filebrowserdocker
    environment:
      - TZ=Europe/Moscow
    volumes:
      - '/Volume1/DockerAppData/filebrowserdocker/data:/data'
    ports:
      - '8128:8080'
    restart: unless-stopped
    user: '1000:1000'
    entrypoint:
      - /filebrowser
      - '-p'
      - '8080'
      - '-r'
      - /data
      - '-d'
      - /data/filebrowser.db
x-app-meta:
  web:
    port: 8128
    protocol: http
```