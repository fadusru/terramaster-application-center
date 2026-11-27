# MiloVault

20261127
Приложение доступно по адресу https://ip-nas:18443.

Текст yaml:
```yaml
services:
  milovault-init:
    read_only: true
    tmpfs:
      - /run
      - /tmp
    security_opt:
      - 'no-new-privileges:true'
    cap_drop:
      - ALL
    healthcheck:
      test:
        - CMD-SHELL
        - test -f /data/.milovault-ready
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 10s
    image: 'alpine:3.20'
    container_name: milovault-init
    volumes:
      - '/Volume1/DockerAppData/milovault/data:/data'
    restart: unless-stopped
    entrypoint:
      - /bin/sh
      - '-c'
      - |
        set -eu
        umask 077
        rm -f /data/.milovault-ready
        mkdir -p /data/certs
        chown -R 10001:10001 /data
        chmod 700 /data /data/certs
        for f in /data/certs/server.key /data/rsa_key.pem /data/db.sqlite3; do
          [ -f "$$f" ] && chmod 600 "$$f" || true
        done
        [ -f /data/certs/server.crt ] && chmod 644 /data/certs/server.crt || true
        printf '%s\n' ready > /data/.milovault-ready
        chown 10001:10001 /data/.milovault-ready
        chmod 600 /data/.milovault-ready
        echo "milovault-init: /data prepared for uid 10001"
        exec su nobody -s /bin/sh -c 'trap "exit 0" TERM INT; while :; do sleep 3600 & wait $$!; done'
    cap_add:
      - CHOWN
      - DAC_OVERRIDE
      - FOWNER
      - SETUID
      - SETGID
  milovault:
    stop_grace_period: 10s
    depends_on:
      milovault-init:
        condition: service_started
    security_opt:
      - 'no-new-privileges:true'
    cap_drop:
      - ALL
    healthcheck:
      test:
        - CMD-SHELL
        - 'curl -fsSk https://127.0.0.1:8443/alive || exit 1'
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 60s
    logging:
      driver: json-file
      options:
        max-size: 10m
        max-file: '3'
    image: 'vaultwarden/server:1.37.5'
    container_name: milovault
    environment:
      - 'DOMAIN=https://localhost:18443'
      - DATA_FOLDER=/data
      - WEB_VAULT_ENABLED=true
      - SIGNUPS_ALLOWED=true
      - ROCKET_ADDRESS=0.0.0.0
      - ROCKET_PORT=8443
      - 'ROCKET_TLS={certs="/data/certs/server.crt",key="/data/certs/server.key"}'
      - TZ=Europe/Moscow
      - LOG_LEVEL=warn
      - EXTENDED_LOGGING=true
      - LOG_FILE=/data/vaultwarden.log
      - SHOW_PASSWORD_HINT=false
      - WEBSOCKET_ENABLED=true
      - PUSH_ENABLED=false
      - DISABLE_ICON_DOWNLOAD=true
      - ICON_CACHE_TTL=0
      - UMASK=0077
    volumes:
      - '/Volume1/DockerAppData/milovault/data:/data'
    ports:
      - '18443:8443'
    restart: unless-stopped
    user: '10001:10001'
    entrypoint:
      - /bin/sh
      - '-c'
      - |
        set -eu
        umask 077
        waited=0
        while [ ! -w /data ]; do
          waited=$$((waited + 1))
          if [ "$$waited" -ge 60 ]; then
            echo "milovault: /data is not writable by uid $$(id -u); the milovault-init service did not prepare ownership" >&2
            exit 1
          fi
          sleep 1
        done
        mkdir -p /data/certs
        chmod 700 /data/certs
        cert_ok() {
          [ -s /data/certs/server.crt ] && [ -s /data/certs/server.key ] || return 1
          openssl x509 -checkend 86400 -noout -in /data/certs/server.crt >/dev/null 2>&1 || return 1
          cert_md5=$$(openssl x509 -noout -modulus -in /data/certs/server.crt 2>/dev/null | openssl md5) || return 1
          key_md5=$$(openssl rsa -noout -modulus -in /data/certs/server.key 2>/dev/null | openssl md5) || return 1
          [ "$$cert_md5" = "$$key_md5" ]
        }
        if ! cert_ok; then
          tmpdir=$$(mktemp -d)
          if ! openssl req -x509 -newkey rsa:2048 -sha256 -days 3650 -nodes \
            -keyout "$$tmpdir/server.key" \
            -out "$$tmpdir/server.crt" \
            -subj "/CN=milovault" \
            -addext "subjectAltName=DNS:localhost,DNS:milovault,DNS:tnas.local,IP:127.0.0.1" \
            >/dev/null 2>&1; then
            echo "milovault: failed to generate the self-signed certificate" >&2
            exit 1
          fi
          chmod 600 "$$tmpdir/server.key"
          chmod 644 "$$tmpdir/server.crt"
          mv "$$tmpdir/server.key" /data/certs/server.key
          mv "$$tmpdir/server.crt" /data/certs/server.crt
          rmdir "$$tmpdir"
        fi
        chmod 600 /data/certs/server.key
        chmod 644 /data/certs/server.crt
        exec /start.sh
version: '3.8'
x-app-meta:
  web:
    port: 18443
    protocol: https
```