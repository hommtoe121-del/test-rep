# Функционирование сервера БД PostgreSQL

Источник: https://help.1forma.ru/maintenance_guide/installation/databases/postgresql/1f_performance_db_server_postgresql/

1. Проверка установки:
 `$ dpkg -l | grep postgres
` 2. Проверка службы:
 `$ systemctl list-units | grep post

postgresql.service                                                                       loaded active exited    PostgreSQL RDBMS

postgresql@10-main.service                                                               loaded active running   PostgreSQL Cluster 10-main

system-postgresql.slice                                                                  loaded active active    system-postgresql.slice
` 3. Проверка статуса:
 `$ systemctl status postgresql.service

● postgresql.service — PostgreSQL RDBMS

  Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)

  Active: active (exited) since Пт 2019-01-11 11:05:35 MSK; 4h 2min ago

 Process: 1478 ExecStart=/bin/true (code=exited, status=0/SUCCESS)

Main PID: 1478 (code=exited, status=0/SUCCESS)

   Tasks: 0

  Memory: 0B

     CPU: 0

  CGroup: /system.slice/postgresql.service
` янв 11 11:05:35 u1604 systemd[1]: Starting PostgreSQL RDBMS...
 янв 11 11:05:35 u1604 systemd[1]: Started PostgreSQL RDBMS.
 `$ ps aux | grep postgres

postgres   806  0.0  0.1 320972 28476 ?        S    15:32   0:00 /usr/lib/postgresql/10/bin/postgres -D /var/lib/postgresql/10/main -c config_file=/etc/postgresql/10/main/postgresql.conf

postgres   863  0.0  0.0 169768  3572 ?        Ss   15:32   0:00 postgres: 10/main: logger process

postgres  1437  0.0  0.0 320972  4220 ?        Ss   15:32   0:00 postgres: 10/main: checkpointer process

postgres  1438  0.0  0.0 320972  6428 ?        Ss   15:32   0:00 postgres: 10/main: writer process

postgres  1439  0.0  0.0 320972  9164 ?        Ss   15:32   0:00 postgres: 10/main: wal writer process

postgres  1440  0.0  0.0 321516  6820 ?        Ss   15:32   0:00 postgres: 10/main: autovacuum launcher process

postgres  1441  0.0  0.0 172164  4796 ?        Ss   15:32   0:00 postgres: 10/main: stats collector process

postgres  1442  0.0  0.0 321256  5084 ?        Ss   15:32   0:00 postgres: 10/main: bgworker: logical replication launcher

user      2870  0.0  0.0  15468   976 pts/0    R+   15:35   0:00 grep --color=auto postgres
`
