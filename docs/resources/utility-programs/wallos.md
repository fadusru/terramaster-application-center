# Wallos

20261126
Приложение открывается по адресу http://ip-nas:8174.
Wallos — Subscription Tracker

Текст yaml:
```yaml
services:
  wallosdocker:
    stop_signal: SIGTERM
    cap_drop:
      - ALL
    security_opt:
      - 'no-new-privileges:true'
    healthcheck:
      test:
        - CMD
        - curl
        - '-fsS'
        - 'http://127.0.0.1:8080/health.php'
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 60s
    image: 'bellamy/wallos:5.11.3'
    container_name: wallosdocker
    environment:
      - TZ=Europe/Moscow
    volumes:
      - '/Volume1/DockerAppData/wallosdocker/db:/var/www/html/db'
      - '/Volume1/DockerAppData/wallosdocker/logos:/var/www/html/images/uploads/logos'
      - '/Volume1/DockerAppData/wallosdocker/tmp:/var/www/html/.tmp'
    ports:
      - '8174:8080'
    restart: unless-stopped
    user: '1000:1000'
    entrypoint:
      - /bin/sh
      - '-c'
      - |
        #!/bin/sh
        # ===========================================================================
        # Wallos entrypoint for a non-root container (TOS 7 requirement).
        #
        # The upstream image starts php-fpm, nginx (port 80) and crond from
        # /var/www/html/startup.sh, which needs root: it rewrites the uid/gid of
        # www-data, chowns the web root, writes /var/log/startup.log and binds port 80.
        # uid 1000 can do none of that, so this wrapper runs the same three processes
        # as uid 1000, publishes nginx on 8080 and keeps the persistent folders on the
        # mounted volumes.
        #
        # It also owns the database bootstrap that startup.sh performs:
        # createdatabase.php (schema, and repair of an empty wallos.db) followed by
        # migrate.php. The 5.11.1-2 package ran only `php -S` here, so db/wallos.db was
        # never created and every page answered HTTP 200 with
        # "SQLite3::query(): Unable to prepare statement: no such table: user" - the
        # blocking App Center finding this version fixes.
        #
        # Stop behaviour: this shell is PID 1, forwards QUIT (the image's stopsignal),
        # TERM and INT to the scheduler, nginx and php-fpm, gives them a moment to shut
        # down and exits 0, so `docker stop` returns in about two seconds instead of
        # waiting for the runtime to SIGKILL the container.
        # ===========================================================================
        set -e
        set -u

        APP=/var/www/html
        DB="$$APP/db"
        LOGOS="$$APP/images/uploads/logos"
        TMP="$$APP/.tmp"
        PHP=/usr/local/bin/php
        [ -x "$$PHP" ] || PHP=php

        RUNDIR=/tmp/wallos
        CRONLOG="$$RUNDIR/log"
        CRONSTATE="$$RUNDIR/state"

        # --------------------------------------------------------------- data folders
        mkdir -p "$$DB" "$$LOGOS/avatars" "$$TMP" 2>/dev/null || true
        mkdir -p "$$RUNDIR/nginx" "$$CRONLOG" "$$CRONSTATE"

        for dir in "$$DB" "$$LOGOS" "$$TMP"; do
            if ! ( : > "$$dir/.write-test" ) 2>/dev/null; then
                echo "wallos: $$dir is not writable by uid $$(id -u); check the volume permissions" >&2
                exit 1
            fi
            rm -f "$$dir/.write-test"
        done

        # ------------------------------------------------------- database bootstrap --
        # Same two steps the upstream startup.sh runs on every start. Both are
        # idempotent: createdatabase.php only creates the schema when the user table is
        # missing (a 0-byte wallos.db counts as missing), migrate.php applies pending
        # migrations and is a no-op when the schema is current.
        echo "wallos: preparing the database in $$DB" >&2
        if ! "$$PHP" "$$APP/endpoints/cronjobs/createdatabase.php"; then
            echo "wallos: createdatabase.php failed - refusing to serve a broken database" >&2
            exit 1
        fi
        if ! "$$PHP" "$$APP/endpoints/db/migrate.php"; then
            echo "wallos: migrate.php failed - refusing to serve a half-migrated database" >&2
            exit 1
        fi

        # -------------------------------------------------------- php-fpm / nginx ----
        # nginx and php-fpm both run as uid 1000, so neither may need a root-only path:
        # pid files, logs and temp dirs all live below /tmp/wallos.
        cat > "$$RUNDIR/php-fpm.conf" <<'FPM'
        [global]
        error_log = /dev/stderr
        daemonize = no
        pid = /tmp/wallos/php-fpm.pid
        process_control_timeout = 10s

        [www]
        listen = 127.0.0.1:9000
        pm = dynamic
        pm.max_children = 15
        pm.start_servers = 2
        pm.min_spare_servers = 1
        pm.max_spare_servers = 4
        pm.max_requests = 500
        catch_workers_output = yes
        php_admin_value[upload_max_filesize] = 256M
        php_admin_value[post_max_size] = 256M
        php_admin_value[session.save_path] = /tmp
        FPM

        # Same routing and the same deny rules as the upstream nginx.default.conf, with
        # the listener moved to 8080 (the unprivileged port this package publishes) and
        # the fastcgi upstream on the TCP socket php-fpm listens on above.
        cat > "$$RUNDIR/nginx.conf" <<'NGINX'
        worker_processes 1;
        error_log /dev/stderr warn;
        pid /tmp/wallos/nginx.pid;

        events {
            worker_connections 1024;
        }

        http {
            include /etc/nginx/mime.types;
            default_type application/octet-stream;
            access_log /dev/stdout;

            client_body_temp_path /tmp/wallos/nginx/client_body;
            proxy_temp_path /tmp/wallos/nginx/proxy;
            fastcgi_temp_path /tmp/wallos/nginx/fastcgi;
            uwsgi_temp_path /tmp/wallos/nginx/uwsgi;
            scgi_temp_path /tmp/wallos/nginx/scgi;

            sendfile on;
            keepalive_timeout 65;
            client_max_body_size 256M;

            server {
                listen 8080;
                server_name localhost;
                root /var/www/html;
                index index.php;

                location / {
                    try_files $$uri $$uri/ /index.php?$$args;
                }

                # Deny rules precede the generic \.php$$ handler: nginx uses the first
                # matching regex, so otherwise \.php$$ would execute PHP under .tmp/.
                location ~ \.db$$ {
                    deny all;
                    return 403;
                }

                location ~* images/uploads/logos/.*\.php$$ {
                    deny all;
                    return 403;
                }

                location ^~ /.tmp/ {
                    deny all;
                    return 403;
                }

                # The upstream image also ships the migration scripts, the developer
                # folder and the test suite inside the web root. None of them is a web
                # entry point: migrations are run from run_migrations.php, the other two
                # are development material, and executing them over HTTP would mutate
                # the schema or run test fixtures without the application's checks.
                location ^~ /migrations/ {
                    deny all;
                    return 403;
                }

                location ^~ /dev/ {
                    deny all;
                    return 403;
                }

                location ^~ /tests/ {
                    deny all;
                    return 403;
                }

                location ~* ^/includes/.*\.php$$ {
                    deny all;
                    return 403;
                }

                location ~ \.php$$ {
                    include /etc/nginx/fastcgi_params;
                    fastcgi_pass 127.0.0.1:9000;
                    fastcgi_param SCRIPT_FILENAME $$document_root$$fastcgi_script_name;
                }
            }
        }
        NGINX

        # ------------------------------------------------------------- scheduler ------
        # The upstream image runs these scripts from /etc/cron.d/cronjobs through
        # crond, but crond in the image is /usr/sbin/crond (mode 0700, root) and cannot
        # be used as uid 1000. This loop replaces it: every job is spawned from the PHP
        # CLI, the 300 s cap matches the upstream crontab, and a per-day (or per-6h
        # bucket) state file makes a delayed start catch up instead of skipping a day.
        # Every test is an `if`, never a bare `[ ... ] && cmd`: with `set -e` a
        # false test at the top level of the loop body would end the function.
        run_job() { # <script> <key> [argument]
            _marker="$$CRONSTATE/$$1.$$2"
            if [ -e "$$_marker" ]; then
                return 0
            fi
            : > "$$_marker"
            # Sequential on purpose: the upstream crontab caps each job at 300 s because
            # crond may overlap runs. This loop never overlaps, and every job's network
            # call already has its own PHP/curl timeout, so a hung request only delays
            # the next job instead of blocking a second copy of itself.
            "$$PHP" "$$APP/endpoints/cronjobs/$$1.php" $${3:-} \
                >> "$$CRONLOG/$$1.log" 2>&1 || true
            return 0
        }

        scheduler() {
            while :; do
                now=$$(date +%H%M)
                n=$$(printf '%d' "$$now")
                today=$$(date +%F)
                dow=$$(date +%u)
                dom=$$(date +%d)
                hour=$$(printf '%d' "$$(date +%H)")
                bucket=$$((hour / 6))

                # every-two-minute jobs from the upstream crontab
                run_job sendverificationemails "$$today$$now"
                run_job sendresetpasswordemails "$$today$$now"

                if [ "$$n" -ge 100 ]; then run_job updatenextpayment "$$today"; fi
                if [ "$$n" -ge 200 ]; then run_job updateexchange "$$today"; fi
                if [ "$$n" -ge 300 ]; then run_job cleanupresettokens "$$today"; fi
                if [ "$$n" -ge 800 ]; then run_job sendcancellationnotifications "$$today"; fi
                if [ "$$n" -ge 900 ]; then run_job sendnotifications "$$today"; fi
                if [ "$$dow" = 1 ] && [ "$$n" -ge 130 ]; then
                    run_job storetotalyearlycost "$$today"
                fi
                if [ "$$dow" = 1 ] && [ "$$n" -ge 330 ]; then
                    run_job generaterecommendations "$$today-weekly" weekly
                fi
                if [ "$$dom" = 01 ] && [ "$$n" -ge 400 ]; then
                    run_job generaterecommendations "$$today-monthly" monthly
                fi

                run_job checkforupdates "$$today.$$bucket"
                sleep 60
            done
        }

        # ---------------------------------------------------------------- processes --
        FPM_PID=
        WEB_PID=
        CRON_PID=

        shutdown() { # <exit code>
            trap - TERM INT
            echo "wallos: stopping (scheduler, nginx, php-fpm)" >&2
            kill -TERM "$$CRON_PID" 2>/dev/null || true
            kill -QUIT "$$WEB_PID" 2>/dev/null || true
            kill -QUIT "$$FPM_PID" 2>/dev/null || true
            # Deliberately no `wait`: a running cron job would make `docker stop` hang
            # until it finishes. php-fpm and nginx get their QUIT above, and the runtime
            # kills whatever is left when PID 1 exits.
            sleep 2
            exit "$$1"
        }

        # The upstream image declares STOPSIGNAL SIGQUIT, so `docker stop` (and the TOS
        # stop button) sends QUIT, not TERM: PID 1 ignores an unhandled QUIT and the
        # platform then has to SIGKILL the container after its grace period - the
        # "service fails to stop cleanly" finding on 5.11.1-2. All three stop signals
        # are handled, and docker-compose.yml pins stop_signal: SIGTERM as well.
        trap 'shutdown 0' TERM INT QUIT

        # A process that has exited but has not been reaped yet shows state Z, and a pid
        # that no longer exists has no /proc entry: both mean "gone". busybox ps in this
        # image has no -p option, so the state is read from /proc directly.
        service_alive() { # <pid>
            if [ ! -d "/proc/$$1" ]; then
                return 1
            fi
            ! grep -q '^State:[[:space:]]*Z' "/proc/$$1/status" 2>/dev/null
        }

        php-fpm --nodaemonize --fpm-config "$$RUNDIR/php-fpm.conf" &
        FPM_PID=$$!

        nginx -e /dev/stderr -c "$$RUNDIR/nginx.conf" -g 'daemon off;' &
        WEB_PID=$$!

        scheduler &
        CRON_PID=$$!

        # woken up as soon as one of the three exits (or by the TERM trap above); the
        # loop then checks all three, so a service that dies stops the container
        # instead of leaving a half-working installation behind.
        while service_alive "$$FPM_PID" && service_alive "$$WEB_PID" && \
              service_alive "$$CRON_PID"; do
            wait -n 2>/dev/null || true
        done

        echo "wallos: a service process exited - stopping the container" >&2
        shutdown 1
x-app-meta:
  web:
    port: 8174
    protocol: http
```