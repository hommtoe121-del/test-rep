# Настройка PostgreSQL 18 на Debian-подобных операционных системах

Источник: https://help.1forma.ru/maintenance_guide/installation/databases/postgresql/tech_req_1f_prepare_postgresql/

ℹ️ В данном разделе приведена инструкция для установки PostgreSQL 18+ для версии ОС Ubuntu 22.04 LTS

Шаг 1: Подготовка системы
 `sudo apt update

sudo apt install -y apt-transport-https \

               ca-certificates \

               curl \

               freetds-common \

               gnupg2
` Шаг 2: Добавление репозитория PostgreSQL
 `curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg

sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

sudo apt update
` Шаг 3: Установка PostgreSQL 18
 `sudo apt install -y postgresql-18 \

               postgresql-server-dev-18 \

               postgresql-client-18
` Шаг 4: Установка расширений
 `sudo apt install -y postgresql-18-rum \

               postgresql-18-pg-hint-plan \

               postgresql-18-pg-qualstats \

               postgresql-18-pgvector
` Расширения plpgsql, pgcrypto, pg_buffercache, btree_gin, pg_stat_statements, pg_trgm входят в стандартную поставку PostgreSQL и не требуют отдельной установки.
 Шаг 5: Установка pg_background
 `cd /tmp

wget --user USERNAME --ask-password https://download.1forma.ru/raw-1f-app/pg-extensions/pg_background.zip

unzip pg_background.zip

rm pg_background.zip

sudo apt install -y build-essential libkrb5-dev

cd pg_background-master/

make

sudo make install

cd ..

rm -rf pg_background-master
` Шаг 6: Остановка службы PostgreSQL
 `sudo systemctl stop postgresql.service
` Шаг 7 (опционально): Перенос каталога данных
 Если требуется переместить данные PostgreSQL в другой каталог (например, /pgdata/18-main):
 # Создание каталога
 `sudo mkdir -p /pgdata/18-main
` # Копирование данных
 `sudo cp -r -p /var/lib/postgresql/18/main/. /pgdata/18-main/
` # Назначение владельца
 `sudo chown -R postgres:postgres /pgdata
` ℹ️ После переноса данных обновите параметр data_directory в postgresql.conf

Обзор расширений
 Расширение Тип Назначение plpgsql Встроенное Процедурный язык для хранимых процедур pgcrypto Встроенное Криптографические функции pg_buffercache Встроенное Мониторинг буферного кеша btree_gin Встроенное GIN индексы для B-tree типов pg_stat_statements Встроенное Статистика выполнения запросов pg_trgm Встроенное Триграммы для нечеткого поиска rum Внешнее Улучшенный полнотекстовый поиск pg_hint_plan Внешнее Управление планировщиком запросов pg_qualstats Внешнее Статистика предикатов WHERE vector Внешнее Векторные операции для ML/AI pg_background Внешнее Выполнение команд в фоновых воркерах

Встроенные расширения
 plpgsql
 Процедурный язык PL/pgSQL для написания хранимых процедур и функций. Является основным языком для серверной бизнес-логики в PostgreSQL. Включен по умолчанию.
 pgcrypto
 Криптографические функции для шифрования данных. Используется для хеширования паролей, шифрования/дешифрования данных. Входит в contrib-модули.
 pg_buffercache
 Мониторинг содержимого буферного кеша. Используется для анализа использования памяти и оптимизации производительности. Входит в contrib-модули.
 btree_gin
 Классы операторов GIN для B-tree индексируемых типов данных. Используется для расширения возможностей GIN индексов. Входит в contrib-модули.
 pg_stat_statements
 Сбор статистики выполнения SQL-запросов. Используется для мониторинга и оптимизации производительности запросов. Входит в contrib-модули. Необходимо добавить в shared_preload_libraries.
 pg_trgm
 Определение схожести текста на основе триграмм. Используется для полнотекстового и нечёткого поиска строк. Входит в contrib-модули.
 Внешние расширения
 rum
 Улучшенный тип индекса для полнотекстового поиска. Более быстрый и функциональный полнотекстовый поиск по сравнению с GIN. Входит в пакет postgresql-18-rum.
 pg_hint_plan
 Управление планировщиком запросов через подсказки (hints). Используется для принудительного изменения плана выполнения запросов. Входит в пакет postgresql-18-pg-hint-plan. Необходимо добавить в shared_preload_libraries.
 pg_qualstats
 Сбор статистики по предикатам WHERE. Используется для анализа условий запросов и оптимизации индексов. Входит в пакет postgresql-18-pg-qualstats. Необходимо добавить в shared_preload_libraries.
 pgvector (vector)
 Поддержка векторных типов данных и операций. Используется для работы с векторными представлениями в ML/AI-задачах и поиска по сходству. Входит в пакет postgresql-18-pgvector.
 pg_background
 Расширение для выполнения SQL-команд в фоновых рабочих процессах. Используется для асинхронного выполнения длительных операций без блокировки основного подключения. Требует компиляции из исходников (GitHub). Применяется для длительных миграций, фоновых задач, параллельной обработки.

