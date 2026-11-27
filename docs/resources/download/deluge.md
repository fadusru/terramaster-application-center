# Deluge

20261127
Приложение открывается по адресу http://ip-nas:8020.
Default Web UI password is deluge

Текст yaml:
```yaml
services:
  delugedocker:
    cap_drop:
      - ALL
    security_opt:
      - 'no-new-privileges:true'
    tmpfs:
      - '/run:rw,exec,nosuid,nodev,mode=0755,uid=1000,gid=1000'
    healthcheck:
      test:
        - CMD-SHELL
        - 'curl -s -o /dev/null http://127.0.0.1:8112/ || exit 1'
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 45s
    image: 'linuxserver/deluge:2.2.0-ls388@sha256:52314116ba684ae5cdcbd380758be2a5f7cc866b4235f44ba7af8ff7c35c42b7'
    container_name: delugedocker
    environment:
      - TZ=Europe/Moscow
      - HOME=/config
      - DELUGE_LOGLEVEL=error
    volumes:
      - '/Volume1/DockerAppData/delugedocker/config:/config'
      - '/Volume1/DockerAppData/delugedocker/downloads:/downloads'
    ports:
      - '8020:8112'
    restart: unless-stopped
    user: '1000:1000'
    entrypoint:
      - /bin/sh
      - '-c'
      - |
        #!/bin/sh
        # ===========================================================================
        # Entrypoint wrapper for the LinuxServer.io Deluge image.
        #
        # The image is used unmodified, this wrapper only runs before /init:
        #
        #   1. umask 027 - every file Deluge writes (config, resume data, log, torrent
        #      payload) is created without group/world write access instead of the
        #      image's default 022 (finding S11).
        #   2. /run is a tmpfs owned by uid 1000 (see docker-compose.yml). Recreate the
        #      socket temp dir the image expects there, with owner-only access.
        #   3. Tighten /config: it holds the Web UI credential hash (config/auth) and
        #      the daemon settings, so the tree is set to owner-only (0700 dirs, 0600
        #      files). Only files this uid owns are changed; a root-owned entry left by
        #      the platform is skipped instead of aborting the start-up.
        #   4. /downloads is the user's payload and stays readable by the group, so a
        #      shared download folder keeps working while nothing is world writable.
        #
        # The script must not contain a literal double dollar sign (the packaging step
        # uses it as the escape sequence) and is executed as uid 1000, never as root.
        # ===========================================================================
        set -u

        umask 027

        if [ -d /run ]; then
            mkdir -p /run/deluged-temp 2>/dev/null || true
            chmod 0700 /run/deluged-temp 2>/dev/null || true
        fi

        if [ -d /config ]; then
            find /config -xdev -exec chmod go-rwx {} + 2>/dev/null || true
            chmod 0700 /config 2>/dev/null || true
        fi

        if [ -d /downloads ]; then
            chmod 0750 /downloads 2>/dev/null || true
        fi

        exec /init
x-app-meta:
  web:
    port: 8020
    protocol: http
```