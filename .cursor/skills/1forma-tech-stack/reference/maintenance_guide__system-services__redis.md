# Redis

Источник: https://help.1forma.ru/maintenance_guide/system-services/redis/

Для скалирования SignalR в .NET-приложении (бэкенде) "Первая Форма" необходимо использовать Redis как бэкплейн.
 Redis — это хранилище данных в памяти, которое часто применяется как кэш, а также использует протокол pub/sub (publish/subscribe) для распределения сообщений между несколькими экземплярами приложения.
 При использовании Redis-бэкплейна все серверы обмениваются сигналами через Redis — сообщения, отправленные одним экземпляром, поступают всем остальным.

Операционная система
 Linux x86-64 с поддержкой Docker (например, Ubuntu/Debian).
 CPU
 Минимум 2 vCPU (рекомендуется ≥2 ядра при высокой нагрузке). Redis — однопоточное приложение, но масштабирование возможно через несколько инстансов.
 Память (RAM)
 Минимум 1GB (рекомендуется ≥4GB).
 Сеть
 1Gbps интерфейс достаточен. Для продакшн-кластера/высокой нагрузки рекомендуется — 10Gbps или multi-NIC.

Для обеспечения максимальной совместимости с .NET-приложением "Первая Форма" рекомендуется использовать преднастроенный Docker-образ Redis, подготовленный компанией:
 `docker-public.1forma.ru/redis`

-  Образ оптимизирован под использование в связке c другими компонентами приложения "Первая Форма".


-  Не требует авторизации для загрузки (анонимный pull доступ).

Для защиты Redis от несанкционированного доступа рекомендуется использовать парольную аутентификацию.
 Через конфигурацию Redis
 Используйте параметр requirepass:
 `redis-server --requirepass "ваш_пароль"`
 Через переменную окружения
 В Docker-образе от компании "Первая Форма" пароль можно задать через переменную окружения:
 `REDIS_PASSWORD=ваш_пароль`
 Требования к паролю

-  Рекомендуемая длина: не менее 32 символов


-  Пароль должен быть случайным и непредсказуемым

  Генерация безопасного пароля
 Можно использовать команду:
 `openssl rand -hex 16`
 Вы получите случайную hex-строку длиной 32 символа.

Для сохранения данных Redis между перезапусками контейнера требуется настроить volume с правильными правами доступа.
 `mkdir ./redis-data`
 `chown -R 1001:1001 ./redis-data`
 Почему UID/GID 1001?
 Docker-образ Redis от Первая Форма запускается под uid 1001, чтобы избежать запуска под root. Если volume не принадлежит этому uid/gid, при монтировании Redis не сможет записывать данные (возникнет ошибка "permission denied").

Для запуска Redis в Docker выполните:
 `docker run -d \\
  --name redis \\
  -p 6379:6379 \\
  -e REDIS_PASSWORD=ваш_пароль \\
  -v ./redis-data:/1f/redis/data \\
  docker-public.1forma.ru/redis:latest
`
-  -p 6379:6379 — Перенаправляет локальный порт 6379 на стандартный порт Redis 6379 внутри контейнера.


-  -e REDIS_PASSWORD=... — Задаёт пароль через переменную окружения.


-  docker-public.1forma.ru/redis:latest — Образ Redis.

  Рекомендуется использовать случайно сгенерированный пароль длиной не менее 32 символов (см. выше).

Для более удобного и воспроизводимого развёртывания Redis рекомендуется использовать Docker Compose с внешним .env-файлом.
 Создайте docker-compose.yml
 `services:

 redis:

   image: docker-public.1forma.ru/redis:latest

   container_name: redis

   ports:

     - '6379:6379'

   volumes:

     - ./redis-data:/1f/redis/data:Z

   environment:

     - REDIS_PASSWORD

     - TZ

   healthcheck:

     test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]

     interval: 30s

     timeout: 10s

     retries: 3

   restart: unless-stopped

   networks:

     service_virtual_network:

networks:

 service_virtual_network:
`
-  image: используется образ docker-public.1forma.ru/redis:latest


-  container_name: читаемое имя контейнера - redis .


-  ports: перенаправление хоста 6379 → 6379 контейнера.


-  volumes:


-  ./redis-data:/1f/redis/data:Z — сохранение данных на хосте в директории `./redis-data`


-  environment:


-  REDIS_PASSWORD подтягивается из .env, позволяет задавать пароль безопасно.


-  TZ задаёт часовой пояс внутри контейнера.


-  healthcheck:

  Проверяется способность Redis отвечать на PING через CLI с аутентификацией:

-  interval: 30s — интервал проверки;


-  timeout: 10s — тайм-аут выполнения;


-  retries: 3 — контейнер считается нездоровым после 3 неудачных попыток.


-  restart: unless-stopped: автоматический перезапуск контейнера при сбоях.


-  networks: подключение к виртуальной сети `service_virtual_network`, обеспечивая доступ к контейнеру Redis по имени `redis` из других сервисов в той же сети.

  Рядом с docker-compose.yml файлом создайте .env
 `REDIS_PASSWORD=ваш_пароль`
 `TZ=Europe/Moscow`

-  REDIS_PASSWORD — Задает пароль для Redis; рекомендуется использовать hex-генерацию через openssl rand -hex 16.


-  TZ — Устанавливает часовой пояс, чтобы логи и метки времени в контейнере соответствовали вашему региону.

  Запустите сервис:
 `docker compose up -d`
 Контейнер:

-  запускается в фоновом режиме;


-  контролируется на здоровье;


-  автоматически перезапускается при сбоях.

По умолчанию Redis не поддерживается непосредственно в Windows. Есть несколько способов его запустить.
 Вариант 1: MSI от "Первая Форма"
 Скачайте подготовленный MSI-пакет с хранилища "Первой Формы" и установите Redis как обычное Windows-приложение.
 Ссылка на скачивание
 Вариант 2: Docker (Linux-контейнер)
 Самый надежный способ — запуск Linux-образа Redis от "Первая Форма" с помощью Docker Desktop:
 `docker run -d \
  --name redis-windows \
  -p 6379:6379 \
  -e REDIS_PASSWORD=ваш_пароль \
  docker-public.1forma.ru/redis:latest
` Вариант 3: Chocolatey
 Chocolatey — это менеджер пакетов для Windows, аналогичный apt или yum в Linux.
 Для установки Chocolatey запустите PowerShell от имени администратора. Выполните команду:
 `Set-ExecutionPolicy Bypass -Scope Process -Force`
 `iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))`
 Проверьте установку:
 `choco -v`
 Установите Redis в Windows с помощью Chocolatey:
 `choco install redis-64`
 После установки можно запустить сервис:
 `redis-server.exe`
 Или установить как службу:
 `redis-server --service-install`
 `redis-server --service-start`

Использование в appsettings.json
 Добавьте строку подключения к Redis для использования как backplane SignalR:
 `"SignalRedisConnectionString": "redis:6379,Password=ваш_пароль"`
 ℹ️ В зависимости от способа запуска параметр задаётся по-разному: в `appsettings.json` — `SignalRedisConnectionString`, через переменную окружения Docker — `CORE_SIGNAL_REDIS_CONNECTION_STRING`.

-  redis:6379 — Адрес контейнера Redis в Docker Compose (имя сервиса + порт).


-  Password=... — Безопасный пароль вашей установки Redis.

  Исполнение через переменные окружения
 Альтернативный способ — задать строку подключения через переменные окружения (например, в docker compose или k8s).
 `SIGNAL_REDIS_CONNECTION_STRING="redis:6379,Password=ваш_пароль"`
