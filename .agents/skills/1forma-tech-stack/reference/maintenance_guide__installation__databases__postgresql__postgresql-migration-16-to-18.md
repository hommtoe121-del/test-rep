# Миграция PostgreSQL 16 до PostgreSQL 18

Источник: https://help.1forma.ru/maintenance_guide/installation/databases/postgresql/postgresql-migration-16-to-18/

⚠️ Перед началом миграции остановите приложение (сервис 1f-core). Миграция требует полной остановки записи в базу данных.

Стратегия миграции
 Миграция выполняется методом dump-and-restore с параллельной работой обеих версий:

-  PostgreSQL 16 остаётся на порту 5432 (рабочая версия)


-  PostgreSQL 18 устанавливается на порту 5433 (новая версия)

  После тестирования порты меняются местами.
 Преимущества метода

-  Минимальное время простоя


-  Возможность быстрого отката


-  Параллельное тестирование


-  Безопасность данных

  Временные затраты
 Размер БД Время дампа Время восстановления Общее время < 10 GB 5-10 мин 10-15 мин ~30 мин 10-50 GB 15-30 мин 30-60 мин ~2 часа 50-100 GB 30-60 мин 1-2 часа ~3-4 часа > 100 GB 1-2 часа 3-5 часов ~6-8 часов `flowchart TD
    A["PostgreSQL 16 на порту 5432 (рабочая)"] --> B["Установка PostgreSQL 18 на порт 5433"]
    B --> C["Дамп базы из PG16"]
    C --> D["Восстановление дампа в PG18"]
    D --> E["Настройка прав и разрешений"]
    E --> F["Тестирование PG18 параллельно с PG16"]
    F --> G{"Тестирование успешно?"}
    G -- "да" --> H{"Способ переключения"}
    H -- "Вариант A (рекомендуется)" --> I["Поменять порты местами: PG18 на 5432"]
    H -- "Вариант B" --> J["Отключить PostgreSQL 16"]
    G -- "нет" --> K["Откат на PostgreSQL 16 (остался на 5432)"]`

Шаг 1: Проверка текущей версии
 Проверка версии PostgreSQL 16:
 `psql --version
` Проверка статуса службы:
 `sudo systemctl status postgresql@16-main.service
` Шаг 2: Резервное копирование конфигураций
 Создание директории для бэкапов:
 `sudo mkdir -p /opt/migration-backup

cd /opt/migration-backup
` Копирование конфигураций PostgreSQL 16
 `sudo cp /etc/postgresql/16/main/postgresql.conf ./postgresql-16.conf.backup

sudo cp /etc/postgresql/16/main/pg_hba.conf ./pg_hba-16.conf.backup
` Шаг 3: Резервное копирование директории данных
 Определение расположения data_directory:
 `sudo -u postgres psql -p 5432 -t -c "SHOW data_directory;"
` Остановка PostgreSQL 16 перед копированием данных:
 `sudo systemctl stop postgresql@16-main.service
` Копирование всей директории данных PostgreSQL 16 (путь может отличаться, проверьте вывод предыдущей команды):
 `sudo cp -a /pgdata/16-main /opt/migration-backup/pg16-data-backup
` Запуск PostgreSQL 16:
 `sudo systemctl start postgresql@16-main.service
` Проверка, что служба запустилась:
 `sudo systemctl status postgresql@16-main.service
`

Шаг 1: Установка PostgreSQL 18
 Выполните установку PostgreSQL 18 согласно документации.
 Шаг 2: Инициализация кластера (если требуется)
 После установки кластер может не создаться автоматически. В этом случае инициализируйте его вручную:
 Проверка существующих кластеров:
 `pg_lsclusters
` Если кластер 18/main отсутствует — создайте его:
 `sudo pg_createcluster 18 main --start --port=5433
` Шаг 3: Изменение порта на 5433 (если кластер создан автоматически)
 Если кластер был создан автоматически на порту 5432, измените порт чтобы избежать конфликта с PostgreSQL 16:
 `sudo vim /etc/postgresql/18/main/postgresql.conf
` Найдите и измените параметр port:
 port = 5433
 Шаг 4: Запуск PostgreSQL 18
 `sudo systemctl start postgresql@18-main.service

sudo systemctl status postgresql@18-main.service
`

