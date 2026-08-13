# Обслуживание БД: очистка таблиц-логов

Источник: https://help.1forma.ru/domains/system/db-housekeeping/

База данных 1Формы содержит десятки таблиц-логов, которые растут постоянно. Часть чистится автоматически фоновыми заданиями (Quartz jobs), часть требует ручного обслуживания. Документ отвечает на вопрос «какие таблицы можно безопасно чистить и как». Внутри: автоматические Quartz-джобы, безопасная ручная очистка, таблицы которые трогать нельзя, SQL для мониторинга размера, типичные кейсы и сводная таблица retention.

Все автоматические очистки выполняются только на сервере с включённым признаком `IsJobServer` (настройка `appsettings.json`). Расписание можно переопределить через таблицу `JobRecurrence`.

Основной джоб очистки. Timeout: 120 минут. Удаляет записи из нескольких таблиц за один проход.
 Таблица Колонка даты Retention Настройка retention `ExceptionsLog` `Date` 14 дней `Settings.ShelfLifeErrorLog` (дней). Если NULL — 14 `ActionLog` `ActionTimeStamp` 7 дней Жёстко в коде. Удаляются только записи с Comment LIKE определённым паттернам (см. ниже) `ActionLogFiles` `DateAction` 7 дней `Settings.DayLifeActionLogFiles` (дней). Если NULL — 7 `SphinxIndexLog` `Date` 7 дней Жёстко в коде `TaskCounterLog` — — Очищается автоматически отдельным механизмом ActionLog: удаляются НЕ все записи. Джоб удаляет только строки, где `Comment` содержит: - `Push notification` - `SendVoipPush` - `Comment part` и `comment part` - `Exchange.` - `Не удалось определить пользователя` (локализованная строка) - `выполнен за` и `выполнено за` - `Execute Actions Pack` - `Smart доступ` (retention 14 дней, остальные паттерны — 7 дней)
 Записи ActionLog с другими комментариями (действия пользователей, бизнес-операции) не удаляются автоматически.
 Настройки retention через админку: Общие настройки приложения (`system-settings`) содержат поля: - `ShelfLifeErrorLog` — срок хранения ExceptionsLog (дней) - `DayLifeActionLogFiles` — срок хранения ActionLogFiles (дней)

Помимо `ClearOldLogDbRecordsJob`, в системе работают специализированные джобы для отдельных таблиц.
 Джоб Расписание Таблица Колонка даты Retention `DeleteOldJobLog` ежедневно 04:40 `QrtzJobLog` `Date` 7 дней `DeleteOldJobLog` ежедневно 04:40 `EmailJobSessionLog` `DateStart` 14 дней `PurgePushLogJob` ежедневно 00:02 `PushLog` `PushTimeStamp` 14 дней `ClosingUserSession` ежедневно 00:15 `UserActivityInSystem` `StartTimeOfWork` 14 дней `DeleteOldMobileClientStatsJob` ежедневно 22:00 `MobileClientStats` `EventDate` 1 месяц `DeleteUserAbsenceLogsJob` ежедневно 22:30 `UserAbsenceLog` `DateEnd` 1 месяц `DeleteOldPushTokensJob` ежедневно 04:03 `PushDeviceTokens` `DateAdded` 365 дней `DeleteOldResizedImages` каждый час `ImageResized` `LastAccessDate` 10 дней `DeleteDSSOlgLogs` каждый час `DSSCryptoProTransactionLogs` `Created` по порогу `DeleteOldLockTokensJob` ежедневно 04:32 `WebDavFileLocks` `Expires` просроченные `Delete1CLogExceptWeek` воскресенье 05:15 `Sync1CLog` `Date` 7 дней `ClearAutomationScriptsLog` воскресенье 23:59 `AutomationScriptsLog` `Date` 7 дней `ClearCommentRecipientsArchive` суббота 05:00 `CommentRecipientsArchive` по CommentId (Comments.Date) 1 год `CleanMailBoxesJob` ежедневно 01:15 `EmailJobReceiveQueueLog` — 4 дня `CleanMailBoxesJob` ежедневно 01:15 `EmailJobSendQueueLog` — 4 дня `CleanMailBoxesJob` ежедневно 01:15 `EmailJobSessionLog` — 4 дня `SyncHookServiceRoutineJob` каждую минуту `HookServiceEvents` `LastUpdate` 4 дня `CloseStaleThreadsJob` ежедневно 02:00 `Comments` (треды) по дате последнего сообщения 14 дней Email-джобы (`CleanMailBoxesJob` и связанные) активны только при `EmailSettings.IsEmailInstalled && IsEmailEnabled && IsEmailJobsEnabled`.
 `RebuildIndexesJob` (02:30, ежедневно) вызывает SP `dbo.tc_NightlyMaintance` — пересоздание индексов, обновление статистики. Не удаляет данные, но критично для производительности запросов очистки.

