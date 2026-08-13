# Exchange — диагностика синхронизации пользователя

Источник: https://help.1forma.ru/domains/calendar/runbook-exchange-user-sync/

Руководство для поддержки по разбору инцидентов синхронизации с Exchange на уровне пользователя: ошибки доступа к папкам и почтовому ящику, «самопроизвольное» отключение синхронизации, рост счётчика неудачных попыток. Документ описывает устройство механизма (режимы, счётчик ошибок, классификация ошибок, автоматическое восстановление), чек-лист разбора, таблицу «симптом → вероятная причина → что проверить», готовые SQL-запросы и список того, что приложить при обращении в поддержку 1Ф.

Признаки инцидента, при которых применяется это руководство:

- Ошибки `ErrorFolderNotFound`/`ErrorInvalidFolderId`/`ErrorNonExistentMailbox`.

- Синхронизация пользователя «самопроизвольно» отключилась.

- Растёт счётчик неудачных попыток Exchange.

- Встречи не появляются/не обновляются у конкретного пользователя.
  Что приложить при обращении в поддержку 1Ф:

- `userId`, время ошибки, точный `ServiceError`.

- Снимок `GET /api/admin/users/{userId}/service/settings`.

- Результаты SQL-запросов из раздела 5.

- Что уже делали: `reset-fails`, смена режима (`update-sync-mode`), повтор синхронизации.

- Масштаб: один пользователь или массово.



Режим определяется парой флагов пользователя:

- `Disabled` → `DoSyncWithExchange = false` и `CalendarEwsDirectSync = false`.

- `Sync` → `DoSyncWithExchange = true` и `CalendarEwsDirectSync = false`.

- `Direct` → `DoSyncWithExchange = false` и `CalendarEwsDirectSync = true`.
  Admin API для управления:

- `POST /api/admin/users/{userId}/exchange/reset-fails`

- `POST /api/admin/users/{userId}/exchange/update-sync-mode`

- `GET /api/admin/users/{userId}/service/settings` (для контроля текущего режима/счётчиков)
  Механика счётчика ошибок и автоотключения:

- Ошибки Exchange увеличивают `SyncWithExchangeFailedAttempts`.

- Лимит берётся из `CalendarExchangeRetryLimit`.

- Если лимит `!= 0` и попытки `> limit`, режим пользователя переводится в `Disabled`.

- `reset-fails` сбрасывает счётчик в `0`, но сам режим при необходимости надо выставить отдельно.

Система обрабатывает два класса ошибок Exchange:

- `AccessDenied`:

- очищаются `EwsFolders`/`EwsFolderPermissions` для текущего пользователя,

- увеличивается счётчик ошибок.

- Ошибки «плохой пользователь/ящик»:

- `ErrorNonExistentMailbox`, `ErrorFolderNotFound`, `ErrorInvalidFolderId`, `ErrorInvalidSmtpAddress`, `ErrorMailRecipientNotFound`, `ErrorItemNotFound`,

- при наличии `SmtpAddress` в деталях могут отключаться пользователи с этим email,

- иначе увеличивается счётчик ошибок по пользователю.
  Автоматическое восстановление: фоновая задача `EnableDoSyncWithExchangeJob` может повторно включить синхронизацию пользователям, отключённым именно «по ошибке Exchange» (служебный флаг `Sync.Exchange.DisabledByError`).

При разборе инцидента пройдите по шагам:

- Зафиксировать `userId`, время ошибки, текст `ServiceError`.

- Проверить сервисные настройки пользователя (`service/settings`):

- `ewsMode`,

- `syncWithExchangeFailedAttempts`,

- `calendarExchangeRetryLimit`,

- `sid`.

- Если попытки выше лимита:

- сбросить fails,

- вернуть нужный режим синхронизации вручную.

- Проверить, какой класс ошибки:

- `AccessDenied` → проверка доступа, перевоплощения и кэша папок,

- ошибки ящика/папки → проверка email, папки и почтового ящика календаря.

- Для `AccessDenied` убедиться, что после очистки кэша папок и повторной попытки состояние изменилось.

- Если проблема массовая (несколько пользователей) — проверять Exchange service settings и инфраструктурную доступность EWS.

Сопоставление частых симптомов с вероятной причиной и первой проверкой:
 Симптом Вероятная причина Что проверить первым У пользователя `ewsMode = Disabled` после серии ошибок превышен лимит повторов `syncWithExchangeFailedAttempts` vs `calendarExchangeRetryLimit` `ErrorAccessDenied` права/перевоплощение/доступ к папке записи `EwsFolders`/`EwsFolderPermissions`, параметры перевоплощения `ErrorNonExistentMailbox`/`ErrorInvalidSmtpAddress` некорректный email/mailbox `Users.Email`, детали `SmtpAddress` в логе `ErrorFolderNotFound`/`ErrorInvalidFolderId` невалидный folder link/изменение папки в Exchange повторная инициализация папок, проверки кэша папок После `reset-fails` ситуация не изменилась сбросили только счётчик, но не режим текущий `ewsMode` и `update-sync-mode` Событие создано в Outlook, комментарий в задачу не приходит, нет записей в `AutomationScriptsLog` (Type=10) `Settings.DefaultEwsServiceId = NULL` и нет AD-профилей, привязанных к EWS → список EWS-сервисов пуст → подписки не стартовали (тихий отказ без лога) `SELECT DefaultEwsServiceId FROM Settings`; проверить `EwsServiceSettings`; подробнее — Провайдер Exchange (EWS), раздел «Как определяется список EWS-сервисов»

Запросы для быстрой проверки состояния синхронизации пользователя:
 `declare @user_id int = 0;

-- Состояние пользователя по Exchange
select
        u.UserID,
        u.Email,
        u.DoSyncWithExchange,
        u.CalendarEwsDirectSync,
        u.SyncWithExchangeFailedAttempts,
        u.SID
from dbo.Users u
where u.UserID = @user_id;

-- Служебные флаги Exchange по пользователю
select
        uiev.UserID,
        uie.SysName,
        uiev.Value,
        uiev.IntValue
from dbo.UserInfoExtValues uiev
join dbo.UserInfoExt uie
        on uie.ID = uiev.InfoExtID
where uiev.UserID = @user_id and
      lower(uie.SysName) in (
            'sync.exchange.disablederror',
            'sync.exchange.itemsstate'
      )
order by
        uie.SysName;

-- Кеш папок Exchange для пользователя
select top (200)
        ef.UserID,
        ef.ServiceId,
        ef.ChangeDate,
        ef.CalendarFolderValue,
        ef.InboxFolderValue
from dbo.EwsFolders ef
where ef.UserID = @user_id
order by
        ef.ChangeDate desc;

-- Кеш прав на папки Exchange
select top (200)
        ep.UserID,
        ep.FolderOwnerId,
        ep.ServiceId,
        ep.ChangeDate,
        ep.FolderValue
from dbo.EwsFolderPermissions ep
where ep.UserID = @user_id
order by
        ep.ChangeDate desc;

-- Настройки сервиса Exchange (без пароля)
select
        ess.ServiceId,
        ess.EwsUrl,
        ess.EwsLogin,
        ess.EwsDomain,
        ess.UseImpersonalization,
        ess.UseSIDForImpersonalization
from dbo.EwsServiceSettings ess
order by
        ess.ServiceId;
` Связанные документы:

- Календарь — администрирование

- Провайдер Exchange (EWS)

- Интеграция с Exchange — обзор
