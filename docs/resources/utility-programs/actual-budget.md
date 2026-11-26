# Actual Budget

20261126
Приложение открывается по адресу http://ip-nas:8033 с ошибкой:
```
Actual requires access to SharedArrayBuffer in order to function properly. If you're seeing this error, 
either your browser does not support SharedArrayBuffer, or your server is not sending the appropriate headers, 
or you are not using HTTPS. See our troubleshooting documentation to learn more. Advanced options
```
Текст yaml:
```yaml
services:
  actualbudgetdocker:
    healthcheck:
      test:
        - CMD-SHELL
        - 'node -e "fetch(''http://127.0.0.1:5006/'').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"'
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    image: 'actualbudget/actual-server:26.11.0'
    container_name: actualbudgetdocker
    environment:
      - TZ=Europe/Moscow
    volumes:
      - '/Volume1/DockerAppData/actualbudgetdocker/data:/data'
    ports:
      - '8033:5006'
    restart: unless-stopped
    user: '1000:1000'
x-app-meta:
  web:
    port: 8033
    protocol: http
```

20261123
Приложение открывается по адресу http://ip-nas:8033 с ошибкой:
```
Fatal Error
Actual requires access to SharedArrayBuffer in order to function properly. If you're seeing this error, 
either your browser does not support SharedArrayBuffer, or your server is not sending the appropriate headers, 
or you are not using HTTPS. See our troubleshooting documentation to learn more. Advanced options
```

Текст yaml
```yaml
services:
  actualbudgetdocker:
    healthcheck:
      test:
        - CMD-SHELL
        - 'node -e "fetch(''http://127.0.0.1:5006/'').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"'
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    image: 'actualbudget/actual-server:26.11.0'
    container_name: actualbudgetdocker
    environment:
      - TZ=Europe/Moscow
    volumes:
      - '/Volume1/DockerAppData/actualbudgetdocker/data:/data'
    ports:
      - '8033:5006'
    restart: unless-stopped
    user: '1000:1000'
x-app-meta:
  web:
    port: 8033
    protocol: http
```