Раздел перечисляет таблицы, которые содержат только технические логи, не связанные с бизнес-данными. Удаление старых записей безопасно.

Чистится автоматически (14 дней по умолчанию). При аномальном росте можно почистить вручную.
 Структура: `ID`, `Date`, `Exception`, `SourceID`, `AppPool`, `AppVersion`
 `-- Проверить размер
SELECT COUNT(*) AS cnt,
       MIN(Date) AS oldest,
       MAX(Date) AS newest
FROM dbo.ExceptionsLog WITH (NOLOCK);

-- Удалить записи старше 7 дней
DELETE FROM dbo.ExceptionsLog
WHERE Date < DATEADD(DAY, -7, GETDATE());
` Рекомендация: 7-14 дней. Если таблица содержит миллионы записей, удалять батчами (см. раздел 5.1).

Внимание: ActionLog содержит как техническую информацию (push, exchange), так и бизнес-аудит (действия пользователей). Автоочистка удаляет только технические паттерны. Полная очистка по дате уничтожит историю аудита.
 Структура: `LogID`, `SessionUserID`, `ActionTimeStamp`, `Comment`, `GroupID`, `SubcatID`, `CategoryID`, `UserID`, `SignatureID`, `UserAgent`, `UserComputerName`, `ActionID`
 `-- Проверить размер и распределение
SELECT COUNT(*) AS cnt,
       MIN(ActionTimeStamp) AS oldest,
       MAX(ActionTimeStamp) AS newest
FROM dbo.ActionLog WITH (NOLOCK);

-- Безопасно: удалить только технические записи старше 7 дней
-- (повторяет логику ClearOldLogDbRecordsJob)
DELETE FROM dbo.ActionLog
WHERE ActionTimeStamp < DATEADD(DAY, -7, GETDATE())
  AND (Comment LIKE '%Push notification%'
    OR Comment LIKE 'SendVoipPush%'
    OR Comment LIKE '%Comment part%'
    OR Comment LIKE 'comment part%'
    OR Comment LIKE 'Exchange.%'
    OR Comment LIKE '%выполнен за%'
    OR Comment LIKE '%выполнено за%'
    OR Comment LIKE 'Execute Actions Pack%'
    OR Comment LIKE 'Smart доступ%');
` Рекомендация: технические паттерны — 7 дней. Полная очистка — согласовать с заказчиком (потеря аудита).

ActionLogFiles — лог файловых действий.
 Структура: `ID`, `Action`, `UserID`, `FileID`, `FileName`, `DateAction`, `IsExtParam`, `VersionId`
 `-- Проверить размер
SELECT COUNT(*) AS cnt,
       MIN(DateAction) AS oldest
FROM dbo.ActionLogFiles WITH (NOLOCK);

-- Удалить старше 7 дней (по умолчанию автоочистки)
DELETE FROM dbo.ActionLogFiles
WHERE DateAction < DATEADD(DAY, -7, GETDATE());
` Рекомендация: 7 дней. Настраивается через `Settings.DayLifeActionLogFiles`.
 Sync1CLog — лог синхронизации 1С. Чистится автоматически по воскресеньям. При интенсивном обмене с 1С может расти быстро в течение недели.
 Структура: `ID`, `Date`, `SettingsName`, `Origin`, `Message`, `TechData`, `Server1F`, `Server1C`, `Duration`
 `-- Удалить старше 7 дней
DELETE FROM dbo.Sync1CLog
WHERE Date < DATEADD(DAY, -7, GETDATE());
` Рекомендация: 7 дней. Если синхронизация с 1С не используется — можно удалить всё.

