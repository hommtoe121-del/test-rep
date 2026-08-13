# Перенос данных с реальной БД на тестовую

Источник: https://help.1forma.ru/maintenance_guide/installation/databases/mssql/restore_to_test/

Если тестовый сайт уже настроен и нужно сформировать скрипт для последующих переносов, все параметры можно получить запросами и изменить в них адрес:
 `-- Отбираем данные для последующего обновления

select ApplicationPath,ApplicationIPPath,WSSettings,ApplicationInvitePath,InternalApplicationPath,WAFileInfoServer from Settings
select * from FileProviderSettings_MSSQL        --  Перенастройка подключения к файловому провайдеру описана [здесь](tech_req_move.md) в пункте 10

-- Вместо *** нужно подставить адрес тестового сервера с портом
update settings set ApplicationPath ='***'        -- меняем путь к приложению
update settings set ApplicationIPPath ='***'        -- меняем внешний путь к приложению
update settings set WSSettings='***/tcwebservice.asmx;ЛОГИН;ПАРОЛЬ;'        -- меняем адрес веб сервиса
update settings set ApplicationInvitePath ='***'        -- меняем путь к приложению для приглашений
update settings set InternalApplicationPath='***'        -- меняем системный путь к приложению
update settings set WAFileInfoServer='***'        -- меняем сервер для получения файлов из Office Web Apps, если используется

-- идентификатор приложения, 8 символов (маленькие латинские и цифры, должен быть уникален для каждого приложения, необходим для корректной работы мобильного приложения)
update Settings set ApplicationId='********'

update Settings set EnableEmail=0        -- отключение почты в приложении
update Settings set MailErrors=0        -- отключение отправки ошибок на почту
delete from [PushDeviceTokens]        -- удаление девайс токенов
update EmailSettings set IsEmailEnabled=0, IsEmailJobsEnabled=0        -- отключение джобов почты
delete ServiceMailBoxes        -- удаление сервисных почтовых ящиков в категориях
update Settings set CalendarSyncWithExchange=0        -- отключение синхронизации календаря с Exchange
update Settings set CalendarSyncPeriodicWithExchange=0        -- отключение синхронизации периодических встреч с Exchange
update Settings set CalendarEventSyncWithExchange=0        -- отключение синхронизации календаря с Exchange (событийный режим)
set QUOTED_IDENTIFIER off
update users set DoSyncWithExchange=0        -- отключение Exchange у всех пользователей
set QUOTED_IDENTIFIER on
` ℹ️ Начиная с версии 2.190, ConnectionString шифруется. Перед выполнением этой команды настройте провайдер через веб-интерфейс и скопируйте зашифрованное значение из базы данных.
 `update FileProviderSettings_MSSQL set ConnectionString = 'ШИФР' where FileProviderID = 2   -- указать FileProviderID активной базы.

-- Удаляем очереди шины сообщений

declare @cur cursor

declare @stt varchar ( 200 )

declare @sub varchar ( 200 )

set @cur = cursor scroll for

select
` 'drop table [dbo].[' + name + ']' as stt,
 '[dbo].[' + name + ']' as subsciption
 `from
` sys.objects
 `where
` name like 'bus_%' and
 name <> 'bus_Subscriptions' and --таблица подписок, ее не трогаем
 type \= 'U'
 `open @cur

fetch next from @cur into @stt, @sub

while @@FETCH_STATUS = 0

begin

delete from dbo.bus_Subscriptions where [address] = @sub --удаляем подписки

exec ( @stt ) --удаляем очереди неактуальные

fetch next from @cur into @stt, @sub

end

close @cur

deallocate @cur
` Если на тестовом стенде нужна синхронизация с 1С, то нужно проверить, применилась ли новая декларация (см. здесь последний абзац TC1C_ServiceAppAddress). Если синхронизация не нужна, ее можно удалить в настройках синхронизации.
 Также необходимо отключать или изменять иные интеграции, если они могут изменить рабочие данные в основной системе.
 Полезные ссылки
 Перенос на другой физический сервер
 Тестовая настройка параметров обмена
