# Jint — JS-интерпретатор для смарт-скриптов

Источник: https://help.1forma.ru/domains/smart-actions/js-scripting-jint/

Справочник по JavaScript-смарт-скриптам на движке Jint 4.10.1 в платформе 1Форма. Документ для разработчиков смарт-скриптов и интеграторов: статус, ограничения движка, глобальные API-объекты (SQL, UTILS, SMART, HTTP и др.), META API и сравнение с Lua. Все смарт-действия и их параметры — в Справочнике смарт-действий.

Реализовано. JS-скриптинг доступен в продакшене как альтернатива Lua.
 Платформа поддерживает пять языков смарт-скриптов:
 Язык Движок `ScriptLanguage` (enum) `LanguageId` (БД) Документация Lua NLua `Lua = 0` `0` — JavaScript Jint 4.10.1 `JavaScript = 1` `1` этот файл Python Внешний сервис (Docker) `Python = 2` `2` python-scripting.md OneScript OneScript `OneScript = 3` `3` — C# Roslyn `CSharp = 4` `4` csharp-scripting-roslyn.md C# — добавлен 2026-04-20. Нативный async/await, прямой доступ к сервисам платформы.
 Jint — pure .NET JS-движок без внешних зависимостей. Не требует V8 или Node.js.

Каждый JS SmartScript обязан содержать комментарий с версией и датой в начале скрипта:
 `// v1 | 2026-03-08 15:30 | Начальная версия
// v2 | 2026-03-08 16:45 | Добавлена проверка прав
` Версия инкрементируется при каждом изменении. Подробнее — в Паттернах JS-скриптов (§0).
 Движок поддерживает два режима выполнения:
 Параметр Синхронный Асинхронный `await` для C#-действий из JS ❌ ✅ Подключение ES-модулей ❌ ✅ Таймаут промисов — 5 минут Обёртка скрипта напрямую `(async () => { <script> });`

Движок JavaScript работает в изолированном окружении с фиксированными ограничениями (не настраиваются извне):
 Параметр Значение Назначение `TimeoutInterval` 5 минут Защита от бесконечных циклов `MaxStatements` 10 000 000 Защита от вычислительных бомб (увеличено с 1M до 10M в 2.268.66) `ExceptionHandler = _ => true` все CLR-исключения перехватываются CLR-исключения пробрасываются как JS `Error` Лимиты `TimeoutInterval` и `MaxStatements` применяются в том числе внутри тяжёлых операций (`JSON.stringify`, `Array.join`, сортировка больших массивов): при превышении такая операция прерывается, а не выполняется целиком сверх таймаута.
 CLR-типы по умолчанию недоступны — только то, что явно зарегистрировано через `SetValue`.



Переменные, доступные в любом скрипте:
 Переменная Тип Значение `CONTEXT` object Контекст (задача, пользователь, письмо и т.д.) `EVENTPARAMS` object Параметры события (если скрипт привязан к событию). ⚠️ Доступен только на верхнем уровне; внутри `try-catch` теряет scope. Сохраняй в переменную ДО `try`. Для публикаций: `EVENTPARAMS["PublishedObjectParameters"]` содержит `{requestBody, headers}` (вложенная структура, НЕ плоская). Headers в mixed-case (`x-Gitlab-Token`). Требует eventId=106 (OnCallPublishedObject) в SmartScript — без него `EVENTPARAMS` undefined `SESSION_USER` object Текущий пользователь `DB_TYPE` string `"MSSQL"` или `"PG"` `SYSTEM_INFO` object `{ version, app }` `APP_PATH` string Путь к директории приложения `CLIENT` object `{ appType, appVersion }` — тип клиента `RESULT` any Возвращаемое значение — движок читает `RESULT` после выполнения Паритет языков. Ключ `CONTEXT` из `inputParams` публикуется как глобал во всех 5 движках SmartScript. В JS, Lua и C# (Roslyn) это материализованная бизнес-сущность с доступом к свойствам (`CONTEXT.Id` и т.д.) — та же семантика, что и здесь. В Python и OneScript тот же `CONTEXT` проходит через универсальную сериализацию `inputParams` и теряет доступ к свойствам (в Python сущность сводится к её `Id`, в OneScript — к строковому представлению). Подробное описание различения `CONTEXT` (бизнес) и `Context` (execution) для C# — в csharp-scripting-roslyn.md.

