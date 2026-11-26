# PlantUML Server

20261126
Приложение доступно по адресу http://ip-nas:8152/

Текст yaml
```yaml
services:
  plantumldocker:
    healthcheck:
      test:
        - CMD-SHELL
        - 'curl -s -o /dev/null http://127.0.0.1:8080/ || exit 1'
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    image: 'plantuml/plantuml-server:v1.2026.10'
    container_name: plantumldocker
    environment:
      - TZ=Europe/Moscow
    volumes:
      - '/Volume1/DockerAppData/plantumldocker/data:/var/lib/plantuml'
    ports:
      - '8152:8080'
    restart: unless-stopped
    user: '1000:1000'
x-app-meta:
  web:
    port: 8152
    protocol: http
```