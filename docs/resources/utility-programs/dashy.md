# Dashy

20261127
Приложение доступно по адресу http://192.168.1.104:8127.

В интерфейсе пользователя окно с ошибкой:
Failed to fetch configuration: Server responded with status 404

https://github.com/lissy93/dashy/blob/master/user-data/conf.yml

Текст yaml:
```yaml
services:
  dashydocker:
    healthcheck:
      test:
        - CMD-SHELL
        - 'wget -q -O /dev/null http://127.0.0.1:8080/ || exit 1'
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    image: 'lissy93/dashy:4.9.17'
    container_name: dashydocker
    environment:
      - TZ=Europe/Moscow
    volumes:
      - '/Volume1/DockerAppData/dashydocker/user-data:/app/user-data'
    ports:
      - '8127:8080'
    restart: unless-stopped
    user: '1000:1000'
x-app-meta:
  web:
    port: 8127
    protocol: http
```