Объекты API для работы с системой:
 Объект Описание `SQL` SQL-запросы к БД `UTILS` Утилиты (строки, даты, JSON, XML, шифрование) `SMART` Выполнение смарт-действий `HTTP` HTTP-запросы (включая multipart) `CACHE` Кэш скриптов в памяти `REGISTRY` Реестры (чтение/запись записей) `FILES` Чтение файлов `META` / `МЕТА` Доступ к конфигурационным сущностям по именам `VECTORDB` Работа с векторной БД (опциональный, устаревший) `NLP` Склонение, числа прописью, конвертация форматов, спеллчек, раскладка (см. nlp-api.md) `include(id)` Подключение библиотечных скриптов

Глобальные вспомогательные функции:
 `print(str)                         // вывод в output
var_dump(val)                      // JSON-дамп значения в output
is_empty(val)                      // true если null/undefined/""/{}/[]
ts2date(timestamp, utc)            // Unix timestamp → "yyyy-MM-dd"
ts2datetime(timestamp, utc)        // Unix timestamp → "yyyy-MM-dd HH:mm:ss"
string_starts_with(str, prefix)    // аналог str.startsWith
string_ends_with(str, suffix)      // аналог str.endsWith
table_has_value(obj, value)        // поиск значения в объекте
encrypt(str)                       // шифрование строки
decrypt(str)                       // расшифровка строки
`

Методы для SQL-запросов:
 `SQL.scalar(query, params)     // → scalar value
SQL.query(sql, params)        // → .NET List<Dictionary> (все строки)
SQL.query_one(sql, params)    // → первая строка или null — ⚠️ см. ниже
` ⚠️ Это полный список SQL-методов. `SQL.exec` / `SQL.execute` не существует. DML (INSERT/UPDATE/DELETE) — через `SQL.scalar` с возвратом значения или `SQL.query`.
 Параметры — JS-объект, ключи передаются в запрос как именованные параметры.
 ⚠️ `SQL.query_one` не работает в контексте публикаций (eventId=106, OnCallPublishedObject) — возвращает `null` даже для существующих строк. Используй `SQL.query` + `rows[0]`:
 `var rows = SQL.query("SELECT " + topN(1) + "... WHERE TaskID = " + parseInt(id) + limitN(1));
var task = rows.Count > 0 ? rows[0] : null;  // .Count — свойство .NET List
` ⚠️ Весь SQL в SmartScripts обязан быть DB-агностичным (MSSQL + PG). Глобальная переменная `DB_TYPE` = `"MSSQL"` или `"PG"`. Запрещено: `[Key]`, `WITH(NOLOCK)`, `TOP N`, `GETDATE`, `LEN`, `ISNULL` в прямом виде. Обязательно использовать хелперы `qi`, `topN`, `NL` и т.д. — полный набор и примеры в js-jint-patterns.md §0.1.

Утилиты: строки, даты, JSON, XML, шифрование, секреты:
 `UTILS.print(str)
UTILS.json_encode(val)         // → JSON string
UTILS.json_decode(str)         // → object
UTILS.base64_encode(str)           // с 2.268.58 — также доступна как глобальная base64_encode(str)
UTILS.base64_decode(str)           // с 2.268.58 — также доступна как глобальная base64_decode(str)
UTILS.xml_parse(xml)           // XML → object
UTILS.xml_serialize(obj)       // object → XML
UTILS.xml_to_json(xml)
UTILS.json_to_xml(json)
UTILS.encrypt(str)
UTILS.decrypt(str)
UTILS.resolve_instance(asm, type, ctorParams)  // DI resolve
UTILS.create_instance(asm, type, ctorParams)   // Activator.CreateInstance
UTILS.open_stream(data)        // byte[] → MemoryStream
UTILS.getsecretvalue(serviceKey, fieldName)    // → string | null (из IntegrationSecrets)
UTILS.getsecretpayload(serviceKey)             // → object | null (весь payload секрета)
UTILS.set_secret_value(serviceKey, fieldName, value)  // upsert одного поля payload (с 2.268)
UTILS.set_secret_payload(serviceKey, payload)         // upsert всего payload целиком (с 2.268)
UTILS.sha256_hash(text)                        // → Base64 SHA-256 хеш строки (с 2.268)
UTILS.count_tokens(text, encoding?)            // → int, BPE token count (с 2.268)
UTILS.trim_to_tokens(text, maxTokens, encoding?) // → string, обрезать до N токенов (с 2.268)
`