Настройка локализации системы
 Установка русской локали
 `sudo apt install -y language-pack-ru

sudo update-locale LANG=ru_RU.UTF-8
` Альтернативный способ:
 `sudo apt install locales

localedef -i ru_RU -c -f UTF-8 -A /usr/share/locale/locale.alias ru_RU.UTF-8
` Проверка локали
 `locale
` Ожидаемый результат: LANG=ru_RU.UTF-8
 ℹ️ Может потребоваться выход из сессии для применения изменений
 Настройка формата даты
 Создайте файл /etc/freetds/locales.conf:
 `sudo vim /etc/freetds/locales.conf
` Содержимое файла:
 [default]
 `date format = %b %e %Y %I:%M:%S.%z%p
` Проверка:
 `date
` Ожидаемый результат: Чт 30 янв 2025 15:50:30 MSK
 Конфигурационный файл postgresql.conf
 Откройте файл конфигурации:
 `sudo vim /etc/postgresql/18/main/postgresql.conf
` Базовые параметры
 # Путь к данным (если переносили)
 data_directory = '/pgdata/18-main'
 # Подключения
 max_connections = 100  # Для продуктивных установок рекомендуется увеличить до 500-1500 в зависимости от нагрузки (см. документ по оптимизации производительности БД PostgreSQL)
 listen_addresses = '*'
 # JIT
 jit = off
 # Метод ввода-вывода (новый параметр для PostgreSQL 18)
 io_method = 'io_uring'  # При ошибках запуска, связанных с io_uring, измените значение на 'sync'. Подробнее — в разделе "Устранение неполадок" этого документа.
 # Временная зона
 log_timezone = 'Europe/Moscow'
 timezone = 'Europe/Moscow'
 # Конфигурация полнотекстового поиска по умолчанию (русский язык)
 # Критично для приложения "Первая Форма": FTS-индексы строятся через to_tsvector('russian', ...),
 # и без этого параметра поиск websearch_to_tsquery без явного language работает по english словарю и не находит русские документы.
 default_text_search_config = 'pg_catalog.russian'
 Настройте параметры производительности в соответствии с характеристиками вашего сервера. Рекомендации:
 # Память
 shared_buffers = 8GB              # ~25% от RAM
 effective_cache_size = 24GB       # 50-75% от RAM
 work_mem = 32MB                   # (RAM * 0.25) / количество_конкурентных_операций
 maintenance_work_mem = 1GB        # 0.5-2 GB для VACUUM/CREATE INDEX
 # Производительность для SSD
 random_page_cost = 1.1
 effective_io_concurrency = 200
 # WAL
 min_wal_size = 1GB
 max_wal_size = 4GB
 checkpoint_completion_target = 0.9
 # Расширения
 shared_preload_libraries = 'pg_stat_statements,pg_hint_plan,pg_qualstats'
 ℹ️ Для точной настройки используйте утилиту PGTune
 Настройка доступа (pg_hba.conf)
 Откройте файл конфигурации доступа:
 `sudo vim /etc/postgresql/18/main/pg_hba.conf
` Добавьте правила доступа:
 # Для доступа из определенной подсети
 host    all             all             192.168.0.0/24  scram-sha-256
 # Для доступа из любой подсети (используйте с осторожностью)
 host    all             all             0.0.0.0/0       scram-sha-256

