# Установка и настройка PostgreSQL 18 для приложения "Первая Форма"

Источник: https://help.1forma.ru/maintenance_guide/installation/databases/postgresql/tech_req_1f_prepare_postgrepro/

ℹ️ В данном разделе приведена инструкция для установки Postgres Pro 18+ для версии OS Ubuntu 22.04 LTS

1. Обновляем список пакетов и устанавливаем необходимые зависимости:
 `sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl freetds-common freetds-dev \
  g++ gcc git gnupg gnupg2 gpg libcurl4-openssl-dev libsybdb5 lsb-release make softwareproperties-common vim wget
` 2. Добавляем репозиторий Postgres Pro
 `curl -o pgpro-repo-add.sh https://repo.postgrespro.ru/pgpro-18/keys/pgpro-repo-add.sh
sudo sh pgpro-repo-add.sh
sudo apt update
` 3. Устанавливаем пакеты Postgres:
 `sudo apt install -y postgrespro-std-18
sudo apt install -y postgrespro-std-18-dev postgrespro-std-18-client
`

В файле конфигурации '/var/lib/pgpro/std-18/data/postgresql.conf' редактируем параметры:
 `max_connections = 1500
listen_addresses = '*'
log_timezone = 'Europe/Moscow'
timezone = 'Europe/Moscow'
# Критично для русского полнотекстового поиска: FTS-индексы в БД "Первой Формы"
# строятся через to_tsvector('russian', ...). Без этого параметра websearch_to_tsquery
# без явного language идёт по english словарю и не находит русские документы.
default_text_search_config = 'pg_catalog.russian'
` Настраиваем доступ к серверу PostgreSQL. Для этого редактируем файл '/var/lib/pgpro/std-18/data/pg_hba.conf'. Для разрешения соединений только из указанной подсети:
 `host all all 192.168.4.0/24 md5
` Для разрешения соединения из любой подсети:
 `host all all 0.0.0.0/0 md5
`

Устанавливаем локаль ru_RU.utf8:
 `sudo apt install -y language-pack-ru
sudo update-locale LANG=ru_RU.UTF-8
` Проверяем командой locale. Должны получить следующее:
 `LANG=ru_RU.UTF-8
LANGUAGE=
LC_CTYPE="ru_RU.UTF-8"
LC_NUMERIC="ru_RU.UTF-8"
LC_TIME="ru_RU.UTF-8"
LC_COLLATE="ru_RU.UTF-8"
LC_MONETARY="ru_RU.UTF-8
LC_MESSAGES="ru_RU.UTF-8"
LC_PAPER="ru_RU.UTF-8"
LC_NAME="ru_RU.UTF-8"
LC_ADDRESS="ru_RU.UTF-8"
LC_TELEPHONE="ru_RU.UTF-8"
LC_MEASUREMENT="ru_RU.UTF-8"
LC_IDENTIFICATION="ru_RU.UTF-8"
LC_ALL=
` Настраиваем формат даты и времени. Создаем файл '/etc/freetds/locales.conf' со следующим наполнением:
 `sudo vim /etc/freetds/locales.conf
` Содержимое:
 `[default]
date format = %b %e %Y %I:%M:%S.%z%p
` Проверяем командой date. Должны получить дату в следующем формате:
 `Вт 09 мая 2023 22:16:05 UTC
`

1. Для перезапуска службы выполним следующую команду:
 `sudo systemctl restart postgrespro-std-18.service
` 2. Проверим статус:
 `sudo systemctl status postgrespro-std-18.service
`

1. Для использования баз данных "Первой Формы" в PostgreSQL должны быть настроены следующие расширения:

-  http


-  pg_buffercache


-  pg_hint_plan


-  pgcrypto


-  postgres_fdw


-  rum


