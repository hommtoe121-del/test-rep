# Отказоустойчивый PostgreSQL кластер Patroni

Источник: https://help.1forma.ru/maintenance_guide/installation/databases/postgresql/postgresql_patroni_cluster/

Рассмотрим настройку отказоустойчивого PostgreSQL кластера Patroni. В качестве балансировщика нагрузки будем использовать HAProxy и Keepalived.



PostgreSQL — это объектно-реляционная система управления базами данных (СУБД) с открытым исходным кодом. Она используется для хранения и управления данными и поддерживает стандарты SQL.

Patroni предоставляет шаблон для настройки кластера PostgreSQL с высокой доступностью. Это инструмент для управления кластерами, который можно использовать для автоматизации развертывания и обслуживания кластеров PostgreSQL. Patroni написан на Python и использует etcd, Consul или ZooKeeper для хранения конфигураций, обеспечивая высокую доступность. В данной настройке используется etcd. Patroni также может управлять конфигурациями для репликации баз данных, резервного копирования и восстановления. Настройки Patroni управляются с помощью файла YAML.

HAProxy — это бесплатное программное обеспечение с открытым исходным кодом, которое предоставляет балансировщик нагрузки и прокси-сервер с высокой доступностью для приложений, работающих на основе TCP и HTTP, распределяя запросы между несколькими серверами. HAProxy отслеживает изменения в мастер/слейв-узлах и подключается к соответствующему мастер-узлу по запросу клиентов.

Keepalived — это программный комплекс, обеспечивающий высокую доступность и балансировку нагрузки.

ETCD используется как распределенное хранилище конфигураций (DCS), где хранится состояние кластера PostgreSQL. При любом изменении состояния узлов PostgreSQL Patroni обновляет соответствующие данные в хранилище ключей-значений ETCD. ETCD использует эту информацию для выбора узла-лидера и поддержания работоспособности кластера. Для обеспечения высокой доступности рекомендуется развернуть три сервера ETCD отдельно от серверов базы данных.

Рассмотрим настройку кластера, состоящего из 3 нод etcd, 2 нод haproxy и 2 нод postgres:
 `10.0.0.1 etcd-1
10.0.0.2 etcd-2
10.0.0.3 etcd-3

10.0.0.11 patroni-1
10.0.0.12 patroni-2

10.0.0.21 haproxy-1
10.0.0.22 haproxy-2

10.0.0.20 keepalived virtual ip
` `flowchart TD
    APP["Приложение 1Ф"] -- "по virtual IP" --> VIP["Virtual IP 10.0.0.20 (Keepalived)"]
    VIP --> HA1["HAProxy-1 (MASTER)"]
    VIP --> HA2["HAProxy-2 (SLAVE, резерв)"]
    HA1 -- "проверка OPTIONS/master, порт 8008" --> PM["Patroni: лидер, порт 5432"]
    HA2 --> PM
    PM -- "потоковая репликация" --> PR["Patroni: реплика"]
    PM -- "состояние, выбор лидера" --> ETCD["ETCD кластер, 3 ноды"]
    PR --> ETCD`

На каждой ноде кластера выполните установку etcd:
 `sudo apt-get update
sudo apt-get install -y etcd-server etcd-client
`

Проверьте версию etcd сервера после установки:
 `etcd --version
`

Проверьте статус сервиса etcd. Если активен (running), остановите etcd сервера после установки:
 `sudo systemctl status etcd
sudo systemctl stop etcd
`

Отредактируйте файл `/etc/default/etcd` на каждой ноде etcd кластера:
 `sudo vim /etc/default/etcd
`