⚠️ Перед созданием дампа необходимо удалить расширения из базы данных (шаги ниже). Выполните DROP EXTENSION до запуска pg_dump.
 Шаг 1: Создание директории для дампов
 `sudo mkdir -p /opt/migration-backup/dumps

cd /opt/migration-backup/dumps
` Шаг 2: Проверка версии клиента
 Важно: Используйте клиент PostgreSQL 18 для создания дампа!
 Проверка версии клиента:
 `pg_dump --version
` Ожидаемый результат: pg_dump (PostgreSQL) 18.x
 Если версия клиента 16.x, обновите путь:
 `export PATH=/usr/lib/postgresql/18/bin:$PATH

pg_dump --version
` Шаг 3: Создание дампа базы d10task
 `pg_dump -h localhost \

       -p 5432 \

       -U dbo \

       -Fc \

       -v \
` `       --exclude-extension=* \

       -f /opt/migration-backup/dumps/d10task_no_ext.dump

       d10task
` Важно: перед созданием дампа необходимо удалить расширения, если они установлены в БД
 `drop extension tds_fdw cascade;

drop extension http cascade;

drop extension postgres_fdw cascade;
` Параметры команды:
 -h localhost — хост PostgreSQL 16
 -p 5432 — порт PostgreSQL 16
 -U dbo — пользователь с правами на базу
 -Fc — формат custom (сжатый)
 -v — verbose режим (показывать прогресс)
 `--exclude-extension=* — исключить расширения из дампа
` -f — путь к файлу дампа
 Шаг 4: Создание дампа базы d10task_file
 `pg_dump -h localhost \

       -p 5432 \

       -U dbo \

       -Fc \

       -v \
` `       --exclude-extension=* \

       -f /opt/migration-backup/dumps/d10task_file_no_ext.dump

       d10task_file
`

