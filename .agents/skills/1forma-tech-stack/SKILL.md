---
name: 1forma-tech-stack
description: >-
  Стек технологий и установка «Первой Формы»: .NET 10 / C# 14 / ASP.NET Core,
  Angular SPA (1F-SPA + Nginx), dual DB MS SQL 2019+ и PostgreSQL 16+, Redis,
  SignalR/WebSocket, WebDAV, SOAP, Docker Compose, Kubernetes, IIS, 1F-Core,
  1F-dbDeploy, OnlyOffice/R7, MinIO/S3. Use when the user asks about 1Forma
  architecture, components, env vars, deployment, databases, Redis, SPA modes,
  or installation. Official source:
  https://help.1forma.ru/maintenance_guide/installation/tech_stack/
---

# Стек технологий и установка 1Forma

Используй этот скилл для **архитектуры, состава сервисов, требований и установки**. Не путай с пользовательским и админским руководствами.

Полные страницы — в `reference/` (установка + системные службы). Карта: `reference/INDEX.md`.

Источник: [Стек технологий](https://help.1forma.ru/maintenance_guide/installation/tech_stack/). Снимок: 2026-08-13. Связанная страница SPA: [1F-SPA](https://help.1forma.ru/maintenance_guide/system-services/1f-spa/).

## Стек (канон со страницы tech_stack)

### Backend и языки

| Технология | Роль |
|---|---|
| **.NET 10 / C# 14, ASP.NET Core** | Основной backend. ~57 модулей `Valhalla.*`, ~1515 API-эндпоинтов |
| **MS SQL Server 2019+ / PostgreSQL 16+** | Dual-database. Основное хранилище |
| **SignalR** | Real-time; на клиенте через Service Worker |
| **Redis** | Кэш, сессии, очереди; **backplane SignalR** при нескольких инстансах Core |
| **Python** | DevOps-скрипты, автоматизация инфраструктуры (не основной runtime приложения) |

### Frontend

| Технология | Роль |
|---|---|
| **Angular** | SPA (1F-SPA). Раздача через **Nginx** |
| **JavaScript** | Основной язык клиента |
| **jQuery** | Ограниченно, наследие и Типовые решения (ТС) |

Режимы 1F-SPA:

- **Standalone** — только статика, без прокси на backend (`SPA_STANDALONE=true`, дефолт).
- **Reverse Proxy** — статика + прокси API на Core (`SPA_STANDALONE=false`, нужен `SPA_PROXY_CORE_URL`, обычно `http://1f-core:8000`).

В reverse-proxy Nginx проксирует `/notificationHub`, `/pushHub`, `/signalr`, `/denormalizationHub` с Upgrade (WebSocket), без буферизации.

### Данные и кэш

Redis — см. backend. Опционально MinIO/S3 для файлов (TUS pre-upload: Local или S3).

### Протоколы

| Протокол | Зачем |
|---|---|
| **WebSocket (wss)** | Мгновенные обновления UI |
| **WebDAV** | Удалённые файлы, совместная работа с документами |
| **XML (SOAP)** | Интеграции с enterprise-сервисами, которые требуют SOAP |

## Сервисы комплекса

Минимум для работы:

1. **1F-CORE** — API и бизнес-логика (.NET). Порт HTTP по умолчанию `8000`. Флаг `CORE_IS_JOB_SERVER=true` включает фоновые джобы. `CORE_IS_MAIN_SERVER` — джобы, если job-сервер не выделен. `CORE_APPLICATION_INSTANCE_ID` уникален, буквы/цифры/`_`, **без дефиса**.
2. **1F-SPA** — Angular + Nginx.
3. **СУБД** — MSSQL или PostgreSQL (в контейнере или внешняя).
4. **1F-dbDeploy** — однократно до Core: накатывает SQL-миграции. Тип БД `mssql` / `postgresql`. Для PG пользователь миграций по умолчанию `migrationsdaemon`.

Опционально:

- **Redis** — обязателен при горизонтальном масштабировании SignalR. Образ `docker-public.1forma.ru/redis`. Volume от uid **1001**. Пароль ≥32 символов (`REDIS_PASSWORD` / `requirepass`).
- **OnlyOffice / Р7 Docs** — онлайн-редактор. SPA: `SPA_PROXY_ONLYOFFICE_URL` (URL **со слэшем в конце**). Core: `CORE_R7_SECRET_KEY` (JWT, ≥16 символов; до 2.267.512 имя было `R7_SECRET_KEY`).
- **MinIO/S3** — файлы.
- **Matomo** — аналитика через CustomSettings.
- **Gantt PDF converter**, **MPP importer** — отдельные службы (см. `reference/maintenance_guide__system-services__*`).

Секреты в контейнерах: переменная `FOO_FILE=/run/secrets/...` читается в `FOO` при старте (Docker/K8s secrets).

## Конфигурация Core (важное)

`CORE_DB_TYPE`: `mssql` | `postgresql`.  
`CORE_DB_NAME` / пользователь по умолчанию: `d10task` / `d10taskuser`.  
Rebus (шина): по умолчанию та же БД; для PG пользователь rebus. `REBUS_MESSAGE_BUS`: RebusSQL / RebusPostgre / Redis / None.

SMART-запросы можно вынести на read-only: `CORE_ENABLE_SMART_CONNECTION_STRING`, `SMART_DB_USER` (`d10taskreader`).

Redis для SignalR: `CORE_SIGNAL_REDIS_CONNECTION_STRING`.

Auth: токены локальные (срок **не** берётся из SAML/OIDC). Дефолт access ~1500 мин. Login SPA: `~/spa/entry/signin`.

Публикация **по HTTPS** обязательна. Одна нода — HTTPS на frontend; несколько — HTTP на нодах + SSL-терминация на балансировщике (рекомендуют Nginx).

## Варианты установки

| Способ | Когда | Reference |
|---|---|---|
| **Docker Compose** | Linux, типовой контур | `...__docker_compose.md` |
| **Kubernetes** | кластер; отдельно чарт ВКС | `...__kubernetes.md`, `...__kubernetes_vks.md` |
| **IIS** | Windows Server, классика | `...__iis-deployment__*` |

Compose: Docker Engine 24+, Compose 2.17+. Минимум **4 vCPU / 8 GB RAM / 35 GB**. Образы из **приватного** `docker.1forma.ru` (`docker login`). Комплект: `https://download.1forma.ru/raw-1f-app/configurations/1forma-compose.zip` → `/opt/1forma`, `cp env.example .env`.

Обязательные переменные запуска: `CORE_VERSION`, `SPA_VERSION`, `CORE_DB_*`, Rebus user/password, `CORE_APPLICATION_INSTANCE_ID`.

IIS (кратко): Windows Server 2019+, пул **.NET 4.8**, 1 worker, 32-bit = False, AlwaysRunning, idle timeout 0, identity LocalSystem (или служебная учётка). Системный язык сервера — русский.

ОС сервера (из требований): Windows Server 2019+ / Ubuntu 22.04 LTS / Debian 10+ / РЕД ОС 7.3 / Astra Linux.

## Базы данных

**MSSQL 2019+:** collation `Cyrillic_General_CI_AS`, Mixed mode, Full-Text, MaxDOP 4, Optimize for Ad hoc Workloads. SSMS ≥ 17.3. Express только для <50 пользователей (лимит 10 ГБ).

**PostgreSQL 16+** (в доке также подготовка PG 18 / Postgres Pro 18): для русского FTS критичен словарь `russian` (`to_tsvector('russian', ...)`). Иначе `websearch_to_tsquery` без языка идёт в english и не находит русские документы. Patroni — отказоустойчивый кластер. Миграция 16→18 — отдельная страница.

Не смешивай диалекты смарт-SQL: то, что написано под MSSQL, на PG не «само поедет» (см. lua-pg и sql_smart в reference).

## Карта reference

Стек и службы:

- `reference/maintenance_guide__installation__tech_stack.md`
- `reference/maintenance_guide__system-services__1f-spa.md`
- `reference/maintenance_guide__system-services__1f-core.md`
- `reference/maintenance_guide__system-services__1f-dbdeploy.md`
- `reference/maintenance_guide__system-services__redis.md`
- `reference/maintenance_guide__system-services__redis-setup.md`
- `reference/maintenance_guide__system-services__gantt-pdf-converter.md`
- `reference/maintenance_guide__system-services__mpp-importer.md`

Установка приложения: `reference/maintenance_guide__installation__application__*`  
MSSQL: `...__databases__mssql__*`  
PostgreSQL: `...__databases__postgresql__*`

Полный список файлов — `reference/INDEX.md`.

## Как отвечать

1. Назови компоненты стека терминами из tech_stack (не подменяй Angular на «React», не забывай dual-DB).
2. Для env vars и портов открывай страницу конкретного сервиса — не сокращай таблицы по памяти.
3. Различай публичный Redis-образ `docker-public.1forma.ru/redis` и приватный registry приложения `docker.1forma.ru`.
4. Установка и смена прод-конфига — только если пользователь явно попросил; иначе опиши шаги, не выполняй их на живой площадке.

## Чего не делать

- Не утверждать, что поддерживается только MSSQL или только PG.
- Не рекомендовать SPA Standalone как единственный прод-режим, если API должен идти через тот же хост.
- Не запускать Redis-volume без uid 1001.
- Не оставлять дефис в `CORE_APPLICATION_INSTANCE_ID`.
- Не публиковать прод без HTTPS (или SSL на балансировщике).