В серверных смарт-скриптах можно читать секреты интеграций из централизованного хранилища `IntegrationSecrets`.
 Метод JavaScript Lua Одно значение `UTILS.getsecretvalue(serviceKey, fieldName)` `UTILS:getsecretvalue(serviceKey, fieldName)` Весь набор `UTILS.getsecretpayload(serviceKey)` `UTILS:getsecretpayload(serviceKey)`
- `serviceKey` — ключ секрета в хранилище интеграционных секретов.

- `fieldName` — имя поля внутри payload секрета.
  Пример JavaScript:
 `var apiKey = UTILS.getsecretvalue("dadata-5", "ApiKey");
var creds = UTILS.getsecretpayload("sbis-3");
` Пример Lua:
 `local apiKey = UTILS:getsecretvalue("dadata-5", "ApiKey")
local creds = UTILS:getsecretpayload("sbis-3")
` Поведение при ошибках: если секрет или поле не найдены — возвращается `null`. HTTP 500 не возникает.
 Аудит: каждое обращение фиксируется в `IntegrationSecretsAuditLog` с источником `smartScript`. Расшифрованные значения секретов в логи не попадают.
 Ограничения: методы работают только с новым хранилищем `IntegrationSecrets` — не предназначены для чтения legacy-секретов из старых таблиц настроек сервисов.

С 2.268 секреты можно обновлять из SmartScript. Нужно, например, для rotating-токенов (refresh token Passwork, OAuth, …) — SS, запускаемый по расписанию, получает новый токен и сохраняет его обратно в `IntegrationSecrets`.
 Метод JavaScript Lua Одно поле (upsert) `UTILS.set_secret_value(serviceKey, fieldName, value)` `UTILS:set_secret_value(serviceKey, fieldName, value)` Весь payload (upsert) `UTILS.set_secret_payload(serviceKey, payload)` `UTILS:set_secret_payload(serviceKey, payload)`
- `set_secret_value` — читает текущий payload, подменяет одно поле, сохраняет результат. Остальные поля сохраняются.

- `set_secret_payload` — полностью заменяет payload. Поля, не указанные в `payload`, удаляются.

- Если секрет с таким `serviceKey` не существует — создаётся новый. `DisplayName` берётся из существующей записи или равен `serviceKey` для новых.

- Значения payload всегда сохраняются как строки.
  Пример JavaScript (refresh-токен):
 `var newToken = UTILS.getsecretvalue("passwork-1", "refresh_token");
// ... обмен на новую пару ...
UTILS.set_secret_value("passwork-1", "access_token", resp.access_token);
UTILS.set_secret_value("passwork-1", "refresh_token", resp.refresh_token);

// Или одним вызовом:
UTILS.set_secret_payload("passwork-1", {
    access_token: resp.access_token,
    refresh_token: resp.refresh_token,
    expires_at: resp.expires_at
});
` Пример Lua:
 `UTILS:set_secret_value("passwork-1", "access_token", resp.access_token)

UTILS:set_secret_payload("passwork-1", {
    access_token = resp.access_token,
    refresh_token = resp.refresh_token,
    expires_at = resp.expires_at
})
` Аудит: запись фиксируется в `IntegrationSecretsAuditLog` с источником `smartScript` и `UserId` текущего пользователя — так же, как при чтении. Сами значения секретов в лог не попадают.