`[member]
ETCD_NAME="etcd-1"
ETCD_DATA_DIR="/var/lib/etcd/default"
ETCD_LISTEN_PEER_URLS="http://10.0.0.1:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.0.0.1:2379,http://127.0.0.1:2379"
ETCD_HEARTBEAT_INTERVAL="1000"
ETCD_ELECTION_TIMEOUT="5000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
ETCD_ENABLE_V2="true"
ETCD_AUTO_COMPACTION_MODE="periodic"
ETCD_AUTO_COMPACTION_RETENTION="1"
ETCD_QUOTA_BACKEND_BYTES="8589934592"

[cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.0.0.1:2380"
ETCD_INITIAL_CLUSTER="etcd-1=http://10.0.0.1:2380,etcd-2=http://10.0.0.2:2380,etcd-3=http://10.0.0.3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="http://10.0.0.1:2379"
`

`[member]
ETCD_NAME="etcd-2"
ETCD_DATA_DIR="/var/lib/etcd/default"
ETCD_LISTEN_PEER_URLS="http://10.0.0.2:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.0.0.2:2379,http://127.0.0.1:2379"
ETCD_HEARTBEAT_INTERVAL="1000"
ETCD_ELECTION_TIMEOUT="5000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
ETCD_ENABLE_V2="true"
ETCD_AUTO_COMPACTION_MODE="periodic"
ETCD_AUTO_COMPACTION_RETENTION="1"
ETCD_QUOTA_BACKEND_BYTES="8589934592"

[cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.0.0.2:2380"
ETCD_INITIAL_CLUSTER="etcd-1=http://10.0.0.1:2380,etcd-2=http://10.0.0.2:2380,etcd-3=http://10.0.0.3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="http://10.0.0.2:2379"
`

`[member]
ETCD_NAME="etcd-3"
ETCD_DATA_DIR="/var/lib/etcd/default"
ETCD_LISTEN_PEER_URLS="http://10.0.0.3:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.0.0.3:2379,http://127.0.0.1:2379"
ETCD_HEARTBEAT_INTERVAL="1000"
ETCD_ELECTION_TIMEOUT="5000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
ETCD_ENABLE_V2="true"
ETCD_AUTO_COMPACTION_MODE="periodic"
ETCD_AUTO_COMPACTION_RETENTION="1"
ETCD_QUOTA_BACKEND_BYTES="8589934592"

[cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.0.0.3:2380"
ETCD_INITIAL_CLUSTER="etcd-1=http://10.0.0.1:2380,etcd-2=http://10.0.0.2:2380,etcd-3=http://10.0.0.3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="http://10.0.0.3:2379"
`