Не чистится автоматически. Может расти неограниченно.
 Структура: `ID`, `UserID`, `LoginDate`, `IP`, `ServerIP`, `URL`, `LocationID`, `UserAgent`, `UserComputerName`, `AdditionalInfo`
 `-- Проверить размер
SELECT COUNT(*) AS cnt,
       MIN(LoginDate) AS oldest
FROM dbo.LoginsLog WITH (NOLOCK);

-- Удалить старше 90 дней
DELETE FROM dbo.LoginsLog
WHERE LoginDate < DATEADD(DAY, -90, GETDATE());
` Рекомендация: 90 дней. Некоторые заказчики требуют хранить дольше (регуляторные требования к аудиту входов). Согласовать retention с ИБ-политикой клиента.

Не чистится автоматически. Накапливается при активной синхронизации с Exchange.
 Структура: `SyncID`, `ICalUid`, `Email`, `TaskId`, `InvokeSource`, `InvokeChangeTime`, `Description`, `JobDT`, `SyncFinished`, `ErrorDescription`, `CleanGlobalObjectId`
 `-- Проверить размер
SELECT COUNT(*) AS cnt,
       MIN(JobDT) AS oldest
FROM dbo.ExchangeSyncLog WITH (NOLOCK);

-- Удалить старше 30 дней
DELETE FROM dbo.ExchangeSyncLog
WHERE JobDT < DATEADD(DAY, -30, GETDATE());
` Рекомендация: 30 дней. Если Exchange не используется — можно удалить всё.

QrtzJobLog — лог выполнения Quartz-заданий.
 Структура: `ID`, `Date`, `IsSuccess`, `Message`, `SecondsExecuted`, `JobKey`
 `DELETE FROM dbo.QrtzJobLog
WHERE Date < DATEADD(DAY, -7, GETDATE());
` Рекомендация: 7 дней. Чистится автоматически.
 SphinxIndexLog — лог индексации Sphinx.
 Структура: `ID`, `Date`, `IsIncrement`, `IndexName`
 `DELETE FROM dbo.SphinxIndexLog
WHERE Date < DATEADD(DAY, -7, GETDATE());
` Рекомендация: 7 дней. Чистится автоматически.
 PushLog — лог пуш-уведомлений.
 Структура: `ID`, `PushTimeStamp`, `SendServerName`, `DeviceTokens`, `Data`, `Type`, `Priority`, `SenderUserID`, `RecipientUserID`
 `DELETE FROM dbo.PushLog
WHERE PushTimeStamp < DATEADD(DAY, -14, GETDATE());
` Рекомендация: 14 дней. Чистится автоматически.

AutomationScriptsLog — лог смарт-действий.
 Структура: `Id`, `Type`, `ObjectKey`, `ObjectParams`, `AdditionalInfo`, `TaskId`, `UserID`, `ThreadId`, `Duration`, `Date`, `ObjectName`, `PlanExecution`, `AppPool`
 `DELETE FROM dbo.AutomationScriptsLog
WHERE Date < DATEADD(DAY, -7, GETDATE());
` Рекомендация: 7 дней. Чистится автоматически (воскресенье).
 MobileClientStats — статистика мобильных клиентов.
 Структура: `ID`, `EventDate`, `AppID`, `Udid`, `Firmware`, `Version`, `UserId`, `ActionTypeID`, `Action`, `SessionDuration`
 `DELETE FROM dbo.MobileClientStats
WHERE EventDate < DATEADD(MONTH, -1, GETDATE());
` Рекомендация: 1 месяц. Чистится автоматически.

Содержит `VARBINARY(MAX)` — тяжёлая таблица. Данные — кеш, не источник.
 Структура: `Id`, `FileId`, `VersionId`, `Width`, `Height`, `MaxDimension`, `IsCropped`, `IsLandscape`, `CreationDate`, `LastAccessDate`, `ReadCount`, `Content`
 `-- Проверить размер (может быть десятки GB)
SELECT COUNT(*) AS cnt,
       SUM(DATALENGTH(Content)) / 1024 / 1024 AS size_mb
FROM dbo.ImageResized WITH (NOLOCK);

-- Удалить неиспользуемые старше 10 дней (логика автоочистки)
DELETE FROM dbo.ImageResized
WHERE LastAccessDate < DATEADD(DAY, -10, GETDATE());
` Рекомендация: 10 дней. Чистится автоматически каждый час. Безопасно удалить всё — данные будут пересозданы при обращении.

