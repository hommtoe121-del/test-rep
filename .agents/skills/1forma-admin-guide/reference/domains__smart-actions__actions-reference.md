# Справочник смарт-действий

Источник: https://help.1forma.ru/domains/smart-actions/actions-reference/

⚠️ Символом * помечены обязательные параметры
 Каталог смарт-действий, доступных для вызова из SmartScript: создание и изменение задач, комментарии, подписчики, email, HTTP-запросы, действия с файлами, отчётами, ЭДО Диадок и СБИС, маршруты, оргструктура, ДП, подписи, чаты и системные операции. Здесь — как вызывать действия из SmartScript (частые действия, примеры), карта разделов справочника, типы данных и каталог событий; описания действий (кодовое имя, числовой ID, таблица параметров) — в разделах по группам, см. § Разделы справочника. Используется при проектировании пакетов действий и интеграционных сценариев.

Все смарт-действия вызываются из Lua/JS-скриптов через `SMART:execute_action` (Lua) или `SMART.execute_action_in_task_context` (JS). Кодовое имя действия (enum `StandardAction`) — первый аргумент вызова.

Кодовые имена и числовые ID самых востребованных смарт-действий:
 Русское название Кодовое имя (StandardAction) Enum ID Отправить комментарий `PostComment` 1 Создать задачу `CreateTask` 0 Добавить подписчика `AddSubscriber` 2 Добавить исполнителя `AddPerformer` 3 Изменить значение ДП `ChangeExtParamValue` 5 Изменить статус задачи `ChangeTaskStateForcibly` 38 Выполнить переход `MakeStep` 4 Отправить email `SendEmail` 17 Отправить HTTP-запрос `SendHttpRequest` 85 Создать файл отчёта `CreateReportFile` 75 Выполнить смарт-скрипт `ExecuteSmartScript` 106

Выполняет HTTP-запрос с автоматической сериализацией JSON, обработкой статус-кодов, повторными попытками и нарастающими паузами. Заменяет ручной сбор URL, headers и парсинг ответа в Lua/JS.
 Параметр Тип Описание Method* `String` HTTP-метод: GET, POST, PUT, PATCH, DELETE Url* `String` Базовый URL запроса Query `Object` Объект query-параметров (автоматически добавляется к URL) Headers `Object` Заголовки запроса (например, `{ "Authorization": "Bearer ..." }`) BodyJson `Object` Тело запроса (автоматически сериализуется в JSON для POST/PUT/PATCH) TimeoutMs `Integer` Таймаут запроса в мс (по умолчанию 15000) ExpectedStatusCodes `Collection<Integer>` Ожидаемые успешные статус-коды (по умолчанию [200]) FailOnUnexpected `Boolean` Если true — неожиданный статус считается ошибкой (по умолчанию true) RetryPolicy `Object` Политика повторных попыток: `{ Retries: 3, BackoffMs: 500, BackoffMultiplier: 2, RetryOnStatuses: [429, 500, 502, 503, 504] }` ResponseMode `String` Формат ответа: `Json` (авто-парсинг) или `Text` Возвращает: `{
  "Ok": true,
  "StatusCode": 200,
  "ResponseHeaders": { "content-type": "application/json" },
  "Json": { "result": "..." },
  "Text": null,
  "Attempts": 1,
  "FinalUrl": "https://api.example.com/resource?a=1&b=text",
  "Error": null
}
`
 Если `FailOnUnexpected=false` и получен неожиданный статус: `{
  "Ok": false,
  "StatusCode": 403,
  "Json": null,
  "Text": "Forbidden",
  "Attempts": 1,
  "Error": {
    "Code": "unexpected_status",
    "Message": "Expected [200], got 403"
  }
}
`
 Пример (Lua): `local r = SMART:execute_action('HttpRequestJson', CONTEXT.Id, 'task', {
  Method = "POST",
  Url = "https://api.example.com/users",
  Headers = { ["Authorization"] = "Bearer token123" },
  BodyJson = { name = "Иван", email = "ivan@example.com" },
  ExpectedStatusCodes = { 201 },
  RetryPolicy = { Retries = 3, BackoffMs = 500, BackoffMultiplier = 2, RetryOnStatuses = { 429, 500 } }
})

if r.Ok then
  SS:log("Пользователь создан: id=" .. r.Json.id)
else
  SS:log("Ошибка: " .. r.Error.Message)
end
`