Параметр Описание ETCD_NAME Имя узла. Значение должно быть уникальным ETCD_DATA_DIR Путь к каталогу данных ETCD_HEARTBEAT_INTERVAL Время (в миллисекундах), между рассылками лидером оповещений о том, что он всё ещё лидер. По умолчанию: "100" ETCD_ELECTION_TIMEOUT Время (в миллисекундах), которое проходит между последним принятым оповещением от лидера кластера, до попытки захватить роль лидера на ведомом узле. Рекомендуется задавать его в несколько раз большим, чем ETCD_HEARTBEAT_INTERVAL. По умолчанию: "1000". ETCD_LISTEN_PEER_URLS Список URL-адресов для прослушивания трафика между узлами. Этот параметр указывает etcd принимать входящие запросы от других узлов по заданным комбинациям scheme://IP. Схема может быть http или https. В качестве альтернативы можно использовать unix:// или unixs:// для Unix-сокетов. Если указано 0.0.0.0 как IP-адрес, etcd будет прослушивать данный порт на всех интерфейсах. Если указан конкретный IP-адрес и порт, etcd будет прослушивать данный порт и интерфейс. Можно указать несколько URL-адресов, чтобы настроить прослушивание на нескольких адресах и портах. Etcd будет отвечать на запросы с любого из указанных адресов и портов. ETCD_LISTEN_CLIENT_URLS Список URL-адресов для прослушивания клиентского трафика. Этот параметр указывает etcd принимать входящие запросы от клиентов по заданным комбинациям scheme://IP ETCD_INITIAL_ADVERTISE_PEER_URLS Начальный список URL-адресов. Эти адреса используются для обмена данными etcd по всему кластеру. По крайней мере один из них должен быть доступен для всех членов кластера. Эти URL-адреса могут содержать доменные имена ETCD_INITIAL_CLUSTER Список узлов кластера на момент запуска ETCD_INITIAL_CLUSTER_STATE Начальное состояние кластера («new» или «existing»). Установите значение new для всех членов, присутствующих во время начальной статической или DNS-загрузки. Если этот параметр установлен в existing, etcd попытается присоединиться к существующему кластеру. Если установлено неверное значение, etcd попытается запуститься, но завершится с ошибкой. ETCD_INITIAL_CLUSTER_TOKEN Токен кластера. Должен совпадать на всех узлах кластера ETCD_ADVERTISE_CLIENT_URLS Список равноправных URL-адресов, по которым его могут найти остальные узлы кластера. Можно использовать содержать доменные имена ETCD_AUTO_COMPACTION_MODE Режим автокомпакции истории ревизий: `periodic` — по времени, `revision` — по числу ревизий. Без компакции база etcd неограниченно растёт. ETCD_AUTO_COMPACTION_RETENTION Окно хранения. При `periodic` — в часах (`1` = удалять историю старше 1 часа). При `revision` — число ревизий. ETCD_QUOTA_BACKEND_BYTES Жёсткий лимит размера БД (по умолчанию 2 ГБ). Рекомендуется 8 ГБ. При превышении etcd уходит в alarm и блокирует запись.

Запустите службу etcd и проверьте её статус:
 `sudo systemctl enable etcd
sudo systemctl start etcd
sudo systemctl status etcd
`

Автокомпакция помечает старые ревизии как удалённые, но физически не освобождает место в файле БД. Для возврата свободного места нужна периодическая дефрагментация. Запускать defrag одновременно на всех узлах нельзя — узел блокируется на время операции, кворум может развалиться. Разносим запуск по узлам со смещением 5 минут.
 Создайте лог-файл и задайте права:
 `sudo touch /var/log/etcd-defrag.log
sudo chown etcd:etcd /var/log/etcd-defrag.log
` Добавьте задание в cron (`sudo crontab -e -u root`) — на каждом узле своё время:

`0 3 * * * ETCDCTL_API=3 etcdctl --endpoints=http://127.0.0.1:2379 defrag >> /var/log/etcd-defrag.log 2>&1
`

`5 3 * * * ETCDCTL_API=3 etcdctl --endpoints=http://127.0.0.1:2379 defrag >> /var/log/etcd-defrag.log 2>&1
`

`10 3 * * * ETCDCTL_API=3 etcdctl --endpoints=http://127.0.0.1:2379 defrag >> /var/log/etcd-defrag.log 2>&1
`

Если автокомпакция была подключена постфактум и БД уже большая, выполните на каждом узле последовательно (с интервалом, не параллельно):
 `# Узнать текущую ревизию
ETCDCTL_API=3 etcdctl --endpoints=http://127.0.0.1:2379 endpoint status --write-out=table

# Принудительная компакция до текущей ревизии
REV=$(ETCDCTL_API=3 etcdctl --endpoints=http://127.0.0.1:2379 endpoint status --write-out=json | jq '.[0].Status.header.revision')
ETCDCTL_API=3 etcdctl --endpoints=http://127.0.0.1:2379 compact $REV

# Дефрагментация
ETCDCTL_API=3 etcdctl --endpoints=http://127.0.0.1:2379 defrag

# Снять alarm, если был выставлен по quota
ETCDCTL_API=3 etcdctl --endpoints=http://127.0.0.1:2379 alarm disarm
`

Если БД настолько большая, что etcd падает по таймауту при старте:

- Остановите сервис: `sudo systemctl stop etcd`

