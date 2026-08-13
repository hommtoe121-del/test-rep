# Работа с объектами ASSEMBLY и хранимыми процедурами

Источник: https://help.1forma.ru/maintenance_guide/installation/databases/mssql/stored-procedures/1f_sql_proc_admin_assembly/

В большинстве случаев база данных "Первой Формы" имеет название "D10task", но может называться и по-другому (информацию о названии базы данных можно уточнить в тех. поддержке).
 В данном разделе представлено описание структуры файла create_crl.sql.
 1. Процедура включения доверия к БД. Нужно, чтобы процедуры вообще работали, это "доверие" сервера к БД
 `alter database [D10Task] set trustworthy on
` 2. Проверка наличия пользователя D10TaskOwner в БД.
 `if not exists (SELECT name FROM master.dbo.syslogins WHERE name = N'D10TaskOwner') —
begin
  create login D10TaskOwner with password = '<укажите_надёжный_пароль>',cHECK_EXPIRATION = OFF, cHECK_POLICY = OFF  -- Замените на надёжный пароль. Не используйте пароль из документации.
end

EXEC dbo.sp_changedbowner @loginame = N'D10TaskOwner', @map = false — если пользователя "D10TaskOwner" нет, то система назначает его владельцем БД.
go
` 3. Пользователю, назначенному владельцем БД, выдаются права на использование исполняемых файлов *.dll
 `use [master]
go
GRANT EXTERNAL ACCESS ASSEMBLY TO D10TaskOwner
` 4. Создается привязка объектов ASSEMBLY к сборкам StoredProcedures.dll и StoredProcedures.XmlSerializers.dll
 `use [D10Task]
go
create ASSEMBLY StoredProcedures — генерируется сборка с именем StoredProcedures.dll  (параметр assembly_name). В данном наименование сборки StoredProcedures. Обратите внимание, что данное имя должно быть уникальным в БД.

FROM 'C:\Program Files (x86)\TCSQL\StoredProcedures.dll' — локальный путь сборки исполняемых файлов *.dll. Путь нужно указывать тот, куда были загружены *.dll файлы.
` WITH PERMISSION_SET = EXTERNAL_ACCESS; — сборка StoredProceduress.dll получает доступ к внешним ресурсам `GO

create ASSEMBLY [StoredProcedures.XmlSerializers] — генерируется сборка с именем StoredProcedures.dll  (параметр assembly_name). В данном наименование сборки StoredProcedures. Обратите внимание, что данное имя должно быть уникальным в БД.

FROM 'C:\Program Files (x86)\TCSQL\StoredProcedures.XmlSerializers.dll'
` WITH PERMISSION_SET = SAFE; — генерируется сборка с именем StoredProcedures.XmlSerializers.dll ограниченный доступ к внешним ресурсам. Путь нужно указывать тот, куда были загружены *.dll файлы. `GO
` На данном шаге производится проверка наличия хранимой процедуры для отправки почты почты пользователя "D10TaskOwner". `use [master]
go
if not exists  (select id from sysobjects where name = 'xp_smtp_sendmail')  — проверяем существует ли хранимая процедура для отправки почты "xp_smtp_sendmail"
begin
  exec sp_addextendedproc 'xp_smtp_sendmail', 'C:\Program Files (x86)\TCSQL\xpsmtp80.dll'  — в случае, если необходимая хранимая процедура ("xp_smtp_sendmail") не обнаружена, система регистрирует её в Microsoft SQL Server для последующего использования
  grant execute on xp_smtp_sendmail to public — вызов хранимой процедуры "xp_smtp_sendmail"
end
go
`
 5. Вызов хранимых процедур, описанных в разделе Хранимые процедуры "Первой Формы". При этом источником указываются объекты ASSEMBLY, описанные в пункте 4.
 `use [D10Task]
go

CREATE PROCEDURE tc_ws_newTask(
`     @UserID int, @Task nvarchar(4000), @OrderedTime datetime\=NULL,     @Category int, @Comment nvarchar(4000) = '',   @extParamStr nvarchar(4000) = '',     @perfID int = NULL, @priorityID int = 1, @D3TaskID int = 0, @ParentID int = 0,     @UserNick nvarchar(20) = '', @remind bit = 0, @PropagateSubscribers bit = 1) AS EXTERNAL NAME StoredProcedures.StoredProcedures.NewTask
 ℹ️ В данном разделе хранимая процедура tc_ws_newTask представлена для примера, файл create_crl.sql содержит полный список процедур. С назначением и описанием параметров хранимых процедур более подробно  можно ознакомиться в разделе Хранимые процедуры "Первой Формы".
 Полезные ссылки
 Прямая ссылка на файл create_crl.sql, необходимый для работы с классом ASSEMBLY
