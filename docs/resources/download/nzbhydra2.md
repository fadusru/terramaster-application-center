# Nzbhydra2


20261126
Приложение открывается по адресу http://ip-nas:8142.
Wiki: https://github.com/theotherp/nzbhydra2/wiki

Текст yaml:
```yaml
services:
  nzbhydradocker:
    healthcheck:
      test:
        - CMD-SHELL
        - 'curl -s -o /dev/null http://127.0.0.1:5076/ || exit 1'
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 90s
    image: 'linuxserver/hydra2:version-v3.0.0'
    container_name: nzbhydradocker
    environment:
      - TZ=Europe/Moscow
    volumes:
      - '/Volume1/DockerAppData/nzbhydradocker/config:/config'
    ports:
      - '8142:5076'
    restart: unless-stopped
    user: '1000:1000'
    entrypoint:
      - /bin/sh
      - '-c'
      - |
        #!/bin/sh
        # ===========================================================================
        # NZBHydra 2 entrypoint for a non-root container (TOS 7 requirement).
        #
        # The LinuxServer.io image starts the Java application through s6 as root; s6
        # cannot run inside a container that already runs as uid 1000. The upstream
        # python launcher can prepare and start the jar by itself, so this wrapper calls
        # it directly with the data folder on the mounted volume.
        # ===========================================================================
        set -e
        set -u

        DATA=/config
        mkdir -p "$$DATA" 2>/dev/null || true

        if ! ( : > "$$DATA/.write-test" ) 2>/dev/null; then
          echo "nzbhydra: $$DATA is not writable by uid $$(id -u)" >&2
          exit 1
        fi
        rm -f "$$DATA/.write-test"

        exec python3 /app/hydra2/nzbhydra2wrapperPy3.py \
          --datafolder "$$DATA" \
          --port 5076 \
          --nobrowser
x-app-meta:
  web:
    port: 8142
    protocol: http
```