- Сделайте бэкап данных: `sudo cp -r /var/lib/etcd/default /var/lib/etcd/default.bak-$(date +%F)`

- Очистите каталог данных: `sudo rm -rf /var/lib/etcd/default/*`

- На первом узле выставьте `ETCD_INITIAL_CLUSTER_STATE="new"` и запустите etcd, дайте инициализировать кластер.

- После запуска кластера перезапустите узлы Patroni: `sudo systemctl restart patroni` — Patroni перерегистрирует кластер в чистом etcd.

`etcdctl --write-out=table endpoint status --cluster
etcdctl --write-out=table endpoint health --cluster
`

Подключите репозитории PostgreSQL:
 `curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc|sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg

sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
`

`sudo apt-get update
sudo apt install -y postgresql-18 \
                postgresql-server-dev-18 \
                postgresql-client-18
` `sudo apt install -y postgresql-18-rum \
                postgresql-18-pg-hint-plan \
                postgresql-18-pg-qualstats \
                postgresql-18-pgvector
`

Остановите и отключите службу PostgreSQL:
 `sudo systemctl stop postgresql.service
sudo systemctl disable postgresql.service
`

Для использования баз данных Первая Форма в PostgreSQL должны быть настроены следующие расширения:

- plpgsql

- rum

- pgcrypto

- pg_buffercache

- btree_gin

- pg_stat_statements

- pg_trgm

- pg_qualstats

- pg_hint_plan

- vector

- pg_background

Установите необходимые для работы расширений пакеты:
 `sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl freetds-common freetds-dev g++ gcc git gnupg gnupg2 gpg libcurl4-openssl-dev libsybdb5 lsb-release make software-properties-common wget
`

Скачайте исходники модуля pg_background, скомпилируйте и установите:
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
`

Установите локаль ru_RU.utf8:
 `sudo apt-get update
sudo apt-get install -y language-pack-ru
sudo update-locale LANG=ru_RU.UTF-8
`

Проверьте командой locale. Должны получить: LANG=ru_RU.UTF-8

Настройте формат даты и времени. Создайте файл `/etc/freetds/locales.conf` со следующим наполнением:
 `sudo vim /etc/freetds/locales.conf
` `[default]
 date format = %b %e %Y %I:%M:%S.%z%p
`

Проверьте формат, выполнив команду date. Должны получить дату в следующем формате: Вт 09 мая 2023 22:16:05 UTC

Установите необходимые пакеты:
 `sudo apt-get update
sudo apt-get install -y python3 python3-etcd python3-etcd3
sudo apt-get install -y patroni
`

Создайте каталог для данных postgres:
 `sudo mkdir /pgdata
sudo chown -R postgres:postgres /pgdata
sudo chmod -R 700 /pgdata
`

Настройте конфигурацию кластера Patroni. Создайте и внесите настройки в файл `/etc/patroni/config.yml`:
 `sudo vim /etc/patroni/config.yml
`

`scope: patroni-cluster
name: patroni-1
namespace: /service

restapi:
  listen: 10.0.0.11:8008
  connect_address: 10.0.0.11:8008

etcd3:
  hosts: 10.0.0.1:2379,10.0.0.2:2379,10.0.0.3:2379