-  tds_fdw

  1. Скачиваем и устанавливаем модуль rum:
 `git clone https://github.com/postgrespro/rum
cd rum
make USE_PGXS=1
sudo make USE_PGXS=1 install
cd
` В случае ошибки pg_config not found выполните:
 `sudo ln -s /opt/pgpro/std-18/bin/pg_config /usr/bin/pg_config
` 2. Скачиваем с GitHub исходники модуля tds_fdw, компилируем и устанавливаем:
 `git clone https://github.com/tds-fdw/tds_fdw.git
cd tds_fdw
make USE_PGXS=1
sudo make USE_PGXS=1 install
cd
` 3. Скачиваем с GitHub исходники модуля pg_hint_plan, компилируем и устанавливаем:
 `git clone https://github.com/ossc-db/pg_hint_plan.git
cd pg_hint_plan
git checkout PG18
make
sudo make install
cd
` 4. Скачиваем с GitHub исходники модуля http, компилируем и устанавливаем:
 `git clone https://github.com/pramsey/pgsql-http.git
cd pgsql-http
make
sudo make install
cd
` 5. Заходим в терминальный клиент Postgres и подключаемся к базе d10task:
 `sudo -u postgres psql -d d10task
` 6. Активируем расширения:
 `LOAD 'pg_hint_plan';

CREATE EXTENSION IF NOT EXISTS tds_fdw schema public;

CREATE EXTENSION IF NOT EXISTS http schema public;

CREATE EXTENSION IF NOT EXISTS rum schema public;

CREATE EXTENSION IF NOT EXISTS postgres_fdw schema public;

CREATE EXTENSION IF NOT EXISTS pgcrypto schema public;

CREATE EXTENSION IF NOT EXISTS pg_buffercache schema public;

CREATE EXTENSION IF NOT EXISTS btree_gin;
`

1. Создаем пользователей:
 `create user dbo with superuser createdb createrole inherit login password 'PASSWORD';

create user d10taskuser with inherit login password 'PASSWORD';

create user migrationsdaemon with inherit login password 'PASSWORD';

create user rebus with inherit login password 'PASSWORD';

create user implementer with inherit login password 'PASSWORD';
` Вместо PASSWORD пропишите пароли пользователей PostgreSQL.
 2. Выйдите из консоли psql:
 \q
 3. Создаем базы данных d10task и taskfilesdb:
 `create database d10task
  template template0
  owner dbo
  encoding 'utf8'
  lc_collate 'ru_RU.utf8'
  lc_ctype 'ru_RU.utf8';
` `create database taskfilesdb
  template template0
  owner dbo
  encoding 'utf8'
  lc_collate 'ru_RU.utf8'
  lc_ctype 'ru_RU.utf8';
` `GRANT ALL PRIVILEGES ON DATABASE d10task TO d10taskuser;

GRANT ALL PRIVILEGES ON DATABASE taskfilesdb TO d10taskuser;
` Выйти из консоли можно командой \q.
 4. Создаем схему в базе taskfilesdb:
 `sudo -u postgres psql -d taskfilesdb
` `create extension if not exists tds_fdw schema public;

create schema if not exists dbo authorization dbo;

create table dbo.UploadFiles
(
  id int not null generated by default as identity,
  FileContent bytea,
  Ext varchar,
  UserID int,
  FileName varchar,
  Compressed bool constraint DF_UploadFiles_Compressed default false,
  constraint PK_UploadFiles primary key (id)
);
` \q
 5. Копируем на сервер файл с расширением dump, содержащий структуру и контент базы данных, в определенную папку, например: /src/d10task_hd.dump. Данный файл вам может предоставить техническая поддержка.
 Выполняем рестор 'pg_restore -d DB_NAME DB_DUMP' в созданную базу:
 `sudo su - postgres
pg_restore -d d10task /var/lib/pgpro/std-18/data/backup/d10task-template.dump
exit
` 6. Настраиваем схемы для пользователей в новой базе:
 `sudo -u postgres psql -d d10task
` `alter role D10TaskUser set search_path = "dbo", "public";

alter role dbo set search_path = "dbo", "public";

alter user MigrationsDaemon set search_path = "dbo", "public";

alter user rebus set search_path = "rebus", "public";

grant dbo, rebus to MigrationsDaemon;

grant all on all tables in schema dbo, sys to dbo, MigrationsDaemon;

grant all on all sequences in schema dbo, sys to dbo, MigrationsDaemon;

grant all on all functions in schema dbo, sys to dbo, MigrationsDaemon;

grant all on schema dbo, sys to dbo, MigrationsDaemon;

alter default privileges in schema dbo, sys, SignalR grant all on tables to dbo, MigrationsDaemon;

alter default privileges in schema dbo, sys, SignalR grant all on sequences to dbo, MigrationsDaemon;

alter default privileges in schema dbo, sys, SignalR grant all on functions to dbo, MigrationsDaemon;

alter default privileges in schema dbo, sys, SignalR grant all on types to dbo, MigrationsDaemon;
` На этом настройка СУБД PostgreSQL PRO 18 завершена.
