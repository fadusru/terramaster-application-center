# RabbitMQ

20261125
Приложение открывается по адресу http://ip-nas:15672

Текст yaml:
```yaml
services:
  rabbitmq:
    healthcheck:
      test:
        - CMD
        - rabbitmq-diagnostics
        - '-q'
        - check_port_connectivity
      interval: 10s
      timeout: 8s
      start_period: 30s
      retries: 8
    image: 'rabbitmq:4.4.1-management@sha256:61f90c401a5b5ed2486771973ea92ad66046471eacab9c9149f1a6e091c91677'
    container_name: rabbitmq
    environment:
      - RABBITMQ_NODENAME=rabbit@rabbitmq
    volumes:
      - '/Volume1/DockerAppData/rabbitmq/data:/var/lib/rabbitmq'
    ports:
      - '15673:5672'
      - '15672:15672'
    restart: unless-stopped
    hostname: rabbitmq
    entrypoint:
      - /bin/sh
      - '-ec'
      - |
        (
          umask 077
          if [ ! -s /var/lib/rabbitmq/.tos-admin-password ]; then
            tr -d '\n-' < /proc/sys/kernel/random/uuid > /var/lib/rabbitmq/.tos-admin-password.new
            tr -d '\n-' < /proc/sys/kernel/random/uuid >> /var/lib/rabbitmq/.tos-admin-password.new
            mv /var/lib/rabbitmq/.tos-admin-password.new /var/lib/rabbitmq/.tos-admin-password
          fi
          {
            printf 'default_user = tosadmin\ndefault_pass = '
            cat /var/lib/rabbitmq/.tos-admin-password
            printf '\n'
          } > /etc/rabbitmq/conf.d/30-tos.conf
          chown rabbitmq:rabbitmq /etc/rabbitmq/conf.d/30-tos.conf
        )
        exec /usr/local/bin/docker-entrypoint.sh rabbitmq-server
x-app-meta:
  web:
    port: 15672
    protocol: http
```