bootstrap:
  method: initdb
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
  initdb:
    - data-checksums
  postgresql:
    use_pg_rewind: true
    use_slots: true
    parameters:
      superuser_reserved_connections: 5
      max_connections: 100
      shared_buffers: 512MB
      effective_cache_size: 4GB
      work_mem: 128MB
      maintenance_work_mem: 256MB
      random_page_cost: 4
      effective_io_concurrency: 2
      huge_pages: try
      checkpoint_timeout: 15min
      wal_level: replica
      wal_compression: on
      max_wal_senders: 10
      synchronous_commit: on
      max_replication_slots: 10
      min_wal_size: 1GB
      max_wal_size: 4GB
      checkpoint_completion_target: 0.9
      wal_buffers: 16MB
      wal_keep_size: 2GB
      default_statistics_target: 1000
      log_timezone: 'Europe/Moscow'
      timezone: 'Europe/Moscow'
      lc_messages: 'en_US.UTF-8'
      lc_monetary: 'en_US.UTF-8'
      lc_numeric: 'en_US.UTF-8'
      lc_time: 'en_US.UTF-8'
      # Критично для русского полнотекстового поиска: FTS-индексы в БД "Первой Формы"
      # строятся через to_tsvector('russian', ...). Без этого параметра websearch_to_tsquery
      # без явного language идёт по english словарю и не находит русские документы.
      default_text_search_config: 'pg_catalog.russian'
      io_method: 'io_uring'
      shared_preload_libraries: 'pg_stat_statements,pg_hint_plan,pg_qualstats'
      jit: off

  pg_hba:
    - local all postgres peer
    - host all all 0.0.0.0/0 md5
    - host replication replicator 127.0.0.1/32 md5
    - host replication replicator 10.0.0.11/32 md5
    - host replication replicator 10.0.0.12/32 md5

postgresql:
  listen: 10.0.0.11,127.0.0.1:5432
  connect_address: 10.0.0.11:5432
  data_dir: /pgdata/18-main
  config_dir: /pgdata/18-main
  bin_dir: /usr/lib/postgresql/18/bin
  pgpass: /pgdata/.pgpass_patroni
  authentication:
    superuser:
      username: postgres
      password: postgres-password
    replication:
      username: replicator
      password: replicator-password
  remove_data_directory_on_rewind_failure: false
  remove_data_directory_on_diverged_timelines: false
  use_unix_socket: true
  parameters:
    unix_socket_directories: /var/run/postgresql

watchdog:
  mode: off
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
`

`scope: patroni-cluster
name: patroni-2
namespace: /service

restapi:
  listen: 10.0.0.12:8008
  connect_address: 10.0.0.12:8008

etcd3:
  hosts: 10.0.0.1:2379,10.0.0.2:2379,10.0.0.3:2379

bootstrap:
  method: initdb
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
  initdb:
    - data-checksums
  postgresql:
    use_pg_rewind: true
    use_slots: true
    parameters:
      superuser_reserved_connections: 5
      max_connections: 100
      shared_buffers: 512MB
      effective_cache_size: 4GB
      work_mem: 128MB
      maintenance_work_mem: 256MB
      random_page_cost: 4
      effective_io_concurrency: 2
      huge_pages: try
      checkpoint_timeout: 15min
      wal_level: replica
      wal_compression: on
      max_wal_senders: 10
      synchronous_commit: on
      max_replication_slots: 10
      min_wal_size: 1GB
      max_wal_size: 4GB
      checkpoint_completion_target: 0.9
      wal_buffers: 16MB
      wal_keep_size: 2GB
      default_statistics_target: 1000
      log_timezone: 'Europe/Moscow'
      timezone: 'Europe/Moscow'
      lc_messages: 'en_US.UTF-8'
      lc_monetary: 'en_US.UTF-8'
      lc_numeric: 'en_US.UTF-8'
      lc_time: 'en_US.UTF-8'
      # Критично для русского полнотекстового поиска: FTS-индексы в БД "Первой Формы"
      # строятся через to_tsvector('russian', ...). Без этого параметра websearch_to_tsquery
      # без явного language идёт по english словарю и не находит русские документы.
      default_text_search_config: 'pg_catalog.russian'
      io_method: 'io_uring'
      shared_preload_libraries: 'pg_stat_statements,pg_hint_plan,pg_qualstats'
      jit: off

  pg_hba:
    - local all postgres peer
    - host all all 0.0.0.0/0 md5
    - host replication replicator 127.0.0.1/32 md5
    - host replication replicator 10.0.0.11/32 md5
    - host replication replicator 10.0.0.12/32 md5