Запуск службы PostgreSQL
 `sudo systemctl start postgresql.service

sudo systemctl enable postgresql.service
` Проверка статуса
 `sudo systemctl status postgresql.service

sudo systemctl status postgresql@18-main.service
` Подключение к PostgreSQL
 `sudo -u postgres psql
` Активация расширений
 `-- Загрузка pg_hint_plan

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
` Создание пользователей
 `CREATE USER dbo WITH SUPERUSER CREATEDB CREATEROLE INHERIT LOGIN PASSWORD 'pass1';

CREATE USER d10taskuser WITH INHERIT LOGIN PASSWORD 'pass2';

CREATE USER migrationsdaemon WITH INHERIT LOGIN PASSWORD 'pass3';

CREATE USER rebus WITH INHERIT LOGIN PASSWORD 'pass4';

CREATE USER implementer WITH INHERIT LOGIN PASSWORD 'pass5';
` ℹ️ Обязательно замените пароли на сложные!
 Создание роли для SMART тестирования
 `CREATE ROLE role_D10TaskReader WITH INHERIT;

CREATE USER D10TaskReader WITH INHERIT LOGIN PASSWORD 'pass6';

GRANT role_D10TaskReader TO D10TaskReader;

ALTER USER D10TaskReader SET search_path = "dbo", "public";
` Создание баз данных
 `CREATE DATABASE d10task

   TEMPLATE template0

   OWNER dbo

   ENCODING 'utf8'

   LC_COLLATE 'ru_RU.utf8'

   LC_CTYPE 'ru_RU.utf8';
` `CREATE DATABASE d10task_file

   TEMPLATE template0

   OWNER dbo

   ENCODING 'utf8'

   LC_COLLATE 'ru_RU.utf8'

   LC_CTYPE 'ru_RU.utf8';
` Выход из консоли: \q
 Создание расширений в базе d10task (перед восстановлением дампа)
 Подключитесь к базе:
 `sudo -u postgres psql -d d10task
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
 Настройка файловой базы данных
 Подключитесь к базе d10task_file:
 `sudo -u postgres psql -d d10task_file
` Выполните скрипт создания схемы:
 `CREATE SCHEMA IF NOT EXISTS dbo AUTHORIZATION dbo;