Обязательные параметры действия — `CommentAuthor`, `CommentText`, `Task`, `ForcedEmail`, `CommentSMS`, `NoSubscription`, `TextAsHTML`. Пометка вопросом называется `MarkAsQuestion` (не `NeedsAnswer` — такого параметра у действия нет).
 Lua: `SMART:execute_action('PostComment', CONTEXT.Id, 'task', {
    CommentAuthor = 3,          -- ОБЯЗАТЕЛЬНО: автор (UserID); 3 — Робот 1Ф
    CommentText = 'Текст комментария',
    Task = CONTEXT.Id,          -- ОБЯЗАТЕЛЬНО, иначе NullRef
    Recipients = {181},         -- адресаты (массив UserID), опционально
    ForcedEmail = false,        -- ОБЯЗАТЕЛЬНО: дублировать на почту
    CommentSMS = false,         -- ОБЯЗАТЕЛЬНО: дублировать по SMS
    NoSubscription = true,      -- ОБЯЗАТЕЛЬНО: не подписывать адресатов на задачу
    TextAsHTML = false,         -- ОБЯЗАТЕЛЬНО: трактовать текст как HTML
    MarkAsQuestion = false      -- пометить вопросом, опционально
})
`
 JavaScript: `SMART.execute_action_in_task_context("PostComment", taskId, {
    CommentAuthor: 3,           // ОБЯЗАТЕЛЬНО: автор (UserID); 3 — Робот 1Ф
    CommentText: "Текст комментария",
    Task: taskId,               // ОБЯЗАТЕЛЬНО, иначе NullRef
    Recipients: [181],          // адресаты (массив UserID), опционально
    ForcedEmail: false,         // ОБЯЗАТЕЛЬНО: дублировать на почту
    CommentSMS: false,          // ОБЯЗАТЕЛЬНО: дублировать по SMS
    NoSubscription: true,       // ОБЯЗАТЕЛЬНО: не подписывать адресатов на задачу
    TextAsHTML: false,          // ОБЯЗАТЕЛЬНО: трактовать текст как HTML
    MarkAsQuestion: false       // пометить вопросом, опционально
});
`
 Важно: параметр `Task` обязателен даже при использовании `execute_action_in_task_context`. Без него — `NullReferenceException`.
 `CommentAuthor` обязателен и в событийных сценариях (пакет на событии, анонимная публикация): без него `SessionUserId = -1` → `NullReferenceException` → HTTP 500.
 Полный перечень параметров с типами — в справочнике действия.

Lua (текст задачи — `TaskText`, не `Text`; связи — `CreateLink`/`CreateSubtask`, не `MakeLinked`/`MakeSubTask` — таких параметров у действия нет): `SMART:execute_action('CreateTask', CONTEXT.Id, 'task', {
    Subcat = 5641,              -- ОБЯЗАТЕЛЬНО: ID категории (пример SubcatID, тип параметра: 1F.Subcat)
    Owner = 28,                 -- ОБЯЗАТЕЛЬНО: заказчик (UserID)
    TaskText = 'Текст задачи',
    CreateLink = true,          -- ОБЯЗАТЕЛЬНО: сделать связанной с текущей
    CreateSubtask = false,      -- ОБЯЗАТЕЛЬНО: сделать подзадачей
    NewTaskCopySubscribers = false,  -- ОБЯЗАТЕЛЬНО: копировать подписчиков из родительской
    CreateCopyFiles = false,    -- ОБЯЗАТЕЛЬНО: копировать вложения
    CopyParentText = false,     -- ОБЯЗАТЕЛЬНО: включать текст исходной
    CreateCopiesForEachPerformer = false,  -- ОБЯЗАТЕЛЬНО: каждому исполнителю отдельную копию
    Priority = 1                -- ОБЯЗАТЕЛЬНО: 0 — низкий, 1 — обычный, 3 — высокий
})
`