`sha256_hash(text)` → string — возвращает SHA-256 хеш строки в кодировке Base64.
 `var hash = UTILS.sha256_hash("текст промпта");
// → "x3Xnt1ft5jDNCqERO9ECZhqziCnKUqZCKreChi8mhkY="
` Пустая строка и `null` возвращают пустую строку.
 `count_tokens(text, encoding?)` → int — подсчитывает количество BPE-токенов в тексте. Используется для оценки размера контекста при работе с LLM.
 Параметр Тип По умолчанию Описание `text` string — Текст для подсчёта `encoding` string `"cl100k_base"` BPE-кодировка Поддерживаемые кодировки:
 Кодировка Модели `cl100k_base` Claude (все версии), GPT-4, GPT-4-turbo — по умолчанию `o200k_base` GPT-4o, GPT-4o-mini `var n = UTILS.count_tokens("Привет, как дела?");            // cl100k_base (по умолчанию)
var n = UTILS.count_tokens(text, "o200k_base");              // для GPT-4o
` Пустая строка и `null` возвращают `0`.
 `trim_to_tokens(text, maxTokens, encoding?)` → string — обрезает текст до указанного количества токенов. Возвращает подстроку, гарантированно не превышающую лимит.
 Параметр Тип По умолчанию Описание `text` string — Исходный текст `maxTokens` int — Максимальное число токенов `encoding` string `"cl100k_base"` BPE-кодировка `var trimmed = UTILS.trim_to_tokens(longText, 100000);  // обрезать до 100K токенов
` Если текст уже укладывается в лимит — возвращается без изменений.

Выполнение смарт-действий и запуск других скриптов:
 `SMART.execute_action(action, contextId, contextType, params, async)
SMART.execute_action_in_task_context(action, taskId, params, async)
SMART.execute_action_in_user_context(action, userId, params, async)
SMART.execute_action_in_email_context(action, emailId, params, async)
SMART.run_script_background(scriptId, contextId, eventParams, extraParams, ignoreScriptEvent)
SMART.run_script_sync(scriptIdOrName, contextId, contextType, eventParams, extraParams, ignoreScriptEvent)
` `action` — имя из `StandardAction` (enum). `async = true` — выполнение в фоне.
 `run_script_background` — асинхронный запуск смарт-скрипта по ID в фоновой задаче. `contextId` и `eventParams` — опциональны (можно `null`). `extraParams` — JS-объект с произвольными ключами, пробрасывается в целевой скрипт как `extraInputParams`. Возврата RESULT нет (fire-and-forget). Подробнее: Смарт-скрипты.
 `run_script_sync` — синхронный запуск с возвратом RESULT (добавлен 2026-04-21). Принимает id (number) или name (string). `contextType` — строка `"task"|"user"|"email"|"edocumentlink"|"edocumentlinksbis"`, default `"task"`. Блокирующий вызов; целевой скрипт выполняется под теми же правами, что и вызывающий.
 Глубина каскада ограничена 3 уровнями. На 4-м уровне выполнение прерывается с ошибкой «Превышена максимальная глубина синхронных вызовов SmartScript (3)».
 Когда что использовать: | Нужно | Метод | |-------|-------| | Выполнить SS в фоне, результат не нужен | `run_script_background` | | Получить RESULT синхронно, передать extraParams | `run_script_sync` | | Выполнить в том же scope (shared globals) | `include` — но без extraParams и с потерей Promise из async-скриптов | | Выполнить action (не SmartScript) | `execute_action` |
 ⚠️ PostComment в анонимных публикациях: `CommentAuthor` обязателен (без него SessionUserId=-1 → 500). Используй `CommentAuthor: 3` (Робот 1Ф) + `ForcedEmail: false, CommentSMS: false, NoSubscription: true`.
 ⚠️ Обновление ДП: действие `ChangeExtParamValue` (не `UpdateExtParam`). Параметры: `Task`, `User`, `ExtParam` (ID), `Value` (строка), `WriteCommentOnChange`.