Шаг 1: Создание пользователей
 Подключитесь к PostgreSQL 18:
 `sudo -u postgres psql -p 5433
` Создайте пользователей:
 `CREATE USER dbo WITH SUPERUSER CREATEDB CREATEROLE INHERIT LOGIN PASSWORD 'CHANGE_ME_pass1';

CREATE USER d10taskuser WITH INHERIT LOGIN PASSWORD 'CHANGE_ME_pass2';

CREATE USER migrationsdaemon WITH INHERIT LOGIN PASSWORD 'CHANGE_ME_pass3';

CREATE USER rebus WITH INHERIT LOGIN PASSWORD 'CHANGE_ME_pass4';

CREATE USER implementer WITH INHERIT LOGIN PASSWORD 'CHANGE_ME_pass5';

-- Роль для SMART тестирования

CREATE ROLE role_D10TaskReader WITH INHERIT;

CREATE USER D10TaskReader WITH INHERIT LOGIN PASSWORD 'CHANGE_ME_pass6';

GRANT role_D10TaskReader TO D10TaskReader;

ALTER USER D10TaskReader SET search_path = "dbo", "public";
` \q
 Важно: используйте те же пароли, что и в PostgreSQL 16.
 Шаг 2: Создание базы данных d10task
 `sudo -u postgres psql -p 5433
` `CREATE DATABASE d10task

   TEMPLATE template0

   OWNER dbo

   ENCODING 'utf8'

   LC_COLLATE 'ru_RU.utf8'

   LC_CTYPE 'ru_RU.utf8';

-- Критично для русского полнотекстового поиска: FTS-индексы строятся через to_tsvector('russian', ...),

-- без этого параметра websearch_to_tsquery без явного language идёт по english словарю.

ALTER DATABASE d10task SET default_text_search_config = 'pg_catalog.russian';
` \q
 Шаг 3: Активация расширений в d10task
 `sudo -u postgres psql -p 5433 -d d10task
` `-- Загрузка pg_hint_plan

LOAD 'pg_hint_plan';

-- Создание расширений

CREATE EXTENSION IF NOT EXISTS plpgsql SCHEMA pg_catalog;

CREATE EXTENSION IF NOT EXISTS rum SCHEMA public;

CREATE EXTENSION IF NOT EXISTS pgcrypto SCHEMA public;

CREATE EXTENSION IF NOT EXISTS pg_buffercache SCHEMA public;

CREATE EXTENSION IF NOT EXISTS btree_gin SCHEMA public;

CREATE EXTENSION IF NOT EXISTS pg_stat_statements SCHEMA public;

CREATE EXTENSION IF NOT EXISTS pg_trgm SCHEMA public;

CREATE EXTENSION IF NOT EXISTS pg_qualstats SCHEMA public;

CREATE EXTENSION IF NOT EXISTS vector SCHEMA public;

CREATE EXTENSION IF NOT EXISTS pg_background SCHEMA public;
` \q
 Шаг 4: Восстановление дампа d10task
 `pg_restore -p 5433 \

          -U dbo \

          -d d10task \

          -v \

          /opt/migration-backup/dumps/d10task_no_ext.dump
` Шаг 5: Создание базы данных d10task_file
 `sudo -u postgres psql -p 5433
` `CREATE DATABASE d10task_file

   TEMPLATE template0

   OWNER dbo

   ENCODING 'utf8'

   LC_COLLATE 'ru_RU.utf8'

   LC_CTYPE 'ru_RU.utf8';
` \q
 Шаг 6: Восстановление дампа d10task_file
 `pg_restore -p 5433 \

          -U dbo \

          -d d10task_file \

          -v \

          /opt/migration-backup/dumps/d10task_file_no_ext.dump
`