Lua (доп. параметр — `ExtParam`, не `ExtParamId` — такого параметра у действия нет): `SMART:execute_action('ChangeExtParamValue', CONTEXT.Id, 'task', {
    Task = CONTEXT.Id,          -- ОБЯЗАТЕЛЬНО: задача
    User = SESSION_USER.Id,     -- ОБЯЗАТЕЛЬНО: от чьего имени меняется ДП
    ExtParam = 12408,           -- ОБЯЗАТЕЛЬНО: ID доп. параметра
    Value = 'новое значение',
    WriteCommentOnChange = false -- ОБЯЗАТЕЛЬНО: писать комментарий о смене
})
`

Lua: `SMART:execute_action('ChangeTaskStateForcibly', CONTEXT.Id, 'task', {
    Task = CONTEXT.Id,
    Initiator = SESSION_USER.Id,
    State = 3                   -- ID целевого статуса
})
`

Последний аргумент `true` — выполнение в фоне (не блокирует скрипт):
 `SMART:execute_action('PostComment', CONTEXT.Id, 'task', {
    CommentAuthor = 3,
    CommentText = 'Фоновый комментарий',
    Task = CONTEXT.Id,
    ForcedEmail = false,
    CommentSMS = false,
    NoSubscription = true,
    TextAsHTML = false
}, true)
`

Полный список действий (~215, enum `StandardAction` ID 0–215) доступен в редакторе SmartScript → вкладка «Смарт-действия».

Смежные разделы:

- Смарт-действия — администрирование — вызов `SMART:execute_action`, контекст, примеры

- Паттерны и примеры

Полный справочник действий (каждое — с кодовым именем, ID и таблицей параметров) разбит по группам:
 Раздел Группы действий Задачи, связи, маршруты Категории, задачи, связи · Маршруты, переходы Оповещения, чаты, комментарии Оповещения · Чаты, комментарии, чек-листы Участники, оргструктура, группы Заказчики, исполнители, подписчики · Оргструктура, группы, пользователи ЭДО и подписи Работа с операторами ЭДО (Диадок, СБИС, Контур.КЭДО) · Подписи Файлы, отчёты, данные, сроки, ДП Файлы и отчёты · Данные · Сроки и трудозатраты · ДП — модификация значений · Файлы (расширение) Системные действия HTTP, SQL, SmartScript, события, повторения

Типы параметров, используемые в таблицах параметров разделов справочника:
 Тип Описание `1F.Task` Задача в системе `1F.User` Пользователь `1F.UserGroup` Группа пользователей `1F.Subcat` Категория `1F.State` Статус задачи `1F.Step` Переход по маршруту `1F.File` Файл `1F.CustomerZone` Личный кабинет `Collection<T>` Коллекция объектов типа T `String` Строка `Integer` Целое число `Boolean` Логическое значение `DateTime` Дата и время `Money` Денежная сумма

Полный реестр событий (197) с EventID, именем, группой и контекстными параметрами `@eventParam*` доступен в редакторе SmartScript. Используется для проектирования пакетов действий: на какое событие подписать пакет и какие данные доступны в контексте.
 Покрытие:

- Всего событий: 197 (значения 0–196).

- С описанием параметров контекста: 127.

- Без задокументированного контекста: 69.
  Распределение по группам:
 Группа Событий Не классифицировано 69 Задачи и комментарии 30 Подписи 22 Ресурсное планирование 15 Сроки и даты 12 Связи и подзадачи 12 ДП и свойства задачи 9 Пользователи 8 Глобальные 7 Файлы 6 Другое 3 Exchange 3 Ключевые правила:

- События `Перед ...` отменяемы, если в пакете есть действие `Отменить`.

- Группы `Exchange`, `Глобальные`, `Пользователи` работают как глобальные (используются в «Общих SMART»).

- Флаг «может быть глобальным» (`CanBeGlobal`): такое категорийное событие может срабатывать без привязки к категории (SubcatID=NULL). Первое такое событие — `AfterPostComment` (29).

- Для событий ресурсного планирования параметр `Дата` приходит массивом — в SQL читается через `OPENJSON`.

- Нумерация в C# enum нелинейная: `AfterSignatureSigned=70` объявлен после `BeforeSubscriberAdded=58`. В реестре — точные числовые значения.