postgresql:
  listen: 10.0.0.12,127.0.0.1:5432
  connect_address: 10.0.0.12:5432
  data_dir: /pgdata/18-main
  config_dir: /pgdata/18-main
  bin_dir: /usr/lib/postgresql/18/bin
  pgpass: /pgdata/.pgpass_patroni
  authentication:
    superuser:
      username: postgres
      password: postgres-password
    replication:
      username: replicator
      password: replicator-password
  remove_data_directory_on_rewind_failure: false
  remove_data_directory_on_diverged_timelines: false
  use_unix_socket: true
  parameters:
    unix_socket_directories: /var/run/postgresql

watchdog:
  mode: off
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
`

Параметр Описание scope Название кластера namespace Путь в хранилище конфигурации, где Patroni будет хранить информацию о кластере. Значение по умолчанию: «/service» name Имя хоста. Должно быть уникальным для кластера restapi listen IP-адрес (или имя хоста) и порт, который Patroni будет прослушивать для REST API connect_address IP-адрес (или имя хоста) и порт для доступа к REST API Patroni. Все члены кластера должны иметь возможность подключаться к этому адресу bootstrap method Специальный скрипт, который будет использоваться для начальной загрузки этого кластера initdb (необязательно) Список параметров, которые будут переданы в initdb dcs Глобальная динамическая конфигурация кластера pg_hba Список строк, которые Patroni будет использовать для создания pg_hba.conf postgresql.use_pg_rewind Использовать pg_rewind. Обратите внимание, что либо кластер должен быть инициализирован с использованием контрольных сумм страниц данных (опция --data-checksums для initdb) и/или для параметра wal_log_hints должно быть установлено значение on, либо pg_rewind не будет работать postgresql.use_slots Использовать replication slots postgresql.parameters Список настроек конфигурации для Postgres. Мы рекомендуем воспользоваться "калькулятором" конфигурации Postgres для тонкой настройки конфигурации для вашего сервера (vCPU, RAM, max connections и т.д.) etcd или etcd3 Настройка подключения к etcd кластеру. Для версии кластера etcd v3 используйте etcd3 postgresql listen IP-адрес + порт, который слушает Postgres; должен быть доступен с других узлов кластера, если вы используете потоковую репликацию. Допускается использование нескольких адресов, разделенных запятыми, при условии, что компонент порта добавляется после последнего через двоеточие, т. е. прослушивание: 127.0.0.1, 127.0.0.2:5432. Patroni будет использовать первый адрес из этого списка для установки локального подключения к узлу PostgreSQL. connect_address IP-адрес + порт, через который Postgres доступен из других нод и приложений. authentication.superuser Суперпользователь, задается во время инициализации (initdb) и позже используется Patroni для подключения к postgres. authentication.replication Пользователь репликации; будет создан во время инициализации. Реплики будут использовать этого пользователя для доступа к источнику репликации посредством потоковой репликации. data_dir Расположение каталога данных Postgres, существующего или инициализируемого Patroni config_dir Расположение каталога конфигурации Postgres по умолчанию — каталог данных. Должен быть доступен для записи Patroni. pgpass Путь к файлу паролей .pgpass. Patroni создает этот файл перед выполнением pg_basebackup, сценария post_init и при некоторых других обстоятельствах. Местоположение должно быть доступно для записи Patroni. remove_data_directory_on_rewind_failure Если этот параметр включен, Patroni удалит каталог данных PostgreSQL и заново создаст реплику. В противном случае он попытается следовать за новым лидером. Значение по умолчанию — false. remove_data_directory_on_diverged_timelines Patroni удалит каталог данных PostgreSQL и заново создаст реплику, если заметит, что сроки расходятся и бывший основный сервер не может начать потоковую передачу с нового основного сервера. Эта опция полезна, когда pg_rewind невозможно использовать. При выполнении проверки расхождения сроков в PostgreSQL v10 и более ранних версиях Patroni попытается подключиться с учетными данными репликации к базе данных «postgres». Следовательно, такой доступ должен быть разрешен в файле pg_hba.conf. Значение по умолчанию — false. use_unix_socket Указывает, что Patroni предпочтет использовать сокеты unix для подключения к кластеру. Значение по умолчанию — false. Если определен unix_socket_directories, Patroni будет использовать первое подходящее значение для подключения к кластеру. Если подключение не удастся, то Patroni вернется к использованию tcp. Если unix_socket_directories не указан в postgresql.parameters, Patroni будет использовать значение по умолчанию, и опустит хост в параметрах соединения. parameters Список настроек конфигурации для Postgres

Включите и запустите службу patroni, проверьте статус:
 `sudo systemctl enable patroni