Шаг 1: Настройка схем и базовых разрешений
 `sudo -u postgres psql -p 5433 -d d10task
` `-- Настройка search_path

ALTER ROLE D10TaskUser SET search_path = "dbo", "public";

ALTER ROLE dbo SET search_path = "dbo", "public";

ALTER USER MigrationsDaemon SET search_path = "dbo", "public";

ALTER USER rebus SET search_path = "rebus", "public";

-- Права USAGE на схемы для d10taskuser

GRANT USAGE ON SCHEMA dbo, sys TO d10taskuser;

GRANT USAGE ON SCHEMA SignalR TO d10taskuser;

-- Назначение ролей

GRANT dbo, rebus TO MigrationsDaemon;

-- Разрешения для dbo и MigrationsDaemon

GRANT ALL ON ALL TABLES IN SCHEMA dbo, sys TO dbo, MigrationsDaemon;

GRANT ALL ON ALL SEQUENCES IN SCHEMA dbo, sys TO dbo, MigrationsDaemon;

GRANT ALL ON ALL FUNCTIONS IN SCHEMA dbo, sys TO dbo, MigrationsDaemon;

GRANT ALL ON SCHEMA dbo, sys TO dbo, MigrationsDaemon;

-- Разрешения по умолчанию

ALTER DEFAULT PRIVILEGES IN SCHEMA dbo, sys, SignalR GRANT ALL ON TABLES TO dbo, MigrationsDaemon;

ALTER DEFAULT PRIVILEGES IN SCHEMA dbo, sys, SignalR GRANT ALL ON SEQUENCES TO dbo, MigrationsDaemon;

ALTER DEFAULT PRIVILEGES IN SCHEMA dbo, sys, SignalR GRANT ALL ON FUNCTIONS TO dbo, MigrationsDaemon;

ALTER DEFAULT PRIVILEGES IN SCHEMA dbo, sys, SignalR GRANT ALL ON TYPES TO dbo, MigrationsDaemon;
` Шаг 2: Настройка для SMART тестирования
 `-- Права для role_D10TaskReader на основные схемы

GRANT USAGE ON SCHEMA dbo, sys, SignalR TO role_D10TaskReader;

GRANT SELECT ON ALL TABLES IN SCHEMA dbo, sys, SignalR TO role_D10TaskReader;

GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA dbo, sys, SignalR TO role_D10TaskReader;

GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA dbo, sys, SignalR TO role_D10TaskReader;

-- Разрешения по умолчанию

ALTER DEFAULT PRIVILEGES IN SCHEMA dbo, sys, SignalR GRANT SELECT ON TABLES TO role_D10TaskReader;

ALTER DEFAULT PRIVILEGES IN SCHEMA dbo, sys, SignalR GRANT EXECUTE ON FUNCTIONS TO role_D10TaskReader;

ALTER DEFAULT PRIVILEGES IN SCHEMA dbo, sys, SignalR GRANT USAGE, SELECT ON SEQUENCES TO role_D10TaskReader;

-- Права на схему custom

GRANT ALL ON ALL TABLES IN SCHEMA custom TO role_D10TaskReader;

GRANT ALL ON ALL SEQUENCES IN SCHEMA custom TO role_D10TaskReader;

GRANT ALL ON ALL FUNCTIONS IN SCHEMA custom TO role_D10TaskReader;

ALTER DEFAULT PRIVILEGES IN SCHEMA custom GRANT ALL ON TABLES TO role_D10TaskReader;

ALTER DEFAULT PRIVILEGES IN SCHEMA custom GRANT ALL ON SEQUENCES TO role_D10TaskReader;

ALTER DEFAULT PRIVILEGES IN SCHEMA custom GRANT ALL ON FUNCTIONS TO role_D10TaskReader;

ALTER DEFAULT PRIVILEGES IN SCHEMA custom GRANT ALL ON TYPES TO role_D10TaskReader;

GRANT ALL ON SCHEMA custom TO role_D10TaskReader;

-- Права на схему rebus

GRANT ALL ON ALL TABLES IN SCHEMA rebus TO role_D10TaskReader;

GRANT ALL ON ALL SEQUENCES IN SCHEMA rebus TO role_D10TaskReader;

GRANT ALL ON ALL FUNCTIONS IN SCHEMA rebus TO role_D10TaskReader;

ALTER DEFAULT PRIVILEGES IN SCHEMA rebus GRANT ALL ON TABLES TO role_D10TaskReader;

ALTER DEFAULT PRIVILEGES IN SCHEMA rebus GRANT ALL ON SEQUENCES TO role_D10TaskReader;

ALTER DEFAULT PRIVILEGES IN SCHEMA rebus GRANT ALL ON FUNCTIONS TO role_D10TaskReader;

ALTER DEFAULT PRIVILEGES IN SCHEMA rebus GRANT ALL ON TYPES TO role_D10TaskReader;

GRANT ALL ON SCHEMA rebus TO role_D10TaskReader;

-- Права на языки

GRANT USAGE ON LANGUAGE plpgsql, sql TO role_D10TaskReader;
` Шаг 3: Права для новой админки
 `-- Если есть схема dbadmin

GRANT USAGE ON SCHEMA dbadmin TO d10taskuser;

GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA dbadmin TO d10taskuser;

GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA dbadmin TO d10taskuser;

GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA dbadmin TO d10taskuser;
` Шаг 4: Права для pg_background
 `GRANT EXECUTE ON FUNCTION pg_background_launch(text, int4) TO dbo;

GRANT EXECUTE ON FUNCTION pg_background_result(int4) TO dbo;

GRANT EXECUTE ON FUNCTION pg_background_detach(int4) TO dbo;
` \q

