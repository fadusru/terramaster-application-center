# Homer

20261123
Приложение в логах пишет, что не найдена конфигурация, стартует с default настройками.
Приложение открывается на странице http://ip-nas:8154

Текст yaml
```yaml
services:
  homerdocker:
    healthcheck:
      test:
        - CMD-SHELL
        - 'wget -q -O /dev/null http://127.0.0.1:8080/ || exit 1'
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    image: 'moechz/homer:2026.11.14'
    container_name: homerdocker
    environment:
      - TZ=Europe/Moscow
    volumes:
      - '/Volume1/DockerAppData/homerdocker/assets:/www/assets'
    ports:
      - '8154:8080'
    restart: unless-stopped
    user: '1000:1000'
x-app-meta:
  web:
    port: 8154
    protocol: http
```