# Piwigo

## Официальный сайт [piwigo.org](https://ru.piwigo.org)

## Тестирование версий

20261127
Устанавливается 2 контейнера. Из Центра Приложений не запускается, "руками" отправить ссылку из контейнера на приложение
на рабочий стол. Приложение доступно по адресу http://ip-nas:8023.

Текст yaml:
```yaml
services:
  piwigodocker:
    depends_on:
      - piwigodocker-db
    healthcheck:
      test:
        - CMD-SHELL
        - 'curl -s -o /dev/null http://127.0.0.1:8080/ || exit 1'
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    image: 'linuxserver/piwigo:version-16.4.0'
    container_name: piwigodocker
    environment:
      - TZ=Europe/Moscow
    volumes:
      - '/Volume1/DockerAppData/piwigodocker/local:/app/www/public/local'
      - '/Volume1/DockerAppData/piwigodocker/_data:/app/www/public/_data'
      - '/Volume1/DockerAppData/piwigodocker/upload:/app/www/public/upload'
      - '/Volume1/DockerAppData/piwigodocker/galleries:/app/www/public/galleries'
    ports:
      - '8023:8080'
    restart: unless-stopped
    user: '1000:1000'
    entrypoint:
      - /bin/sh
      - '-c'
      - |
        #!/bin/sh
        # Piwigo's LinuxServer image starts s6 as root and symlinks its writable
        # directories into /app/www/public. TOS forbids root containers, so run the
        # unmodified Piwigo code with PHP's built-in server as uid 1000 instead.
        # local/, _data/, upload/ and galleries/ are mounted writable and persist the
        # installer config, cache, uploads and the photo library.
        set -u

        for d in local _data upload galleries; do
            mkdir -p "/app/www/public/$$d" 2>/dev/null || true
        done

        cd /app/www/public || exit 1
        exec php -S 0.0.0.0:8080 -t /app/www/public
  piwigodocker-db:
    healthcheck:
      test:
        - CMD-SHELL
        - mariadb-admin -u root ping || exit 1
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    image: 'mariadb:11.4'
    container_name: piwigodocker-db
    environment:
      - TZ=Asia/Shanghai
      - MARIADB_DATABASE=piwigo
      - MARIADB_ALLOW_EMPTY_ROOT_PASSWORD=yes
    volumes:
      - '/Volume1/DockerAppData/piwigodocker/db:/var/lib/mysql'
      - '/Volume1/DockerAppData/piwigodocker/run:/run/mysqld'
    restart: unless-stopped
    user: '1000:1000'
x-app-meta:
  web:
    port: 8023
    protocol: http
```

20261123
Открывается интерфейс для установки версии 16.4.0.

Текст yaml
```yaml
services:
  piwigodocker:
    depends_on:
      - piwigodocker-db
    healthcheck:
      test:
        - CMD-SHELL
        - 'curl -s -o /dev/null http://127.0.0.1:8080/ || exit 1'
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    image: 'linuxserver/piwigo:version-16.4.0'
    container_name: piwigodocker
    environment:
      - TZ=Europe/Moscow
    volumes:
      - '/Volume1/DockerAppData/piwigodocker/local:/app/www/public/local'
      - '/Volume1/DockerAppData/piwigodocker/_data:/app/www/public/_data'
      - '/Volume1/DockerAppData/piwigodocker/upload:/app/www/public/upload'
      - '/Volume1/DockerAppData/piwigodocker/galleries:/app/www/public/galleries'
    ports:
      - '8023:8080'
    restart: unless-stopped
    user: '1000:1000'
    entrypoint:
      - /bin/sh
      - '-c'
      - |
        #!/bin/sh
        # Piwigo's LinuxServer image starts s6 as root and symlinks its writable
        # directories into /app/www/public. TOS forbids root containers, so run the
        # unmodified Piwigo code with PHP's built-in server as uid 1000 instead.
        # local/, _data/, upload/ and galleries/ are mounted writable and persist the
        # installer config, cache, uploads and the photo library.
        set -u

        for d in local _data upload galleries; do
            mkdir -p "/app/www/public/$$d" 2>/dev/null || true
        done

        cd /app/www/public || exit 1
        exec php -S 0.0.0.0:8080 -t /app/www/public
  piwigodocker-db:
    healthcheck:
      test:
        - CMD-SHELL
        - mariadb-admin -u root ping || exit 1
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    image: 'mariadb:11.4'
    container_name: piwigodocker-db
    environment:
      - TZ=Asia/Shanghai
      - MARIADB_DATABASE=piwigo
      - MARIADB_ALLOW_EMPTY_ROOT_PASSWORD=yes
    volumes:
      - '/Volume1/DockerAppData/piwigodocker/db:/var/lib/mysql'
      - '/Volume1/DockerAppData/piwigodocker/run:/run/mysqld'
    restart: unless-stopped
    user: '1000:1000'
x-app-meta:
  web:
    port: 8023
    protocol: http
```