Стратегия переключения
 Существует два подхода:
 Вариант A: Изменить порты (PostgreSQL 18 → 5432, PostgreSQL 16 → 5434) — рекомендуется
 Вариант B: Остановить PostgreSQL 16 и использовать только PostgreSQL 18

Шаг 1: Остановка обеих служб
 Остановка PostgreSQL 16:
 `sudo systemctl stop postgresql@16-main.service
` Остановка PostgreSQL 18:
 `sudo systemctl stop postgresql@18-main.service
` Проверка:
 `sudo systemctl status postgresql
` Шаг 2: Изменение порта PostgreSQL 16 на 5434
 `sudo vim /etc/postgresql/16/main/postgresql.conf
` Измените:
 port = 5434
 Шаг 3: Изменение порта PostgreSQL 18 на 5432
 `sudo vim /etc/postgresql/18/main/postgresql.conf
` Измените:
 port = 5432
 Шаг 4: Запуск служб в новой конфигурации
 Запуск PostgreSQL 18 на порту 5432:
 `sudo systemctl start postgresql@18-main.service

sudo systemctl status postgresql@18-main.service
` Запуск PostgreSQL 16 на порту 5434 (резервная копия):
 `sudo systemctl start postgresql@16-main.service

sudo systemctl status postgresql@16-main.service
` Шаг 5: Проверка портов
 PostgreSQL 18 на основном порту 5432:
 `sudo -u postgres psql -p 5432 -c "SELECT version();"
` PostgreSQL 16 на резервном порту 5434:
 `sudo -u postgres psql -p 5434 -c "SELECT version();"
`

Если тестирование прошло успешно и откат не требуется:
 Остановка PostgreSQL 16:
 `sudo systemctl stop postgresql@16-main.service
` Отключение автозапуска PostgreSQL 16:
 `sudo systemctl disable postgresql@16-main.service
` Изменение порта PostgreSQL 18 на 5432:
 `sudo vim /etc/postgresql/18/main/postgresql.conf
` port = 5432
 Перезапуск PostgreSQL 18
 `sudo systemctl restart postgresql@18-main.service
`

Шаг 1: Проверка версии
 `sudo -u postgres psql -p 5432 -c "SELECT version();"
` Ожидаемый результат: PostgreSQL 18.x
 Шаг 2: Проверка расширений
 `sudo -u postgres psql -p 5432 -d d10task -c "\dx"
` Должны быть установлены:
 plpgsql
 rum
 pgcrypto
 pg_buffercache
 btree_gin
 pg_stat_statements
 pg_trgm
 pg_qualstats
 vector
 pg_background
 Шаг 3: Проверка io_method
 `sudo -u postgres psql -p 5432 -c "SHOW io_method;"
` Ожидаемый результат: io_uring
 Шаг 4: Проверка данных
 `sudo -u postgres psql -p 5432 -d d10task
` `-- Проверка количества таблиц

SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'dbo';

-- Проверка количества записей в основных таблицах

SELECT

   schemaname,

   tablename,

   pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
` `FROM pg_tables

WHERE schemaname = 'dbo'

ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
` LIMIT 10;
 \q

Если обнаружены критические проблемы:
 Быстрый откат (если использовали Вариант A)
 1. Остановка PostgreSQL 18
 `sudo systemctl stop postgresql@18-main.service
` 2. Изменение порта PostgreSQL 16 обратно на 5432
 `sudo vim /etc/postgresql/16/main/postgresql.conf
` port = 5432
 3. Перезапуск PostgreSQL 16
 `sudo systemctl restart postgresql@16-main.service
` 4. Проверка
 `sudo -u postgres psql -p 5432 -c "SELECT version();"
` Полный откат
 Если PostgreSQL 16 был отключен:
 1. Остановка PostgreSQL 18
 `sudo systemctl stop postgresql@18-main.service

sudo systemctl disable postgresql@18-main.service
` 2. Включение PostgreSQL 16
 `sudo systemctl enable postgresql@16-main.service

sudo systemctl start postgresql@16-main.service
` 3. Проверка
 `sudo systemctl status postgresql@16-main.service
`

