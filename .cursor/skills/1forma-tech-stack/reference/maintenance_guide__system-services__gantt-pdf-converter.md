# Сервис экспорта PDF

Источник: https://help.1forma.ru/maintenance_guide/system-services/gantt-pdf-converter/

Сервис используется для экспорта проекта в PDF файл в проектном управлении "Первой Формы". После настройки необходимо указать адрес подключения к сервису в параметре ganttExportPdfUrl кастомной настройки приложения custom-app-settings.

Установите Docker и Docker Compose с официальных ресурсов под нужный дистрибутив.
 `docker login docker.1forma.ru
` `Username: USER
Password:
Login Succeeded
` Создайте файл docker-compose.yml:
 `services:

 web:

   container_name: gantt-pdf-converter

   image: "docker.1forma.ru/1forma/gantt-pdf-converter:latest"

   ports:

     - "8083:8080"

   healthcheck:

     test: ["CMD", "curl", "-f", "http://localhost"]

     interval: 30s

     timeout: 10s

     retries: 3

   restart: unless-stopped
` где:

-  image — используется образ docker.1forma.ru/1forma/gantt-pdf-converter:latest.


-  container_name — читаемое имя контейнера — gantt-pdf-converter.


-  ports — перенаправление хоста контейнера.


-  healthcheck:

  Проверяется способность контейнера отвечать на PING через CLI:

-  interval: 30s — интервал проверки;


-  timeout: 10s — тайм-аут выполнения;


-  retries: 3 — контейнер считается нездоровым после 3 неудачных попыток.


-  restart: unless-stopped: автоматический перезапуск контейнера при сбоях.

  Запустите сервис командой:
 `docker compose up -d`

Обычный сценарий (прокси на локальный сервер)
 Используется, если необходимо просто перенаправить входящий трафик на локальный сервер с PDF Export.
 Готовая конфигурация Nginx:
 `upstream docservice {

server backendserver-address;
` }
 `map $http_host $this_host {
` "" $host;
 default $http_host;
 }
 `map $http_x_forwarded_proto $the_scheme {
` default $http_x_forwarded_proto;
 "" $scheme;
 }
 `map $http_x_forwarded_host $the_host {
` default $http_x_forwarded_host;
 "" $this_host;
 }
 `map $http_upgrade $proxy_connection {
` default upgrade;
 "" close;
 }
 `proxy_set_header Upgrade $http_upgrade;

proxy_set_header Connection $proxy_connection;

proxy_set_header X-Forwarded-Host $the_host;

proxy_set_header X-Forwarded-Proto $the_scheme;

proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

server {

listen 0.0.0.0:80;

listen [::]:80 default_server;

server_tokens off;

location / {

proxy_pass http://docservice/;

proxy_http_version 1.1;

add_header 'Access-Control-Allow-Origin' 'APP_URL'; # Замените APP_URL на реальный URL вашего сервера 1Ф, например: https://1forma.company.ru

add_header 'Access-Control-Allow-Methods' 'GET ,POST, OPTIONS';

add_header 'Access-Control-Allow-Headers' '*';
` }
 }
 Виртуальный путь (виртуальный префикс)
 Подходит в ситуациях, когда нужно разместить PDF Export по определённому URL-префиксу (например, /pdfExport/ ) в рамках существующего веб-приложения.
 `upstream docservice {

server backendserver-address;
` }
 `map $http_x_forwarded_proto $the_scheme {
` default $http_x_forwarded_proto;
 "" $scheme;
 }
 `map $http_x_forwarded_host $the_host {
` default $http_x_forwarded_host;
 "" $host;
 }
 `map $http_upgrade $proxy_connection {
` default upgrade;
 "" close;
 }
 `proxy_set_header Upgrade $http_upgrade;

proxy_set_header Connection $proxy_connection;

proxy_set_header X-Forwarded-Host $the_host/office;

proxy_set_header X-Forwarded-Proto $the_scheme;

proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

server {

listen 0.0.0.0:80;

listen [::]:80 default_server;

server_tokens off;

location /pdfExport/ {

proxy_pass http://docservice/;

proxy_http_version 1.1;

add_header 'Access-Control-Allow-Origin' *;

add_header 'Access-Control-Allow-Methods' 'GET ,POST, OPTIONS';

add_header 'Access-Control-Allow-Headers' '*';
` }
 }
 где:
 APP_URL — адрес приложения Первой Формы, обращающегося к сервису экспорта PDF.
 ℹ️  Мы рекомендуем работу с виртуальным префиксом домена "Первой Формы"
