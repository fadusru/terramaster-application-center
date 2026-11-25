# Podfetch

20261123
Приложение открылось на странице http://ip-nas:8061

Текст yaml
```yaml
services:
  podfetchdocker:
    healthcheck:
      test:
        - CMD
        - /app/podfetch
        - users
        - list
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    image: 'samuel19982/podfetch:v5.2.3'
    container_name: podfetchdocker
    environment:
      - TZ=Europe/Moscow
      - 'DATABASE_URL=sqlite:///app/db/podcast.db'
      - POLLING_INTERVAL=300
      - BASIC_AUTH=false
      - GPODDER_INTEGRATION_ENABLED=false
    volumes:
      - '/Volume1/DockerAppData/podfetchdocker/db:/app/db'
      - '/Volume1/DockerAppData/podfetchdocker/podcasts:/app/podcasts'
    ports:
      - '8061:8000'
    restart: unless-stopped
    user: '1000:1000'
x-app-meta:
  web:
    port: 8061
    protocol: http
```