Важно: выполняйте только после подтверждения стабильной работы PostgreSQL 18 (рекомендуется выдержать 1–2 недели).
 Шаг 1: Проверка, что PostgreSQL 16 не используется
 Убедитесь, что приложения подключаются к PostgreSQL 18
 `sudo -u postgres psql -p 5432 -c "SELECT version();"
` Должен показать PostgreSQL 18.x
 Проверка активных подключений к PostgreSQL 16
 `sudo -u postgres psql -p 5434 -c "SELECT count(*) FROM pg_stat_activity WHERE datname IS NOT NULL;"
` Должно быть 0 (кроме служебных)
 Шаг 2: Остановка и отключение PostgreSQL 16
 `sudo systemctl stop postgresql@16-main.service

sudo systemctl disable postgresql@16-main.service
` Шаг 3: Удаление кластера PostgreSQL 16
 Важно: выбор метода удаления зависит от расположения data_directory.
 Сначала проверьте расположение данных:
 `cat /etc/postgresql/16/main/postgresql.conf | grep data_directory
` Если data_directory = /pgdata/16-main (отдельная директория):
 Можно использовать pg_dropcluster — это безопасно:
 Удаление кластера (удаляет данные и конфиги)
 `sudo pg_dropcluster 16 main
` Проверка:
 `pg_lsclusters
` Если data_directory = /pgdata (родительская директория для PG18):
 Важно: не используйте pg_dropcluster — он удалит /pgdata вместе с данными PostgreSQL 18.
 Удаляйте вручную:
 ⚠️ Убедитесь, что сервис PostgreSQL 18 использует другую директорию данных (не /pgdata), прежде чем удалять содержимое. Проверьте параметр data_directory в конфигурации PostgreSQL 18.
 1. Удалить конфиги PG16:
 `sudo rm -rf /etc/postgresql/16
` 2. Вручную удалить ТОЛЬКО файлы PG16 из /pgdata
 Сначала посмотрите что там:
 ls -la /pgdata/
 Удалите только файлы/папки PG16, НЕ трогая 18-main:
 `sudo rm -rf /pgdata/base

sudo rm -rf /pgdata/global

sudo rm -rf /pgdata/pg_*

sudo rm -f /pgdata/PG_VERSION

sudo rm -f /pgdata/postgresql.auto.conf

sudo rm -f /pgdata/postmaster.*
` Шаг 4: Удаление пакетов PostgreSQL 16
 Удаление пакетов:
 `sudo apt remove postgresql-16 postgresql-client-16 postgresql-server-dev-16
` Удаление неиспользуемых зависимостей:
 `sudo apt autoremove
` Проверка оставшихся пакетов версии 16:
 `dpkg -l | grep postgresql | grep 16
` Шаг 5: Удаление логов PostgreSQL 16
 `sudo rm -rf /var/log/postgresql/postgresql-16-main.log*
` Шаг 6: Очистка временных файлов миграции
 Удаление дампов (только после создания свежих бэкапов PG18!)
 `sudo rm -rf /opt/migration-backup/dumps/*.dump
` Удаление бэкапа данных PG16
 `sudo rm -rf /opt/migration-backup/pg16-data-backup
` Удаление бэкапов конфигураций (опционально, можно оставить для истории)
 `sudo rm -rf /opt/migration-backup/*.backup
` Шаг 7: Финальная проверка
 Проверка, что остался только PostgreSQL 18:
 `dpkg -l | grep postgresql
` Проверка работающих кластеров:
 `pg_lsclusters
` Проверка занятого места:
 df -h /var/lib/postgresql
 df -h /pgdata