CREATE TABLE dbo.UploadFiles (

   id          INT NOT NULL GENERATED BY DEFAULT AS IDENTITY,

   FileContent BYTEA,

   Ext         VARCHAR,

   UserID      INT,

   FileName    VARCHAR,

   Compressed  BOOL CONSTRAINT DF_UploadFiles_Compressed DEFAULT FALSE,

   CONSTRAINT PK_UploadFiles PRIMARY KEY (id)
` );
 `alter user d10taskuser set search_path = "dbo";

grant select, insert, update, delete, truncate on all tables in schema dbo to d10taskuser;

grant usage, select on all sequences in schema dbo to d10taskuser;

grant execute on all functions in schema dbo to d10taskuser;

alter default privileges in schema dbo grant select, insert, update, delete, truncate on tables to d10taskuser;

alter default privileges in schema dbo grant usage, select on sequences to d10taskuser;

alter default privileges in schema dbo grant execute on functions to d10taskuser;

alter default privileges in schema dbo grant usage on types to d10taskuser;

grant usage on schema dbo to d10taskuser;
` \q
 Восстановление базы данных d10task
 Загрузите дамп на сервер (например, в /opt/backups/):
 `wget --user USERNAME --ask-password https://download.1forma.ru/raw-1f-app/1f-db/d10task_no_ext.dump
` Выполните восстановление:
 `pg_restore -h localhost -p 5432 -U dbo -d d10task ./d10task_no_ext.dump
` Настройка схем и разрешений
 Подключитесь к базе:
 `sudo -u postgres psql -d d10task
` Выполните настройку:
 `alter role D10TaskUser set search_path = "dbo", "public";

alter role dbo set search_path = "dbo", "public";

alter user MigrationsDaemon set search_path = "dbo", "public";

alter user rebus set search_path = "rebus", "public";

grant usage on schema dbo to d10taskuser;

grant dbo, rebus to MigrationsDaemon;

grant all on all tables in schema dbo, sys to dbo, MigrationsDaemon;

grant all on all sequences in schema dbo, sys to dbo, MigrationsDaemon;

grant all on all functions in schema dbo, sys to dbo, MigrationsDaemon;

grant all on schema dbo, sys to dbo, MigrationsDaemon;

alter default privileges in schema dbo, sys, SignalR grant all on tables to dbo, MigrationsDaemon;

alter default privileges in schema dbo, sys, SignalR grant all on sequences to dbo, MigrationsDaemon;

alter default privileges in schema dbo, sys, SignalR grant all on functions to dbo, MigrationsDaemon;

alter default privileges in schema dbo, sys, SignalR grant all on types to dbo, MigrationsDaemon;
` Настройка для SMART тестирования
 `grant usage on schema dbo, sys, SignalR to role_D10TaskReader;

grant select on all tables in schema dbo, sys, SignalR to role_D10TaskReader;

grant usage, select on all sequences in schema dbo, sys, SignalR to role_D10TaskReader;

grant execute on all functions in schema dbo, sys, SignalR to role_D10TaskReader;

alter default privileges in schema dbo, sys, SignalR  grant select  on tables to role_D10TaskReader;

alter default privileges in schema dbo, sys, SignalR  grant execute on functions to role_D10TaskReader;

alter default privileges in schema dbo, sys, SignalR  grant usage, select on sequences to role_D10TaskReader;

grant all on all tables in schema custom to role_D10TaskReader;

grant all on all sequences in schema custom to role_D10TaskReader;

grant all on all functions in schema custom to role_D10TaskReader;

alter default privileges in schema custom grant all on tables to role_D10TaskReader;

alter default privileges in schema custom grant all on sequences to role_D10TaskReader;

alter default privileges in schema custom grant all on functions to role_D10TaskReader;

alter default privileges in schema custom grant all on types to role_D10TaskReader;

grant all on schema custom to role_D10TaskReader;

grant all on all tables in schema rebus to role_D10TaskReader;

grant all on all sequences in schema rebus to role_D10TaskReader;

grant all on all functions in schema rebus to role_D10TaskReader;

alter default privileges in schema rebus grant all on tables to role_D10TaskReader;

alter default privileges in schema rebus grant all on sequences to role_D10TaskReader;

alter default privileges in schema rebus grant all on functions to role_D10TaskReader;

alter default privileges in schema rebus grant all on types to role_D10TaskReader;

grant all on schema rebus to role_D10TaskReader;

grant usage on language plpgsql, sql to role_D10TaskReader;
` Права для новой админки (схема dbadmin)
 `-- Если есть схема dbadmin

GRANT USAGE ON SCHEMA dbadmin TO d10taskuser;

GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA dbadmin TO d10taskuser;

GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA dbadmin TO d10taskuser;

GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA dbadmin TO d10taskuser;
` Права для pg_background
 `GRANT EXECUTE ON FUNCTION pg_background_launch(text, int4) TO dbo;

GRANT EXECUTE ON FUNCTION pg_background_result(int4) TO dbo;

GRANT EXECUTE ON FUNCTION pg_background_detach(int4) TO dbo;
`

Проверка io_method
 `sudo -u postgres psql -c "SHOW io_method;"
` Ожидаемый результат: io_uring
 Проверка установленных расширений
 `sudo -u postgres psql -d d10task -c "\dx"
` Список установленных расширений
 Должны быть активированы:

-  plpgsql


-  rum


-  pgcrypto


-  pg_buffercache


-  btree_gin


-  pg_stat_statements


-  pg_trgm


-  pg_qualstats


-  pg_hint_plan


-  vector


-  pg_background

Проблема: Ошибка "Too many open files" при io_uring
 Симптомы:
 FATAL:  could not setup io_uring queue: Too many open files
 HINT:  Consider increasing "ulimit -n" to at least 2542.
 Причина: PostgreSQL не может создать io_uring очередь из-за лимита открытых файловых дескрипторов.
 Решение:
 # 1. Редактирование systemd unit файла
 `sudo vim /lib/systemd/system/postgresql@.service
` Добавить в секцию [Service]:
 [Service]
 ...
 # Limits
 LimitNOFILE=65535
 # 2. Перезагрузка конфигурации systemd
 `sudo systemctl daemon-reload
` # 3. Перезапуск PostgreSQL
 `sudo systemctl restart postgresql.service
` # 4. Проверка статуса
 `sudo systemctl status postgresql@18-main.service
` Проверка лимитов процесса:
 # Найти PID процесса postgres
 ps aux | grep postgresql
 # Проверить лимиты (замените 7086 на реальный PID)
 `cat /proc/7086/limits
` Ожидаемый результат в строке "Max open files" должен быть 65535.

Все скрипты, используемые в данной инструкции, доступны для скачивания:
 Расширения PostgreSQL:

- pg_background.zip - https://download.1forma.ru/raw-1f-app/pg-extensions/pg_background.zip
  SQL скрипты:

-  01-create-extensions.sql - Создание расширений PostgreSQL https://download.1forma.ru/raw-1f-app/pg-scripts/01-create-extensions.sql


-  02-create-users.sql - Создание пользователей https://download.1forma.ru/raw-1f-app/pg-scripts/02-create-users.sql


-  03-create-databases.sql - Создание баз данных https://download.1forma.ru/raw-1f-app/pg-scripts/03-create-databases.sql


-  04-setup-file-db.sql - Настройка файловой базы данных https://download.1forma.ru/raw-1f-app/pg-scripts/04-setup-file-db.sql


-  05-setup-permissions.sql - Настройка разрешений и схем https://download.1forma.ru/raw-1f-app/pg-scripts/05-setup-permissions.sql


-  06-setup-smart-testing.sql - Настройка для SMART тестирования https://download.1forma.ru/raw-1f-app/pg-scripts/06-setup-smart-testing.sql


-  07-setup-dbadmin.sql - Права для новой админки https://download.1forma.ru/raw-1f-app/pg-scripts/07-setup-dbadmin.sql


-  08-setup-pg-background.sql - Права для pg_background https://download.1forma.ru/raw-1f-app/pg-scripts/08-setup-pg-background.sql

  Все скрипты доступны по адресу: https://download.1forma.ru/raw-1f-app/pg-scripts/
