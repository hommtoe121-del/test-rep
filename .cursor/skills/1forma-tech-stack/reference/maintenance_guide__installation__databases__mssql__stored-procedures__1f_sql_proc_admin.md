# stored-procedures

Источник: https://help.1forma.ru/maintenance_guide/installation/databases/mssql/stored-procedures/1f_sql_proc_admin/

1. Сделайте резервную копию базы (см. раздел Обновление приложения "Первая Форма", пункт 2).
 2. Выполните скрипт tc_ws_drop.sql (скрипт удаляет tc_ws_ процедуры и ASSEMBLY-ы). Если остались какие-то tc_ws процедуры, удалите их вручную и повторите удаление ASSEMBLY.
 3. Скопируйте TCSQLProject.dll и TCSQLProject.XmlSerializers.dll в папку "C:\Program Files (x86)\TCSQL\" или "C:\Program Files\TCSQL\" на сервере, где установлен SQL-сервер (копировать с заменой).
 4. Подправив, при необходимости, путь к папке с *.dll и название базы, выполните create_crl.sql — создание ASSEMBLY и хранимых процедур (раздел Работа с объектами ASSEMBLY и хранимыми процедурами).
 5. Проверьте работу, выполнив запрос в SQL Studio:
 `exec  tc\_ws\_addComment \_номер админа\_ , \_номер задачи\_ , 'тест'`
 В результате в указанной задаче должен появиться комментарий от администратора.
 Прямые ссылки на файлы:
 tc_ws_drop.sql
 TCSQLProject.dll
 TCSQLProject.XmlSerializers.dll
 create_crl.sql
