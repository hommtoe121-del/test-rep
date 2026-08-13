# 1С не подключается к 1Ф / ошибка аутентификации

Источник: https://help.1forma.ru/domains/integrations/runbook-1c-connect-auth/

Руководство по разбору инцидентов обмена 1С и 1Формы: признаки сбоя, архитектура двунаправленного обмена, чек-лист диагностики, таблица «симптом → причина», SQL-снимки и список артефактов для обращения в поддержку. Охватывает направления 1С→1Ф и 1Ф→1С, очередь повторов и сопоставление пользователей.

Признаки инцидента:

- В админке не проходит `Проверить соединение` для конфигурации 1С.

- При обмене появляются ошибки авторизации/доступа к веб-сервису 1С.

- События из 1С не доходят до 1Ф или «застревают» в очереди.

- После обновления 1С обмен перестал работать.
  Симптом Вероятная причина Что проверить первым `config/test-connection` падает сразу неверный `OneCAddress`/тип учётных данных XML в `SyncSettings1C`, доступность адреса Ошибка авторизации к 1С неверные `OneCUserName`/пароль/режим авторизации настройки конфига + права пользователя сервиса 1С `send-1c-event` даёт ошибку `NoSuchSettings`/`currRecord is null` неверное `ConfigurationName` или `OneCDocumentType` имя конфига и наличие record в XML События копятся и не обрабатываются проблема потока очереди/повторов `Sync1CEventQueues`, `RetryCount`, `Sync1CQueueMaxRetryCount` В логах есть входящие события, но данные не применяются проблема сопоставления/прав/валидации реквизитов `Sync1CUsersMap`, текст ошибок в `Sync1CLog`

Обмен 1С и 1Формы реализован в двух направлениях и через отдельные механизмы проверки, обработки событий и повторов.
 `sequenceDiagram
    participant C as 1С
    participant F as 1Ф
    participant Q as Очередь Sync1CEventQueues
    participant L as Sync1CLog
    Note over C,F: входящее событие, 1С в 1Ф
    C->>F: send-1c-event (TC1CService.asmx / api sync1c)
    F->>F: поиск конфига и Record по ConfigurationName + OneCDocumentType
    alt задан поток очереди InboxQueueFlowId
        F->>Q: поставить событие в очередь
        Q->>Q: EventQueue1CJob, повтор до Sync1CQueueMaxRetryCount
        Q->>F: обработать событие
    else очередь не задана
        F->>F: обработать синхронно
    end
    F->>L: записать результат или ошибку
    Note over F,C: исходящий вызов, 1Ф в 1С
    F->>C: SOAP к OneCAddress (test-connection, отправка настроек)
    C-->>F: ответ
    F->>L: зафиксировать ошибки` 2.1 Два разных направления обмена

- 1С → 1Ф:

- 1С вызывает маршрут в 1Ф (`/TC1CService.asmx` и/или `api/sync1c/send-1c-event`),

- событие обрабатывается на стороне 1Ф.

- 1Ф → 1С:

- 1Ф создаёт SOAP-клиент к адресу `OneCAddress`,

- операции: тест подключения, отправка настроек, первичная загрузка и прочие вызовы.
  2.2 Проверка подключения в админке
 `POST /api/admin/sync1c/config/test-connection?name=...`:

- Берёт настройки из `SyncSettings1C`.

- Создаёт SOAP client с учётом:

- `OneCAddress`,

- `OneCUserName`/`OneCPassword`,

- `OneCCredentialType`,

- режима безопасности (`http/https` binding).

- Выполняет тест подключения.

- Ошибки фиксируются в `Sync1CLog`.
  2.3 Обработка входящего события из 1С
 Обработка идёт по шагам:

- Ищет конфиг и `Record` по `ConfigurationName` + `OneCDocumentType`.

- Если задан поток очереди (`InboxQueueFlowId`), событие ставится в очередь.

- Иначе выполняется синхронно.

- Логирование идёт в `Sync1CLog`.
  2.4 Отдельная очередь повторов по 1С-событиям
 Для части сценариев используется `Sync1CEventQueues`:

- Лимит повторов задаётся `Sync1CQueueMaxRetryCount` (`0` = без лимита).

- Обработка: `api/admin/sync1c/queue/process*` и фоновая задача `EventQueue1CJob`.

- Очистка проблемных записей: `clear-by-retry-count`.

При разборе инцидента пройдите по шагам:

- Уточнить направление проблемы:

- 1С не может достучаться до 1Ф,

- или 1Ф не может авторизоваться в 1С.

- Если это 1Ф → 1С:

- вызвать `config/test-connection`,

- проверить `OneCAddress`, тип учётных данных и логин.

- Если это 1С → 1Ф:

- проверить, что `/TC1CService.asmx` без учётных данных отвечает отказом (401), а с учётными данными интеграции отдаёт WSDL / принимает SOAP; при необходимости — `api/sync1c/test-connection`,

- проверить, что событие реально приходит в `Sync1CLog`.

- Проверить актуальный XML-конфиг в `SyncSettings1C` (имя, адреса, тип учётных данных, поток очереди).

- Проверить сопоставление пользователя 1С (`Sync1CUsersMap`) — при отсутствии сопоставления используется системный пользователь.

- Проверить очередь/повторы:

- рост `RetryCount`,

- записи с одинаковым объектом и повторными ошибками.

- При недавнем обновлении 1С отдельно проверить перепубликацию веб-сервиса и актуальность URL/WSDL.

Запросы для быстрой проверки состояния обмена с 1С:
 `declare @config_name varchar(100) = '';
declare @user_1c_login varchar(256) = '';

-- Конфиги синхронизации
select
        s.Id,
        s.Name,
        len(s.XmlContent) as XmlSize
from dbo.SyncSettings1C s
where @config_name = '' or
      s.Name = @config_name
order by
        s.Name;

-- Последние логи синхронизации
select top (200)
        l.Id,
        l.[Date],
        l.SettingsName,
        l.Origin,
        l.[Message],
        l.TechData,
        l.Server1F,
        l.Server1C,
        l.Duration
from dbo.Sync1CLog l
where @config_name = '' or
      l.SettingsName = @config_name
order by
        l.[Date] desc;

-- Очередь событий 1С + ретраи
select top (200)
        q.Id,
        q.SettingsName,
        q.EventType,
        q.DateAdded,
        q.DateOfLastTry,
        q.RetryCount,
        q.TaskID,
        q.SubcatID,
        q.AdditionalID,
        q.ExtParamID,
        q.UserID
from dbo.Sync1CEventQueues q
where @config_name = '' or
      q.SettingsName = @config_name
order by
        q.DateAdded desc;

-- Mapping логинов 1С -> user 1Ф
select
        m.User1CLogin,
        m.UserID
from dbo.Sync1CUsersMap m
where @user_1c_login = '' or
      lower(m.User1CLogin) = lower(@user_1c_login)
order by
        m.User1CLogin;
` Что приложить при обращении в поддержку 1Ф:

- Направление сбоя: `1С → 1Ф` или `1Ф → 1С`.

- Имя конфига (`ConfigurationName`) и точный текст ошибки.

- Результат `config/test-connection` (если запускали).

- SQL-снимок по `SyncSettings1C`, `Sync1CLog`, `Sync1CEventQueues`, `Sync1CUsersMap`.

- Было ли недавнее обновление/перепубликация 1С веб-сервиса.
  Связанные документы:

- Интеграции — администрирование

- Сопоставление сущностей 1С
