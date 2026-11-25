# Mainsail

20261125
Приложение открывается по адресу http://ip-nas:8094. Для работы необходимо установить Moonraker и Klipper.

Текст yaml
```yaml
services:
  mainsail:
    image: 'ghcr.io/mainsail-crew/mainsail:latest'
    container_name: mainsail
    ports:
      - '8094:8080'
    restart: always
name: mainsail
x-app-meta:
  web:
    port: 8094
    protocol: http
```