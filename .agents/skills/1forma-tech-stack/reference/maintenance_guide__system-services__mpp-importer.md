# Сервис импорта Mpp

Источник: https://help.1forma.ru/maintenance_guide/system-services/mpp-importer/

Mpp-Importer — это инструмент обработки mpp-файлов в Диаграмму Ганта в формате JSON. Используется для загрузки файла проекта с расширением .mpp, созданного в Microsoft Project, в проектном управлении "Первой Формы". После настройки необходимо указать адрес подключения к сервису в параметре ganttImportMppUrl кастомной настройки приложения custom-app-settings.

Операционная система
 Linux x86-64 с поддержкой Docker (например, Ubuntu/Debian).
 CPU
 Минимум 1 vCPU (рекомендуется ≥ 2 ядра при высокой нагрузке).
 Память (RAM)
 Минимум 1 GB (рекомендуется ≥ 2 GB).

Для обеспечения максимальной совместимости с .NET-приложением "Первая Форма" рекомендуется использовать преднастроенный Docker-образ mpp-importer, подготовленный компанией docker-public.1forma.ru/mpp-importer.
 ℹ️ Актуальный адрес образа уточняйте у службы технической поддержки. Адрес зависит от вашего договора и условий поставки.
 Образ оптимизирован под использование в связке c другими компонентами приложения "Первая Форма". Не требует авторизации для загрузки (анонимный pull доступ).

Установите Docker и Docker Compose с официальных ресурсов под нужный дистрибутив.
 Выполните docker login. Используйте логин и пароль, полученные у сотрудников "Первой Формы".
 `docker login docker.1forma.ru
` `Username: USER
Password:
Login Succeeded
` Создайте файл docker-compose.yml:
 `services:

 mpp-importer:

   image: docker.1forma.ru/1forma/mpp-gantt-converter:latest

   container_name: mpp-gantt-converter

   ports:

     - "8084:80"

   healthcheck:

     test: ["CMD", "curl", "-f", "http://localhost"]

     interval: 30s

     timeout: 10s

     retries: 3

   restart: unless-stopped
` где:

-  image — используется образ docker.1forma.ru/1forma/mpp-gantt-converter:latest.


-  container_name — читаемое имя контейнера — mpp-gantt-converter.


-  ports — перенаправление хоста контейнера.


-  healthcheck:

  Проверяется способность контейнера отвечать на PING через CLI:

-  interval: 30s — интервал проверки;


-  timeout: 10s — тайм-аут выполнения;


-  retries: 3 — контейнер считается нездоровым после 3 неудачных попыток.


-  restart: unless-stopped: автоматический перезапуск контейнера при сбоях.

  Запустите сервис:
 `docker compose up -d`
 Контейнер запускается в фоновом режиме, контролируется на предмет работоспособности. В случае сбоев осуществляется автоматический перезапуск.

Обычный сценарий (прокси на локальный сервер)
 Используется, если необходимо просто перенаправить входящий трафик на локальный сервер с MPP import.
 Готовая конфигурация Nginx:
 upstream docservice {
 server backendserver-address;
 }
 map $http_host $this_host {
 "" $host;
 default $http_host;
 }
 map $http_x_forwarded_proto $the_scheme {
 default $http_x_forwarded_proto;
 "" $scheme;
 }
 map $http_x_forwarded_host $the_host {
 default $http_x_forwarded_host;
 "" $this_host;
 }
 map $http_upgrade $proxy_connection {
 default upgrade;
 "" close;
 }
 proxy_set_header Upgrade $http_upgrade;
 proxy_set_header Connection $proxy_connection;
 proxy_set_header X-Forwarded-Host $the_host;
 proxy_set_header X-Forwarded-Proto $the_scheme;
 proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
 server {
 listen 0.0.0.0:80;
 listen [::]:80 default_server;
 server_tokens off;
 location / {
 proxy_pass http://docservice/php/load.php; # Примечание: путь /php/load.php унаследован из конфигурации OnlyOffice. Уточните актуальный путь в документации к используемой версии mpp-importer.
 proxy_http_version 1.1;
 add_header 'Access-Control-Allow-Origin' 'APP_URL';
 add_header 'Access-Control-Allow-Methods' 'GET ,POST, OPTIONS';
 add_header 'Access-Control-Allow-Headers' '*';
 }
 }
 Виртуальный путь (виртуальный префикс)
 Подходит в ситуациях, когда нужно разместить сервис импорта по определенному URL-префиксу (например, /mppImporter/ ) в рамках существующего веб-приложения:
 upstream docservice {
 server backendserver-address;
 }
 map $http_x_forwarded_proto $the_scheme {
 default $http_x_forwarded_proto;
 "" $scheme;
 }
 map $http_x_forwarded_host $the_host {
 default $http_x_forwarded_host;
 "" $host;
 }
 map $http_upgrade $proxy_connection {
 default upgrade;
 "" close;
 }
 proxy_set_header Upgrade $http_upgrade;
 proxy_set_header Connection $proxy_connection;
 proxy_set_header X-Forwarded-Host $the_host/office;
 proxy_set_header X-Forwarded-Proto $the_scheme;
 proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
 server {
 listen 0.0.0.0:80;
 listen [::]:80 default_server;
 server_tokens off;
 location /mppImporter/ {
 proxy_pass http://docservice/php/load.php;
 proxy_http_version 1.1;
 add_header 'Access-Control-Allow-Origin' 'APP_URL';
 add_header 'Access-Control-Allow-Methods' 'GET ,POST, OPTIONS';
 add_header 'Access-Control-Allow-Headers' '*';
 }
 }
 где:
 APP_URL — адрес приложения Первой Формы, обращающегося к сервису импорта MPP.
 ℹ️  Мы рекомендуем работу с виртуальным префиксом домена "Первой Формы"