Не чистится автоматически.
 Структура: `ID`, `FileActionsLogTypeID`, `UserID`, `TaskID`, `ExtParamID`, `FileID`, `FileStorageFolderID`, `DateAction`, `VersionId`, `AddInfo`
 `-- Проверить размер
SELECT COUNT(*) AS cnt,
       MIN(DateAction) AS oldest
FROM dbo.FileActionsLog WITH (NOLOCK);

-- Удалить старше 90 дней
DELETE FROM dbo.FileActionsLog
WHERE DateAction < DATEADD(DAY, -90, GETDATE());
` Рекомендация: 90 дней. Согласовать с заказчиком — некоторые используют аудит файлов.

UserActivityInSystem — сессии пользователей.
 Структура: `ID`, `UserID`, `StartTimeOfWork`, `UpdateTime`, `EndTime`
 `DELETE FROM dbo.UserActivityInSystem
WHERE StartTimeOfWork < DATEADD(DAY, -14, GETDATE());
` Рекомендация: 14 дней. Чистится автоматически.
 DSSCryptoProTransactionLogs — логи DSS/КриптоПро.
 Структура: `Id`, `TransactionId`, `Info`, `Status`, `TaskId`, `TaskSignatureId`, `UserId`, `Created`
 `DELETE FROM dbo.DSSCryptoProTransactionLogs
WHERE Created < DATEADD(DAY, -30, GETDATE());
` Рекомендация: 30 дней. Чистится автоматически каждый час. Если ЭП DSS не используется — можно удалить всё.
 Email-логи чистятся `CleanMailBoxesJob` ежедневно (01:15). При необходимости — вручную:
 `-- EmailJobReceiveQueueLog
DELETE FROM dbo.EmailJobReceiveQueueLog
WHERE DateProcessed < DATEADD(DAY, -4, GETDATE());

-- EmailJobSendQueueLog
DELETE FROM dbo.EmailJobSendQueueLog
WHERE DateProcessed < DATEADD(DAY, -4, GETDATE());

-- EmailJobSessionLog
DELETE FROM dbo.EmailJobSessionLog
WHERE DateStart < DATEADD(DAY, -14, GETDATE());
`

Удаление данных из этих таблиц приведёт к потере бизнес-данных, нарушению целостности или неработоспособности системы.
 Таблица Почему нельзя трогать `Tasks` Задачи — основная бизнес-сущность. Удаление = потеря работы пользователей `Comments` Комментарии — история коммуникации по задачам `CommentRecipients` Получатели комментариев — привязка уведомлений, непрочитанных `ExtParamValues` Значения дополнительных параметров — бизнес-данные задач `Users` Пользователи — FK-зависимости по всей БД `Subcategories` Категории — структура процессов `Categories` Разделы — верхнеуровневая структура `TaskSignatures` Подписи — бизнес-процессы согласования `FileStorageFiles` / `FileStorageFileVersions` Файловое хранилище. Для очистки есть цепочка джобов (PrepareUnused → RemovePrepared → RemoveUnused), которая требует флаг `EnableRemovalOfUnusedFiles` `Emails` Письма — бизнес-данные при включённой почте `MessageQueue` Очередь событий. Удаление вызовет потерю необработанных событий. Чистится автоматически при обработке (QueueEventsJob, 5 сек) `SmartRecurrences` Расписания смарт-действий — удаление остановит автоматизацию `Denormalization_*` / `Denorm_*` Денормализованные таблицы. Пересоздаются автоматически (DenormalizationBadJob). Удаление данных вызовет проблемы до следующего ночного ребилда `Settings` Единственная строка — глобальные настройки. Удаление = неработоспособность системы `SettingsCustom` Кастомные настройки — feature flags