sudo systemctl start patroni
sudo systemctl status patroni
`

`patronictl -c /etc/patroni/config.yml list
`

Установите HAProxy:
 `sudo apt-get update
sudo apt-get install -y haproxy
`

Отредактируйте конфигурацию `/etc/haproxy/haproxy.cfg`:
 `sudo vim /etc/haproxy/haproxy.cfg
` `global
    maxconn 100
    log 127.0.0.1 local2

defaults
    log global
    mode tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

listen production
    bind *:5432
    option httpchk OPTIONS/master
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server patroni-01 10.0.0.11:5432 maxconn 100 check port 8008
    server patroni-02 10.0.0.12:5432 maxconn 100 check port 8008

listen standby
    bind *:5433
    option httpchk OPTIONS/replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server patroni-01 10.0.0.11:5432 maxconn 100 check port 8008
    server patroni-02 10.0.0.12:5432 maxconn 100 check port 8008
`

Проверьте конфигурацию и перезапустите службу HAProxy:
 `sudo haproxy -f /etc/haproxy/haproxy.cfg -c
sudo systemctl restart haproxy
sudo systemctl status haproxy
sudo systemctl enable haproxy
`

Установите Keepalived:
 `sudo apt-get update
sudo apt-get install -y keepalived
`

Выделите/определите виртуальный ip, который будет использоваться keepalived. В нашем примере это 192.168.162.10 и интерфейс, на котором он будет настроен - ens160:
 `ip a
inet 10.0.0.21/24 metric 100 brd 10.0.0.255 scope global dynamic ens160
`

Создайте конфигурацию `/etc/keepalived/keepalived.conf`:
 `sudo vim /etc/keepalived/keepalived.conf
`

`vrrp_instance pg-keepalived {
    state MASTER
    interface ens160
    virtual_router_id 1
    priority 11
    virtual_ipaddress {
        10.0.0.20/29 dev ens160 label ens160:1
    }
}
`

`vrrp_instance pg-keepalived {
    state SLAVE
    interface ens160
    virtual_router_id 1
    priority 10
    virtual_ipaddress {
        10.0.0.20/29 dev ens160 label ens160:1
    }
}
`

Параметр Описание vrrp_instance Имя кластера state Роль сервера, master или slave interface Имя сетевого интерфейса virtual_router_id Идентификатор «виртуального роутера», должен быть одинаковым на всех серверах-балансировщиках priority Приоритет балансировщиков, чем выше цифра тем больше приоритет, самый высокий следует выставлять для master сервера virtual_ipaddress Виртуальный IP

Включите и запустите службу keepalived, проверьте статус:
 `sudo systemctl enable keepalived
sudo systemctl start keepalived
sudo systemctl status keepalived
`

На master ноде проверьте, что появился новый (виртуальный) IP адрес в сетевом интерфейсе:
 `ip a