Модули HTTP, CACHE, REGISTRY, FILES, LIB и устаревший VECTORDB предоставляют сетевые запросы, кэш, реестры и файлы из JS-скрипта.
 HTTP-запросы из скрипта:
 `HTTP.send_http_request(method, url, params, headers, rawBody, options)
HTTP.post_files(url, files, params, headers, options, preuploaded)
HTTP.post_multipart(url, parts, params, headers, options, preuploaded)
` Возвращает десериализованный JSON-ответ.
 Работа с общим кэшем скриптов:
 `CACHE.set(key, value, lifetime)  // lifetime в секундах, default = MaxInt
CACHE.get(key)                   // → string или null
CACHE.delete(key)
` Кэш общий с Lua-скриптами.
 Работа с записями реестров:
 `REGISTRY.record_add(registryId, values, overwrite)   // → recordId
REGISTRY.record_delete(registryId, recordId)
REGISTRY.record_delete_by_key(registryId, keys)
REGISTRY.record_get(registryId, recordId)            // → object
REGISTRY.record_find(registryId, keys, strict)       // → array
` Чтение содержимого файлов:
 `FILES.get_file_content(fileId, versionId)                         // → byte[]
FILES.get_file_content_string(fileId, versionId, encoding)        // → string
` Подключение библиотечных скриптов:
 `include(scriptId)     // number: by ID
include("ScriptName") // string: by name or parseable ID
` Глубина рекурсии включений — не более 5 уровней. Подключать можно только скрипты с `IsLibrary = true`.
 Устаревший модуль VECTORDB (с 2026-05-08). В новых скриптах не используется; раздел сохранён для совместимости со старыми скриптами.
 Работа с векторной базой данных. Опциональный модуль — доступен не на всех площадках.

Глобальные функции для логирования и streaming-вывода (Анфиса):
 `log(message)              // запись в ExecutionLogger (виден в редакторе при тестовом запуске)
log_step(name, data)      // именованный шаг логирования
stream_write(type, data)  // streaming-вывод — токенизированный поток для Анфисы
is_cancelled()            // проверка отмены (CancellationToken) → boolean
` `stream_write` отправляет токенизированный поток (используется Анфисой при потоковых ответах). `is_cancelled` возвращает `true`, если пользователь отменил выполнение, — позволяет скрипту корректно завершиться.

`МЕТА` (alias `META`) — глобальный объект SmartScript для получения конфигурационных сущностей по именам и путям без жёстко заданных ID.
 Доступен под именами `META` (англ.) и `МЕТА` (рус.) — оба идентичны.

Dotted path (строка с точками):
 `МЕТА.Категория("Продажи.Клиенты")        // → Категория proxy
МЕТА.ПолучитьДП("Клиенты.Заказчик")      // → ДП proxy
МЕТА.Состояние("Клиенты.Новый")          // → Состояние proxy
МЕТА.Шаг("Клиенты.Новый->В работе")     // → Шаг proxy
МЕТА.Подпись("Клиенты.Согласование")     // → Подпись proxy
МЕТА.Событие("Клиенты.ClientCreated")    // → Событие proxy
МЕТА.КолонкаДП("Клиенты.Заказы.Кол-во") // → Колонка proxy (мин. 3 сегмента)
` Fluent chain (цепочка типизированных объектов):
 `МЕТА.Раздел("Продажи").Категория("Клиенты").ПолучитьДП("Заказчик")
МЕТА.Раздел("Продажи").Категория("Клиенты").Состояние("Новый")
`

Методы доступа к конфигурационным сущностям:
 Метод Поддержка dotted path Мин. сегментов `МЕТА.Категория(path)` да 1 (уникальное имя) или `Раздел.Кат` `МЕТА.ПолучитьДП(path)` да 2 (`Кат.ДП`) `МЕТА.Состояние(path)` да 2 (`Кат.Состояние`) `МЕТА.Шаг(path)` да `Кат.Откуда->Куда` (мин. 3 сегмента с `->`) `МЕТА.Подпись(path)` да 2 `МЕТА.Событие(path)` да 2 `МЕТА.КолонкаДП(path)` да 3 (`Кат.ДП.Колонка`) `МЕТА.Раздел(name)` fluent entry — `МЕТА.Группа(name)` нет — `МЕТА.Модуль(name)` нет — `МЕТА.Отчёт(name)` нет — `МЕТА.Дашборд(name)` нет — Получение информации о сущностях (интроспекция):
 `МЕТА.СписокДП("Клиенты")       // → ДП[] всех ДП категории
МЕТА.ТипДП("Клиенты.Заказчик") // → string ("Lookup", "String", "Table", ...)
МЕТА.Справочник("Клиенты")     // → bool
`

Правила записи путей к сущностям:

- Greedy-right парсинг: при нескольких точках платформа ищет максимально длинный категорийный префикс. `Продажи.Клиенты.Заказчик` → subcategory=`Продажи.Клиенты`, name=`Заказчик`.

- Мин. 2 сегмента для ДП, Состояния, Подписи, События (`Категория.Имя`).

- Мин. 3 сегмента для КолонкаДП (`Категория.ДПТаблица.Колонка`).

- Мин. 3 сегмента для Шага: формат `Категория.Откуда->Куда` (стрелка без пробелов).

- Неоднозначность имени категории — ошибка времени выполнения. При конфликте имён уточняй через раздел: `МЕТА.Раздел("Продажи").Категория("Клиенты")`. Fluent-нотация безопаснее короткого dotted path при неуникальных именах.

- При ошибке резолвинга (не найдено / неоднозначно) скрипт получает `ArgumentException`.

Соответствие типов JavaScript и платформы:
 JS-тип CLR-тип `null` / `undefined` `null` `boolean` `bool` Целое число `int` Дробное `double` `string` `string` Array `List<object>` Object (plain) `Dictionary<string, object>` CLR-объект (ObjectWrapper) исходный CLR-объект (сохраняется тип) Jint автоматически оборачивает CLR-объекты, переданные через `SetValue`. При конвертации обратно `ObjectWrapper.Target` возвращает оригинальный CLR-объект (важно для `CONTEXT`, `SESSION_USER`).
 Имена таблиц и колонок, используемые в SQL-запросах из скриптов:
 `-- Справочник языков
SELECT * FROM SmartScriptLanguages;
-- 0 = Lua, 1 = JavaScript, 2 = Python

-- Поле в SmartScripts
SELECT Id, Description, LanguageId FROM SmartScripts;
` Маппинг: `SmartScriptEntity.LanguageId` (тип `ScriptLanguage`) ↔ `SmartScriptDto.Language`.

Ключевые отличия JavaScript-скриптов от Lua:
 Аспект Lua (NLua) JavaScript (Jint) Синтаксис вызова API `SQL:query(sql)` (двоеточие) `SQL.query(sql)` (точка) Переменные `local x = ..` `const x = ..` / `let x = ..` Конкатенация `"a" . "b"` ``a${b}`` / `"a" + "b"` Null `nil` `null` / `undefined` Sandbox Ограниченный (LoadCLRPackage даёт доступ к CLR) Строгий (только явно зарегистрированные объекты) Таймаут 5 мин 5 мин Конвертер результата `LuaTypesConverter` `SmartScriptTypesConverter` Ошибки .NET 9 SEHException вместо managed exceptions Нет (pure .NET)

Смарт-скрипты выполняются в следующих контекстах:
 Контекст Sync/Async Смарт-действия (пакеты) sync Действие «Выполнить скрипт» (StandardAction = 106) sync/async Шаблоны печати (Word/PDF) sync Мобильная лента sync Видеоконференции (Jitsi) sync AI-инструменты sync Портальные виджеты (источник данных) async Редактор скриптов (тестовый запуск) async AI-агент Анфиса (потоковый вывод) async + потоковый вывод Язык скрипта определяет движок автоматически — для вызывающего это прозрачно.
 RESULT — глобальная переменная. Скрипт присваивает `RESULT = <значение>` (без `var`/`let`/`const`, чтобы переменная была глобальной). Движок читает `RESULT` после выполнения скрипта.
 Смежные разделы:

- Паттерны JS-скриптов — рецепты, хелперы DB-агностичного SQL

- FAQ: обработка ошибок Lua и pcall

- Справочник смарт-действий

Редактор поддерживает три языка: Lua, JavaScript, Python. Для JavaScript работает автодополнение `ДП.*` (список параметров задачи) и подсказки при наведении на методы ДП (`Значение`, `ТекстовоеЗначение`, `ИзменитьЗначение`, `Сохранить`, `Обновить`, `Скрыть`, `Заблокировать` и др.). Для Python и Lua автодополнение пока не реализовано.
 В редакторе также доступно автодополнение META-выражений — подсказки по разделам, категориям, ДП, lookup-полям, табличным ДП и их колонкам.