Запрос возвращает TOP 30 таблиц, отсортированных по суммарному объёму на диске.
 `SELECT TOP 30
    s.Name + '.' + t.Name AS TableName,
    p.rows AS RowCount,
    CAST(ROUND(SUM(a.total_pages) * 8.0 / 1024, 2) AS DECIMAL(18,2)) AS TotalMB,
    CAST(ROUND(SUM(a.used_pages) * 8.0 / 1024, 2) AS DECIMAL(18,2)) AS UsedMB,
    CAST(ROUND((SUM(a.total_pages) - SUM(a.used_pages)) * 8.0 / 1024, 2) AS DECIMAL(18,2)) AS UnusedMB
FROM sys.tables t
INNER JOIN sys.indexes i ON t.object_id = i.object_id
INNER JOIN sys.partitions p ON i.object_id = p.object_id AND i.index_id = p.index_id
INNER JOIN sys.allocation_units a ON p.partition_id = a.container_id
INNER JOIN sys.schemas s ON t.schema_id = s.schema_id
WHERE t.is_ms_shipped = 0
GROUP BY s.Name, t.Name, p.rows
ORDER BY SUM(a.total_pages) DESC;
`

Запрос фильтрует только известные лог-таблицы — удобно для целевого мониторинга роста логов.
 `SELECT
    t.Name AS TableName,
    p.rows AS RowCount,
    CAST(ROUND(SUM(a.total_pages) * 8.0 / 1024, 2) AS DECIMAL(18,2)) AS TotalMB
FROM sys.tables t
INNER JOIN sys.indexes i ON t.object_id = i.object_id
INNER JOIN sys.partitions p ON i.object_id = p.object_id AND i.index_id = p.index_id
INNER JOIN sys.allocation_units a ON p.partition_id = a.container_id
WHERE t.Name IN (
    'ExceptionsLog', 'ActionLog', 'ActionLogFiles', 'SphinxIndexLog',
    'QrtzJobLog', 'PushLog', 'Sync1CLog', 'AutomationScriptsLog',
    'LoginsLog', 'ExchangeSyncLog', 'MobileClientStats', 'ImageResized',
    'UserActivityInSystem', 'FileActionsLog', 'PushDeviceTokens',
    'DSSCryptoProTransactionLogs', 'EmailJobSessionLog',
    'EmailJobReceiveQueueLog', 'EmailJobSendQueueLog',
    'CommentRecipientsArchive', 'TaskCounterLog', 'WebDavFileLocks',
    'UserAbsenceLog', 'CustomerZoneLog', 'HookServiceEvents'
)
GROUP BY t.Name, p.rows
ORDER BY SUM(a.total_pages) DESC;
` Общий размер БД:
 `SELECT
    DB_NAME() AS DatabaseName,
    CAST(SUM(size) * 8.0 / 1024 AS DECIMAL(18,2)) AS SizeMB,
    CAST(SUM(size) * 8.0 / 1024 / 1024 AS DECIMAL(18,2)) AS SizeGB
FROM sys.database_files;
`

Алгоритм диагностики при резком росте базы данных.

- Запустить запрос из 4.1 — определить, какие таблицы выросли больше всего.

- Типичные виновники:

- `ImageResized` — содержит BLOB, может занимать десятки GB. Безопасно: `DELETE WHERE LastAccessDate < DATEADD(DAY, -10, GETDATE())`.

- `ExceptionsLog` — массовые ошибки (интеграция, 1С, LDAP). Проверить лог ошибок — устранить причину, затем почистить.

- `ActionLog` — при активной автоматизации растёт быстро. Безопасно чистить технические паттерны (см. 2.2).

- `LoginsLog` — не чистится автоматически. Может содержать миллионы записей за годы работы.

- `FileActionsLog` — не чистится автоматически. Растёт при активной работе с файлами.

- `Sync1CLog` — при интенсивном обмене с 1С. Чистить 7+ дней.

- При удалении больших объёмов данных (>1 млн строк) — удалять батчами:
  `-- Батчевое удаление (по 50000 строк за итерацию)
DECLARE @deleted INT = 1;
WHILE @deleted > 0
BEGIN
    DELETE TOP (50000)
    FROM dbo.ExceptionsLog
    WHERE Date < DATEADD(DAY, -14, GETDATE());

    SET @deleted = @@ROWCOUNT;
END
`
- После массового удаления:

- Проверить, что `RebuildIndexesJob` работает (перестроит индексы ночью).

- При необходимости немедленно: `ALTER INDEX ALL ON dbo.ExceptionsLog REORGANIZE;`

- Если нужно вернуть место ОС: `DBCC SHRINKFILE(N'data_file_name', target_size_mb);` — выполнять только в нерабочее время, приводит к фрагментации.

Пошаговая диагностика, если джоб очистки не выполняется по расписанию.

-  Проверить статус джоба: интерфейс администрирования, раздел «Задания по таймеру» или:    `GET /api/admin/jobs
`    Искать `ClearOldLogDbRecordsJob` — статус должен быть `WAITING`.


-  Если статус `BLOCKED`:


- Проверить время последнего выполнения.

- Разблокировать в интерфейсе или через API.

-  Запустить вручную.


-  Если статус `PAUSED`:


-  Возобновить (Resume) в интерфейсе.


-  Если джоб падает с ошибкой:


- Проверить лог: `SELECT TOP 10 * FROM dbo.QrtzJobLog WHERE JobKey = 'ClearOldLogDbRecordsJob' ORDER BY Date DESC;`

-  Частая причина: timeout на огромных таблицах. Предварительно вручную удалить основной объём (батчами), затем джоб справится с ежедневной очисткой.


-  Проверить, что на сервере включён признак `IsJobServer` в `appsettings.json`, — именно там ожидается выполнение джобов.

При первичной очистке запущенной системы, где логи копились годами:

- Начать с `ImageResized` — даёт максимальный эффект, данные пересоздаются автоматически.

- Затем `ExceptionsLog` — обычно второй по размеру.

- Затем `ActionLog` (только технические паттерны) и `ActionLogFiles`.

- Затем остальные логи — `LoginsLog`, `FileActionsLog`, `Sync1CLog`, `ExchangeSyncLog`.

- Каждую таблицу чистить батчами (50000 строк за проход).

- Между таблицами проверять `sp_spaceused` — убедиться, что место освободилось.
  Связанные документы: Системные настройки — администрирование

Справочная таблица по всем лог-таблицам системы: режим очистки и безопасность ручного вмешательства.
 Таблица Автоочистка Retention по умолчанию Настраиваемый Безопасно чистить вручную `ExceptionsLog` ClearOldLogDbRecordsJob 14 дней Settings.ShelfLifeErrorLog Да `ActionLog` ClearOldLogDbRecordsJob 7 дней (только тех. паттерны) Нет Технические — да; бизнес — согласовать `ActionLogFiles` ClearOldLogDbRecordsJob 7 дней Settings.DayLifeActionLogFiles Да `SphinxIndexLog` ClearOldLogDbRecordsJob 7 дней Нет Да `QrtzJobLog` DeleteOldJobLog 7 дней Нет Да `EmailJobSessionLog` DeleteOldJobLog 14 дней Нет Да `PushLog` PurgePushLogJob 14 дней Нет Да `UserActivityInSystem` ClosingUserSession 14 дней Нет Да `MobileClientStats` DeleteOldMobileClientStatsJob 1 месяц Нет Да `UserAbsenceLog` DeleteUserAbsenceLogsJob 1 месяц Нет Да `PushDeviceTokens` DeleteOldPushTokensJob 365 дней Нет Да `ImageResized` DeleteOldResizedImages 10 дней Нет Да (кеш) `DSSCryptoProTransactionLogs` DeleteDSSOlgLogs по порогу Нет Да `WebDavFileLocks` DeleteOldLockTokensJob просроченные Нет Да `Sync1CLog` Delete1CLogExceptWeek 7 дней Нет Да `AutomationScriptsLog` ClearAutomationScriptsLog 7 дней Нет Да `CommentRecipientsArchive` ClearCommentRecipientsArchive 1 год Нет Да `LoginsLog` Нет — — Да (согласовать ИБ) `ExchangeSyncLog` Нет — — Да `FileActionsLog` Нет — — Да (согласовать аудит) `CustomerZoneLog` Нет — — Да `HookServiceEvents` SyncHookServiceRoutineJob 4 дня Нет Да