inet 10.0.0.20/29 scope global ens160:1
` Данный IP используйте для подключения к postgresql.

Авторизуйтесь в PSQL, подключившись пользователем Postgres, которого мы создали ранее, используйте ваш PG клиент, мы будем использовать PSQL. В качестве точки подключения используйте виртуальный ip или ip мастер ноды postgres:
 `psql -h 10.0.0.20 -p 5432 -U postgres -d postgres
`

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
`

`create user dbo with superuser createdb createrole inherit login password 'password';
create user d10taskuser with inherit login password 'password';
create user migrationsdaemon with inherit login password 'password';
create user rebus with inherit login password 'password';
create user implementer with inherit login password 'password';

CREATE ROLE role_D10TaskReader WITH INHERIT;
CREATE USER D10TaskReader WITH INHERIT LOGIN PASSWORD 'pass6';
GRANT role_D10TaskReader TO D10TaskReader;
ALTER USER D10TaskReader SET search_path = "dbo", "public";
`

Создайте базы данных d10task и d10task_taskfiles:
 `create database d10task
template template0
owner dbo
encoding 'utf8'
lc_collate 'ru_RU.utf8'
lc_ctype 'ru_RU.utf8';

create database d10task_taskfiles
template template0
owner dbo
encoding 'utf8'
lc_collate 'ru_RU.utf8'
lc_ctype 'ru_RU.utf8';
`

Создайте схемы в базе taskfilesdb:
 `\c d10task_taskfiles

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
`

Создайте (получите) дамп базы данных "Первой Формы" (d10task). Выполните рестор в созданную базу d10task:
 `pg_restore -h 10.0.0.20 -p 5432 -U postgres -d d10task ./d10task-template.dump
`

Настройте схемы для пользователей в новой восстановленной базе:
 `psql -h 10.0.0.20 -p 5432 -U postgres -d d10task
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

alter role D10TaskUser set search_path = "dbo", "public";
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

grant usage on schema dbo, sys, SignalR to role_D10TaskReader;
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

GRANT USAGE ON SCHEMA dbadmin TO d10taskuser;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA dbadmin TO d10taskuser;
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA dbadmin TO d10taskuser;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA dbadmin TO d10taskuser;

GRANT EXECUTE ON FUNCTION pg_background_launch(text, int4) TO dbo;
GRANT EXECUTE ON FUNCTION pg_background_result(int4) TO dbo;
GRANT EXECUTE ON FUNCTION pg_background_detach(int4) TO dbo;
`

Симптомы:
 `FATAL:  could not setup io_uring queue: Too many open files
HINT:  Consider increasing "ulimit -n" to at least 2542.
` Решение: На двух сервера PG необходимо внести изменения в ограничения файлов `# 1. Редактирование systemd unit файла
sudo vim /lib/systemd/system/patroni.service
` Добавить в секцию `[Service]`:
 `[Service]
...
# Limits
LimitNOFILE=65535
` `# 2. Перезагрузка конфигурации systemd
sudo systemctl daemon-reload

# 3. Перезапуск PostgreSQL
sudo systemctl restart patroni.service

# 4. Проверка статуса
sudo systemctl status patroni
`

Симптомы:

- `/var/lib/etcd/default/member/snap/db` весит несколько ГБ

- `systemctl restart etcd` отваливается по таймауту

- В логах `mvcc: database space exceeded` или `etcdserver: no space`
  Причина: не настроена автокомпакция ревизий и/или дефрагментация.
 Решение: см. шаги в разделе «Установка и настройка ETCD кластера» → «Настройка дефрагментации ETCD» (параметры `ETCD_AUTO_COMPACTION_MODE`, `ETCD_AUTO_COMPACTION_RETENTION` и cron-defrag), а также аварийный сценарий восстановления в том же разделе.

На этом настройка отказоустойчивого кластера завершена. Используйте virtual ip в настройках подключения приложения "Первой Формы".
 Допускается совмещение ролей на серверах. Например, в случае использования трёх и более нод Postgres, вы можете разместить etcd на этих же нодах.
