# Резервное копирование СУБД PostgreSQL

Источник: https://help.1forma.ru/maintenance_guide/installation/databases/postgresql/tech_req_server_backup_postgresql/

Резервное копирование баз данных PostgreSQL — обязательная задача для обеспечения целостности данных и их восстановления в случае сбоя. В этом руководстве рассмотрены различные методы резервного копирования, рекомендации по хранению резервных копий и инструкции по восстановлению данных.

Полный бэкап включает копирование всех баз данных, хранящихся на сервере PostgreSQL. Для этого используется утилита pg_dumpall.
 Шаги:
 1. Откройте терминал.
 2. Выполните следующую команду:
 pg_dumpall -U dbo -f all_databases_backup.dump
 Здесь dbo — это имя пользователя PostgreSQL с необходимыми правами.
 Примечания:

-  Файл all_databases_backup.dump будет содержать полный дамп всех баз данных.


-  Убедитесь, что у пользователя PostgreSQL есть необходимые привилегии для доступа ко всем базам данных.

Для резервного копирования одной базы данных используется утилита pg_dump.
 Шаги:
 1. Откройте терминал.
 2. Выполните следующую команду:
 pg_dump -U dbo -Fc -f d10task.dump d10task
 Здесь dbo — это имя пользователя PostgreSQL, а d10task — название базы данных, которую нужно сохранить.
 Примечания:

- Файл d10task.dump будет содержать дамп указанной базы данных в формате custom.

Шаги:
 1. Откройте терминал
 2. Выполните следующую команду:

psql -U dbo -f all_databases_backup.dump
 Где:

-  dbo — имя пользователя PostgreSQL с необходимыми правами.


-  all_databases_backup.dump — файл дампа всех баз данных PostgreSQL в plain-text формате (создан утилитой pg_dumpall).

Шаги:
 1. Создайте новую базу данных или используйте существующую:
 createdb -U dbo d10task
 2. Выполните следующую команду для восстановления из дампа:
 pg_restore -U dbo -d d10task -Fc -c -v d10task.dump
 Где:

-  dbo — Имя пользователя PostgreSQL.


-  d10task — Имя базы данных, в которую будет восстанавливаться дамп.


-  d10task.dump — Файл дампа конкретной базы данных PostgreSQL в формате custom .

  Опции:

-  -Fc — формат дампа custom, используемый для бинарных файлов.


-  -c — очищает базу данных перед восстановлением.


-  -v — выводит подробный вывод на экран.

  3. Введите пароль, если это необходимо.

По окончании восстановления в новой базе необходимо настроить схемы:
 1. Зайдите в базу:
 psql -d d10task
 2. Выполните скрипт:
 `alter role D10TaskUser set search_path = "dbo", "public";

alter role dbo set search_path = "dbo", "public";

alter user MigrationsDaemon set search_path = "dbo", "public";

alter user rebus set search_path = "rebus", "public";

grant dbo, rebus to MigrationsDaemon;

grant all on all tables in schema dbo, sys to dbo, MigrationsDaemon;

grant all on all sequences in schema dbo, sys to dbo, MigrationsDaemon;

grant all on all functions in schema dbo, sys to dbo, MigrationsDaemon;

grant all on schema dbo, sys to dbo, MigrationsDaemon;

alter default privileges in schema dbo, sys, SignalR grant all on tables to dbo,
` MigrationsDaemon;
 `alter default privileges in schema dbo, sys, SignalR grant all on sequences to
` dbo, MigrationsDaemon;
 `alter default privileges in schema dbo, sys, SignalR grant all on functions to
` dbo, MigrationsDaemon;
 `alter default privileges in schema dbo, sys, SignalR grant all on types to dbo,
` MigrationsDaemon;
 3. Выйдите из консоли psql
 \q

1. Частота резервного копирования
 Рекомендуем выполнять ежедневный полный бэкап для критически важных данных.
 2. Глубина хранения
 Храните резервные копии минимум за последние 7-14 дней. Архивируйте месячные и годовые копии для долгосрочного хранения.
 3. Безопасность
 Храните резервные копии в защищённом месте. Используйте шифрование для резервных копий, особенно при хранении в облачных хранилищах.
 4. Автоматизация
 Используйте скрипты и планировщики задач (например, cron в Linux) для автоматизации процесса резервного копирования и очистки старых копий.
 Пример скрипта для автоматизации полного бэкапа
 `#!/bin/bash

# Параметры
USER="username"
BACKUP_DIR="/path_to_backup"
DATE=$(date +%Y%m%d%H%M)

# Создание полного бэкапа
pg_dumpall -U $USER -f $BACKUP_DIR/all_databases_backup_$DATE.dump

# Удаление бэкапов старше 14 дней
find $BACKUP_DIR -type f -name "*.dump" -mtime +14 -exec rm {} \;

echo "Резервное копирование завершено."
` Этот скрипт создаёт полный бэкап всех баз данных и удаляет резервные копии старше 14 дней. Добавьте созданный скрипт в планировщик задач для регулярного выполнения. Вот как это можно сделать в OS Linux:
 1. Откройте cron таблицу для редактирования с помощью команды:
 crontab -e
 2. Если система спрашивает, какой текстовый редактор использовать, выберите тот, который вам удобен.
 3. В открывшемся файле добавьте следующую строку в конец файла, чтобы задать выполнение скрипта (например, в 23:00 каждую ночь):
 0 23 * * * /path_to_script.sh

-  Первое число (0) указывает на минуты (в данном случае, 0 минут).


-  Второе число (23) указывает на часы (в данном случае, 23 часа или 11 вечера).


-  Звёздочки (*) означают "каждый день", "каждый месяц" и "каждый день недели" соответственно.

  4. Сохраните файл и закройте текстовый редактор.
 Теперь скрипт /path_to_script.sh будет выполняться каждую ночь в 23:00 по местному времени. Убедитесь, что путь к скрипту /path_to_script.sh указан правильно и что у него есть права выполнения для пользователя, под которым будет запущен cron.
 Следование этим рекомендациям и использование приведённых инструкций поможет обеспечить надёжное резервное копирование ваших баз данных PostgreSQL и их быстрое восстановление в случае необходимости.
