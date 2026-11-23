# 2fauth

20261123
При переходе по ссылке (http://ip-nas:8040) открывается белый экран.

Текст yaml
```yaml
services:
  twofauthdocker:
    healthcheck:
      test:
        - CMD-SHELL
        - 'wget -q -O /dev/null http://127.0.0.1:8000/ || exit 1'
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    image: '2fauth/2fauth:6.1.3'
    container_name: twofauthdocker
    environment:
      - TZ=Europe/Moscow
    volumes:
      - '/Volume1/DockerAppData/twofauthdocker/storage:/2fauth/storage'
    ports:
      - '8040:8000'
    restart: unless-stopped
    user: '1000:1000'
    entrypoint:
      - /bin/sh
      - '-c'
      - |
        #!/bin/sh
        # TOS wrapper: 2FAuth needs a persistent APP_KEY and a persistent SQLite DB.
        # Both live on the mounted storage volume; the framework dirs must exist before
        # `php artisan optimize` runs (empty mounted volumes would break view:cache).
        set -u
        for d in framework/views framework/cache framework/sessions logs app; do
          mkdir -p "/2fauth/storage/$$d"
        done
        KEY_FILE=/2fauth/storage/app_key
        if [ ! -s "$$KEY_FILE" ]; then
          head -c 32 /dev/urandom | base64 | tr -d '\n' > "$$KEY_FILE"
          chmod 600 "$$KEY_FILE"
        fi
        export APP_KEY="$$(cat "$$KEY_FILE")"
        export DB_DATABASE=/2fauth/storage/database.sqlite
        exec /usr/local/bin/entrypoint.sh
x-app-meta:
  web:
    port: 8040
    protocol: http
```