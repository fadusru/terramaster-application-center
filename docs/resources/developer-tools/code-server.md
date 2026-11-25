# Code-server

20261125
Приложение установилось. Контейнер штатно не запускается, нужно разбираться.

Текст yaml
```yaml
services:
  codeserver-prepare:
    image: 'codercom/code-server:latest'
    container_name: codeserver-prepare
    volumes:
      - 'codeserver-data:/home/coder'
    restart: 'no'
    user: '0:0'
    entrypoint:
      - sh
      - '-c'
      - 'chown -R 1000:1000 /home/coder'
  codeserver:
    depends_on:
      codeserver-prepare:
        condition: service_completed_successfully
    env_file:
      - .env
    image: 'codercom/code-server:latest'
    container_name: codeserver
    environment:
      - 'PASSWORD=${CODE_SERVER_PASSWORD}'
      - 'TZ=${TZ}'
    volumes:
      - 'codeserver-data:/home/coder'
    ports:
      - '8100:8080'
    restart: always
name: codeserver
volumes:
  codeserver-data: null
x-app-meta:
  web:
    port: 8100
    protocol: http
```