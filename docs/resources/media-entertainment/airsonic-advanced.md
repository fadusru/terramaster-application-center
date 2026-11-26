# Airsonic Advanced

20261126
Приложение открывается по адресу http://ip-nas:8172.
Please log in with username and password "admin".

Текст yaml:
```yaml
services:
  airsonicdocker:
    healthcheck:
      test:
        - CMD-SHELL
        - 'grep -q '':0FC8'' /proc/net/tcp6 || grep -q '':0FC8'' /proc/net/tcp'
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 120s
    image: 'linuxserver/airsonic-advanced:11.1.4'
    container_name: airsonicdocker
    environment:
      - TZ=Europe/Moscow
    volumes:
      - '/Volume1/DockerAppData/airsonicdocker/config:/config'
      - '/Volume1/DockerAppData/airsonicdocker/music:/music'
    ports:
      - '8172:4040'
    restart: unless-stopped
    user: '1000:1000'
    entrypoint:
      - /bin/sh
      - '-c'
      - |
        #!/bin/sh
        # ===========================================================================
        # Airsonic Advanced entrypoint for a non-root container (TOS 7 requirement).
        #
        # The LinuxServer image starts Airsonic through s6 as root; Airsonic also needs a
        # writable home passed as a JVM system property. This wrapper runs the war
        # directly as uid 1000 with -Dairsonic.home on the mounted volume.
        # ===========================================================================
        set -e
        set -u

        DATA=/config
        mkdir -p "$$DATA" /music 2>/dev/null || true

        if ! ( : > "$$DATA/.write-test" ) 2>/dev/null; then
          echo "airsonic: $$DATA is not writable by uid $$(id -u)" >&2
          exit 1
        fi
        rm -f "$$DATA/.write-test"

        exec java -Dairsonic.home="$$DATA" \
          -jar /app/airsonic/airsonic.war \
          --server.port=4040 \
          --server.address=0.0.0.0
x-app-meta:
  web:
    port: 8172
    protocol: http
```