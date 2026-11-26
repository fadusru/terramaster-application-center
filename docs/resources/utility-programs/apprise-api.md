# Apprise API

20261126
Приложение открывается по адресу http://ip-nas:8108.
Web login user 'admin'

Текст yaml:
```yaml
services:
  apprisedocker:
    healthcheck:
      test:
        - CMD-SHELL
        - 'curl -s -o /dev/null http://127.0.0.1:8000/ || exit 1'
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    image: 'linuxserver/apprise-api:2.1.0'
    container_name: apprisedocker
    environment:
      - TZ=Europe/Moscow
    volumes:
      - '/Volume1/DockerAppData/apprisedocker/config:/config'
    ports:
      - '8108:8000'
    restart: unless-stopped
    user: '1000:1000'
    entrypoint:
      - /bin/sh
      - '-c'
      - |
        #!/bin/sh
        # Apprise API stores notification credentials (bot tokens, SMTP passwords,
        # webhook URLs) in /config, so its web interface must not be open to every host
        # on the LAN. Upstream enables HTTP Basic Auth with APPRISE_AUTH_REQUIRED, but
        # it has no built-in account: an administrator password must be supplied.
        #
        # TOS users cannot pass environment variables to a package, so on the first
        # start this wrapper generates a random administrator password, keeps it in
        # /config (owner-only, uid 1000), reuses it on every later start and then hands
        # over to the image's own /init.
        set -u

        CRED=/config/admin-credentials.txt

        # The installer creates /config for uid 1000; make sure it exists anyway when
        # the container is started by hand.
        mkdir -p /config

        if [ ! -s "$$CRED" ]; then
            umask 077
            PASSWORD=$$(head -c 48 /dev/urandom | base64 | tr -dc 'A-Za-z0-9' | head -c 20)
            {
                printf 'Apprise API web interface - administrator login\n'
                printf 'username: %s\n' admin
                printf '%s: %s\n' password "$$PASSWORD"
                printf '\nDelete this file and restart the application to generate a new\n'
                printf 'password. Keep it: anyone with these credentials can read and\n'
                printf 'change every saved notification destination.\n'
            } > "$$CRED"
            chmod 600 "$$CRED"
        fi

        APPRISE_PASSWORD=$$(awk -F': ' '$$1 == "password" {print $$2; exit}' "$$CRED")
        if [ -z "$$APPRISE_PASSWORD" ]; then
            echo "apprisedocker: cannot read the administrator password from $$CRED" >&2
            exit 1
        fi

        export APPRISE_AUTH_REQUIRED=yes
        export APPRISE_USER=admin
        export APPRISE_PASSWORD

        echo "apprisedocker: HTTP Basic Auth is enabled (user 'admin')."
        echo "apprisedocker: the password is stored in $$CRED inside the container"
        echo "apprisedocker: (/Volume1/DockerAppData/apprisedocker/config/admin-credentials.txt on the NAS)."

        exec /init
x-app-meta:
  web:
    port: 8108
    protocol: http
```