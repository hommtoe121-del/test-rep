# JS/Jint SmartScripts — справочник паттернов и граблей

Источник: https://help.1forma.ru/domains/smart-actions/js-jint-patterns/

Практический справочник для написания JS-смарт-скриптов в 1Форме. Каждый паттерн проверен на реальных площадках. Цель — не повторять типичные ошибки.
 Справочник методов JS API — в Jint — JS-интерпретатор. Полный каталог смарт-действий — в Справочнике смарт-действий.
 Каждый SmartScript и JS-вставка обязаны содержать комментарий с версией и датой в начале скрипта:
 `// v1 | 2026-03-08 15:30 | Начальная версия
// v2 | 2026-03-08 16:45 | Добавлена проверка прав
// v3 | 2026-03-10 09:00 | Фикс: TTL кэша 24ч → 1ч
`
- Версия инкрементируется при каждом изменении

- Дата/время — момент правки (local time)

- Краткое описание — что изменилось

- Если скрипт без версии — добавить `v1` при первом касании

`DB_TYPE` — глобальная переменная, доступная в JS SmartScripts: `"MSSQL"` или `"PG"`.
 Проблема: `Key`, `Value` — reserved words. MSSQL требует `[Key]`, PG — `"key"` (lowercase). `TOP N` — только MSSQL, PG — `LIMIT N`.
 Паттерн: функция `qi()` (quote identifier):
 `var IS_PG = DB_TYPE === "PG";
function qi(name) { return IS_PG ? '"' + name.toLowerCase() + '"' : '[' + name + ']'; }
` Использование:
 `// SettingsCustom — самый частый случай
var val = SQL.scalar(
    "SELECT " + qi('Value') + " FROM SettingsCustom WHERE " + qi('Key') + " = 'my_setting'");

// SELECT с TOP/LIMIT
var rows = SQL.query(
    "SELECT " + (IS_PG ? "" : "TOP 1 ") + qi('Value') +
    " FROM SettingsCustom WHERE " + qi('Key') + " = '" + escSql(key) + "'" +
    (IS_PG ? " LIMIT 1" : ""));

// INSERT/UPDATE
SQL.query("INSERT INTO SettingsCustom (" + qi('Key') + ", " + qi('Value') + ") VALUES (@k, @v)",
    { k: key, v: value });
` Полный набор хелперов (для скриптов с тяжёлыми SQL):
 `var IS_PG = DB_TYPE === "PG";
function qi(name) { return IS_PG ? '"' + name.toLowerCase() + '"' : '[' + name + ']'; }
var NL = IS_PG ? '' : ' WITH(NOLOCK)';
function topN(n) { return IS_PG ? '' : 'TOP ' + n + ' '; }
function limitN(n) { return IS_PG ? ' LIMIT ' + n : ''; }
var NOW_FN = IS_PG ? 'NOW()' : 'GETDATE()';
var RV_COL = IS_PG ? "xmin::text::bigint" : "CAST(row_version AS BIGINT)";
var RV_ORDER = IS_PG ? "xmin" : "row_version";
function slen(col) { return IS_PG ? 'LENGTH(' + col + ')' : 'LEN(' + col + ')'; }
function ifnull(expr, def) { return IS_PG ? 'COALESCE(' + expr + ', ' + def + ')' : 'ISNULL(' + expr + ', ' + def + ')'; }
` Важные нюансы:

- PG хранит identifiers в lowercase → `qi('Value')` → `"value"` (не `"Value"`)
  Серверный слой при DataTable -> Dictionary использует StringComparer.OrdinalIgnoreCase, поэтому доступ к колонкам результата через объект/словарь в runtime обычно не зависит от регистра имени поля. Это не отменяет необходимость правильно квотировать identifiers в самом SQL для PG.

- MSSQL: `sc.key` без скобок → `Incorrect syntax near the keyword 'key'`

- PG: `[Key]` → синтаксическая ошибка (квадратные скобки не поддерживаются)

- `WITH(NOLOCK)` — только MSSQL, на PG не существует. `var NL` — динамический хинт

- `LEN()` → `LENGTH()` (PG). `ISNULL()` → `COALESCE()` (PG)

- `row_version` (MSSQL) → `xmin` (PG) для change tracking

- String concat: MSSQL `+`, PG `||`. Пример: `IS_PG ? "a || b" : "a + b"`

- `N'...'` (unicode prefix) — только MSSQL, на PG убирать
  Lua-аналог: `get_mspg_query()` с тегами `{MS}...{/MS}` и `{PG}...{/PG}` + `$NOW`.



Базовый пример:
 `CACHE.set("my_key", "value", 3600);   // lifetime в секундах
var val = CACHE.get("my_key");         // → string или null
CACHE.delete("my_key");
` TTL по умолчанию — `MaxInt` (фактически бесконечно). Без явного lifetime значение живёт вечно (до рестарта процесса). Всегда указывай TTL.
 `// Плохо — кэш никогда не протухнет
CACHE.set("nav_index", data);

// Хорошо — явный TTL
CACHE.set("nav_index", UTILS.json_encode(data), 86400);  // 24ч
` Значение — строка. Объекты → `UTILS.json_encode()` перед set, `UTILS.json_decode()` после get.

`CACHE` в JS/Lua скриптах — тот же `ScriptDataCache` (`LuaCache`), что использует серверный код 1Формы. Это discovery: из скрипта можно читать и инвалидировать серверные кэши.
 `js-scripting-jint.md:189` — «Использует тот же ScriptDataCache (LuaCache), что и Lua-скрипты — кэш общий.»
 Следствия:

- Ключи из разных скриптов (JS и Lua) конфликтуют. Используй уникальные префиксы: `ss492_`, `ai_search_`, `gitlab_`.

- При обновлении скрипта через SQL (не через admin API) CACHE не сбрасывается. Нужно менять имя ключа или ждать TTL. Через admin API — кэш скрипта инвалидируется автоматически (`SQLRefactor/CLAUDE.md`).
  Паттерн: ограничение числа вызовов на пользователя.
 `function checkRateLimit(userId, maxPerHour) {
    var key = "ss492_rate_" + userId;
    var raw = CACHE.get(key);
    var count = raw ? parseInt(raw) : 0;
    if (count >= maxPerHour) return false;
    CACHE.set(key, String(count + 1), 3600);  // TTL = 1 час
    return true;
}

if (!checkRateLimit(authorId, 500)) {
    postComment(taskId, "Превышен лимит запросов. Попробуйте позже.");
    return;
}
` Бюджет токенов, история диалогов, промежуточные состояния — всё хранится в CACHE с TTL.
 `// Трекинг бюджета за день
var budgetKey = "ss492_budget_" + today();
var raw = CACHE.get(budgetKey);
var spent = raw ? parseInt(raw) : 0;
if (spent + estimatedTokens > DAILY_BUDGET) {
    // budget hit — отказ или fallback на дешёвую модель
    return;
}
CACHE.set(budgetKey, String(spent + usedTokens), 86400);
` Проверяйте результат `CACHE.get` и на `null`, и на строку `"null"`:
 `var raw = CACHE.get("my_key");
// raw === null → ключа нет
// raw === "null" → кто-то закэшировал json_encode(null)!

if (!raw || raw === "null") {
    // rebuild
}
` `SQLRefactor/CLAUDE.md` — «CACHE:get проверять на ~= nil и ~= "null". Нет API для ручного сброса».

Базовый паттерн:
 `var resp = HTTP.send_http_request(
    "POST",
    "https://api.example.com/endpoint",
    null,                          // query params
    {
        "Content-Type": "application/json",
        "Authorization": "Bearer " + token
    },
    UTILS.json_encode(body),       // rawBody
    null                           // options
);
` Проблема: `HTTP.send_http_request` возвращает не десериализованный JSON, а обёрточный объект.
 Решение:
 `var resp = HTTP.send_http_request("POST", url, null, headers, rawBody);

// НЕПРАВИЛЬНО — так не работает
// var data = resp.data[0].embedding;

// ПРАВИЛЬНО
if (resp.InnerError) throw new Error(resp.InnerError);
var content = resp.HttpResponse.ResponseContent;  // строка!
var parsed = UTILS.json_decode(content);
var data = parsed.data[0].embedding;
`

Проблема: внутренний `HttpClientType.DefaultHttpClientV2` имеет `TimeoutSeconds = 15`. Для тяжёлых запросов (Claude API, большие payload) 15 секунд мало → `TaskCanceledException`.
 Решение: прямой вызов CLR через `UTILS.resolve_instance`:
 `var httpSvc = UTILS.resolve_instance(
    "Valhalla.Services",
    "Valhalla.Services.HttpRequestService"
);

// Создаём options с увеличенным таймаутом
var options = UTILS.create_instance(
    "Valhalla.Services",
    "Valhalla.Services.Models.HttpRequestOptions"
);
options.RequestTimeoutMs = 120000;  // 2 минуты

var result = httpSvc.SendBodyContainingHttpRequest(
    url, "POST", rawBody, headersDict, options
);
`

Проблема: «простые» запросы (без credentials/proxy/auth) идут через shared `IHttpClientFactory`. Общий `CookieContainer` сохраняет `Set-Cookie` и автоматически подставляет cookie при повторных запросах к тому же домену в течение ~2 минут — в ответе `Set-Cookie` повторно не приходит.
 Решение (если нужно получать свежую cookie на каждый вызов — ротация сессий и т.п.):
 `var resp = HTTP.send_http_request('POST', url, null, headers, rawBody,
    { DisableCookieContainer: true });
// resp.HttpResponse.ResponseHeaders.Set-Cookie теперь приходит на каждом вызове
` Внимание на регистр: в JS-Jint — `DisableCookieContainer` (PascalCase). В Lua — `disable_cookie_container` (snake_case). Полное описание опции и причина переключения с shared-клиента на одноразовый: `admin.md` § «HTTP-опции: изоляция cookie-контейнера».
 Проблема: при сериализации `Dictionary` через ToJSON (внутренний сериализатор HTTP API) пары key-value не оборачиваются в `{}`. JSON невалиден.
 Решение: создавать словари через `UTILS.create_instance`:
 `var headers = UTILS.create_instance(
    "System.Private.CoreLib",
    "System.Collections.Generic.Dictionary`2[[System.String],[System.String]]"
);
headers.Add("Content-Type", "application/json");
headers.Add("X-Api-Key", apiKey);
`

Паттерн для внешних API (Claude, OpenAI, Cohere):
 `function httpWithRetry(method, url, headers, body, maxRetries) {
    maxRetries = maxRetries || 3;
    for (var attempt = 0; attempt < maxRetries; attempt++) {
        var resp = HTTP.send_http_request(method, url, null, headers, body);
        var status = resp.HttpResponse ? resp.HttpResponse.StatusCode : 0;
        if (status === 200) return resp;
        if (status === 429 || status === 500 || status === 529) {
            // backoff: ~1s, ~2s, ~4s
            var waitMs = Math.pow(2, attempt) * 1000;
            var start = Date.now();
            while (Date.now() - start < waitMs) { /* busy wait — Jint не имеет sleep */ }
            continue;
        }
        break;  // 4xx (кроме 429) — не ретраить
    }
    return resp;
}
` Важно: в Jint нет `setTimeout`/`sleep`. Busy wait допустим при коротких паузах (до 5 секунд), но считается в `MaxStatements`.
 Обращение к внутренним стендам с самоподписанным сертификатом:
 `// *.{host} — self-signed сертификаты
// HTTP.send_http_request по умолчанию НЕ проверяет сертификаты
// Дополнительных действий не нужно
var resp = HTTP.send_http_request("GET", "https://{host}/api/...");
`

Прямой доступ к сервисам платформы из смарт-скрипта — без SQL и без HTTP-запросов.
 Когда использовать: когда нужная логика уже реализована в сервисе платформы и дублировать её в SQL нецелесообразно (например, поиск по документам — без SQL, напрямую через сервис).
 Получение существующего сервиса платформы:
 `var searchSvc = UTILS.resolve_instance(
    "Valhalla.Services",                         // assembly name
    "Valhalla.Services.SearchService"             // full type name
);
var result = searchSvc.SimpleSearch(query, userId);
`

Создание объектов напрямую (когда нет DI-регистрации):
 `// Словарь (для HTTP headers и т.п.)
var dict = UTILS.create_instance(
    "System.Private.CoreLib",
    "System.Collections.Generic.Dictionary`2[[System.String],[System.String]]"
);

// List<int>
var list = UTILS.create_instance(
    "System.Private.CoreLib",
    "System.Collections.Generic.List`1[[System.Int32]]"
);

// Npgsql connection (для PG fallback)
var conn = UTILS.create_instance(
    "Npgsql",
    "Npgsql.NpgsqlConnection",
    [connStr]    // constructor args — массив
);
`

`TextUtils` — не в DI (все методы static), поэтому `resolve_instance` не сработает. Но `create_instance` (Activator) создаёт инстанс, а Jint через ObjectWrapper видит static-методы. Проверено на dev 2026-04-11.
 `var tu = UTILS.create_instance("Valhalla.Utils", "Valhalla.Utils.Text.TextUtils");

// FTS (CONTAINS/CONTAINSTABLE) — tokenize, escape, FORMSOF, обрезка длины
var ftsQuery = tu.FormSearchParameterForFullTextContainsSearch(userInput, false);
// "тест поиск" → FORMSOF(INFLECTIONAL, "тест") and FORMSOF(INFLECTIONAL, "поиск")
// второй аргумент true = искать в альтернативной раскладке (EN↔RU)

// LIKE — bracket escaping [%], [_], [[]
var likeQuery = tu.FormSearchParametersForStandartSearch(userInput);
// "test%_[special" → "test[%]_[[]special"

// Обрезка по лимиту (Constants.SearchTextLengthLimit)
var trimmed = tu.CutSearchStringIfTooLong(userInput);
` Зачем: проверенная на реальных площадках очистка (используется в списках задач, отчётах, ленте комментариев, фильтрах) — не нужно дублировать логику в JS.
 Ключевое: `create_instance`, не `resolve_instance`. Static-классы не регистрируются в Autofac → `resolve_instance` бросит исключение.
 Не все типы создаются автоматически:
 Тип create_instance Workaround `List<int>` ✅ — `CancellationToken` ✅ value type — `GridSettings` ✅ (но namespace `Dto.AgGrid.GridSettings`, НЕ `Dto.NewDataSources.GridSettings`) — `GetDataSourceRequest` ❌ возвращает null Duck-typed JS object Jint маппит JS-объект на C#-параметр по совпадению имён свойств:
 `// GetDataSourceRequest не резолвится через create_instance
// Но C#-метод примет JS-объект с нужными полями:
var dsRequest = {
    Request: rowRequest,
    GridSettings: gridSettings,
    UserId: 28,
    TaskIdsOnly: true
};
service.GetTasksDataAsync(dsRequest, ct);  // работает!
`

Точные имена сервисов и сборок:
 Сервис Assembly Type RequestHandlerService `Valhalla.Services` `Valhalla.Services.RequestHandlerService` ISearchService `Valhalla.Services` `Valhalla.Services.SearchService` CommentService `Valhalla.Services` `Valhalla.Services.CommentService` HttpRequestService `Valhalla.Services` `Valhalla.Services.HttpRequestService` НЕ `Valhalla.Services.AgGridServices.RequestHandlerService` — неверный путь, не найдёт.
 Что нужно учитывать:

- Это обращение к внутренним сервисам — гарантий стабильности между версиями нет.

- Имена сервисов и сборок могут меняться между версиями.

- Если тип не разрешён в смарт-скриптах — будет ошибка.

Добавление комментария:
 `SMART.execute_action_in_task_context("PostComment", taskId, {
    Task: taskId,                    // ОБЯЗАТЕЛЬНО — иначе NullRef
    CommentAuthor: 3,                // ОБЯЗАТЕЛЬНО — автор (UserID); 3 — Робот 1Ф
    CommentText: "Текст",
    Recipients: [authorId],          // кому адресован ответ
    ForcedEmail: false,              // ОБЯЗАТЕЛЬНО — дублировать на почту
    CommentSMS: false,               // ОБЯЗАТЕЛЬНО — дублировать по SMS
    NoSubscription: true,            // ОБЯЗАТЕЛЬНО — не подписывать адресатов на задачу
    TextAsHTML: false                // ОБЯЗАТЕЛЬНО — трактовать текст как HTML
});
` Важно: PostComment NullRef без параметра Task. `SMART.execute_action_in_task_context("PostComment", ...)` без `Task` в params → binder добавляет `pars[2]=null` → `(int)null` = NullReferenceException.
 Смена статуса задачи:
 `SMART.execute_action_in_task_context("ChangeTaskState", taskId, {
    StepId: targetStateId,
    Comment: "[auto:gitlab-sync] Создана ветка feature/123-fix"
});
` Обновление значения ДП:
 `SMART.execute_action_in_task_context("UpdateExtParam", taskId, {
    ExtParamId: 12408,
    Value: branchName
});
` Асинхронное выполнение действий:
 `// async = true — выполнение в фоне (не блокирует скрипт)
SMART.execute_action_in_task_context("PostComment", taskId, {
    Task: taskId,
    CommentAuthor: 3,
    CommentText: "Фоновая операция",
    ForcedEmail: false,
    CommentSMS: false,
    NoSubscription: true,
    TextAsHTML: false
}, true);
`

Два механизма загружают library-скрипты (IsLibrary=true). Отличаются временем выполнения и областью видимости.
 Механизм Вызов Когда применим Как резолвит `include(X)` `include(496)` / `include("agent-tools-local")` Синхронный. Можно на top level. ID или Description `import(X)` `await import("Module:agent-utils")` Async. Только внутри `async`-функции. Description (рекомендуется) — не ломается при dev ↔ HD restore Глубина include-рекурсии — не более 5 уровней.
 `js-scripting-jint.md:210-215`
 С Wave 1-5 (2026-04) Анфиса полностью на ES-модулях через `import()`:
 `// оркестратор (фрагмент)
AgentUtils = await import('Module:agent-utils');
AgentConfig = await import('Module:agent-config');
// ... 40+ импортов
// Остались 2 include (отдельный трек миграции / не в плане):
include("ai-search");
include("agent-tools-1c");
` Выносить в отдельные library-SS:

- Конфигурацию (константы, ID, SettingsCustom-ключи)

- Утилиты (парсинг, форматирование, retry-логику)

- API-обёртки (Claude, OpenAI, GitLab, Telegram)
  Новый код — через `import()` с префиксом `Module:` в Description. Один параметризованный скрипт + библиотеки лучше трёх копий одного кода (антипаттерн P5 из Telegram-аудита).

В асинхронном режиме код скрипта оборачивается в `(async () => { ... })()`. Любой `var` внутри попадает в область видимости этой обёртки, а не в глобальные переменные движка. Подключаемый через `include()` скрипт видит только глобальные переменные.
 Симптом: include-скрипт получает `ReferenceError` на переменную, которая «точно объявлена» в orchestrator.
 Реальный пример (agent-loop v223): `// Line 264 — implicit global (видно в include):
_currentTaskId = null;

// Line 1263 — var-дубликат, shadow'ил глобал (НЕ видно в include):
var _currentTaskId = 0;  // ❌
` Все include-скрипты падали с ReferenceError на `_currentTaskId`. Fix — убрать `var`.
 Правило: Переменные, которые должны быть видны include-скриптам, объявлять через implicit global (`_foo = value`), не `var _foo = value`. Перед добавлением переменной — `grep` на совпадения.
 Связанный случай: внутри самого library-скрипта `var _FOO = true` тоже не создаёт engine-global — родительский SS не увидит его через implicit global. Пример: SS 612 dispatcher, починено 2026-04-04 (убрали `var` из `_ANFISA_JOB_MODE = true`).

Оба механизма декларируют функции/переменные в engine globals. Если функция с одним именем определена в обоих — выигрывает тот, что выполнился позже. В оркестраторе типично `import()`-ы идут после `include()`-ов → import перезаписывает include.
 Пример: если одна и та же функция определена и в подключаемом через `include` скрипте, и в импортируемом модуле, то правка include-скрипта может не дать эффекта — при выполнении используется версия из модуля.
 Правило при правке функции:

- `grep funcName` по всем library-скриптам.

- Если больше одного определения — править все. Или (надёжнее) убедиться что правишь ту, что в активной import-цепочке.

- При наличии одновременно include-версии и import-версии приоритет у импортируемой версии.

P1 из Telegram-аудита: три бот-токена Telegram вшиты прямо в Lua-код SmartScript. Любой администратор с доступом к SmartScripts видит токены.
 Хранение настроек в `SettingsCustom`:
 `// Чтение секрета в рантайме
var token = SQL.scalar(
    "SELECT Value FROM SettingsCustom WHERE [Key] =@name",
    { name: "gitlab_private_token" }
);
token = decrypt(token);  // если XOR-encoded
` Для XOR-encoding: ключ хранится в конфигурации (appsettings.json), значение — hex-строка в SettingsCustom.
 `gitlab-1forma-integration-spec.md:415-424` — конфигурация секретов.
 Шифрование и расшифровка строк:
 `var encrypted = encrypt("secret_value");   // → зашифрованная строка
var original = decrypt(encrypted);          // → исходное значение
` Встроенные функции используют серверный ключ шифрования. Подходят для хранения в БД.
 Публикации с анонимным доступом открыты всему интернету. Входящие данные нельзя доверять.
 `// Верификация webhook secret (GitLab)
var secret = decrypt(SQL.scalar(
    "SELECT Value FROM SettingsCustom WHERE [Key] ='gitlab_webhook_secret'"
));
if (params["secret"] !== secret) {
    RESULT = { error: "unauthorized" };
    return;
}
` `gitlab-1forma-integration-spec.md:196-203`

Как устроен модульный скрипт:
 `Внешний сервис (GitLab, Telegram, ...)
  → POST /app/v1.2/api/publications/action/{alias}
    → PublishedObjectsController [AllowAnonymous]
      → SmartPublicationsService
        → ActionsPack:
          1. HTTP-ответ {"ok": 200}      (мгновенный ответ, Action=88)
          2. JS SmartScript               (основная логика, async)
` Payload доступен через `EVENTPARAMS["PublishedObjectParameters"]`.
 ВАЖНО: `EVENTPARAMS` доступен ТОЛЬКО на верхнем уровне скрипта. Внутри `try-catch` блоков не виден (Jint scope quirk). Сохраняй в переменную ДО try-catch.
 `../notifications/admin.md` — настройка публикаций.

Важно: `PublishedObjectParameters` — НЕ плоский объект. Структура:

- `params.requestBody` — JSON-тело запроса (объект)

- `params.headers` — HTTP-заголовки (ключи в mixed-case: `x-Gitlab-Token`, `x-Gitlab-Event`, `content-Type`)
  `var raw = EVENTPARAMS["PublishedObjectParameters"];
var params = typeof raw === "string" ? UTILS.json_decode(raw) : raw;

var headers = params.headers || {};
var body = params.requestBody || {};

// Headers приходят в mixed-case — нужен case-insensitive поиск
function getHeader(name) {
    var lower = name.toLowerCase();
    var keys = Object.keys(headers);
    for (var i = 0; i < keys.length; i++) {
        if (keys[i].toLowerCase() === lower) return headers[keys[i]];
    }
    return "";
}

var event = getHeader("X-Gitlab-Event") || body.object_kind;
` Важно: `CommentAuthor` обязателен при вызове из анонимной публикации. Без него SessionUserId=-1 → NullRef → HTTP 500.
 `var ROBOT_USER_ID = 3;  // Робот 1Ф

SMART.execute_action_in_task_context("PostComment", taskId, {
    CommentAuthor: ROBOT_USER_ID,  // ОБЯЗАТЕЛЬНО в публикациях!
    Task: taskId,
    CommentText: "[auto:gitlab] MR !42 принят",
    SilentComment: false,
    ForcedEmail: false,
    CommentSMS: false,
    NoSubscription: true,
    TextAsHTML: false
});
` Действие называется `ChangeExtParamValue` (не `UpdateExtParam`).
 `SMART.execute_action_in_task_context("ChangeExtParamValue", taskId, {
    Task: taskId,
    User: ROBOT_USER_ID,
    ExtParam: epId,
    Value: String(value),
    WriteCommentOnChange: false
}, false);
`

`ChangeExtParamValue` работает для Table ДП. Value — строка с префиксом-операцией.
 Добавление строки (`+`): `// columnId без кавычек, значения в "First"
Value: '+[{111:{"First":"текст"},222:{"First":"42"}}]'
`
 Изменение строки (`=`): `// "First" = rowId строки, "Second" = объект с колонками
Value: '={"First":"555","Second":{111:{"First":"новый текст"}}}'
`
 Массовое изменение (`=[...]`): `Value: '=[{"First":"1","Second":{100:{"First":"a"}}},{"First":"2","Second":{100:{"First":"b"}}}]'
`
 Удаление строки (`-`): `Value: '-555'   // 555 = rowId
`
 Полная перезапись (`#` smart merge, `|` wipe+create): `// # — удаляет отсутствующие, добавляет новые, обновляет совпадающие
Value: '#{"1":{11:{"First":"текст1"}},"2":{11:{"First":"текст2"}}}'
// | — удаляет все строки, создаёт заново (если rowId неизвестны)
Value: '|{"1":{11:{"First":"текст1"}},"2":{11:{"First":"текст2"}}}'
`
 Экранирование в Value: `"` → `\"`, `\` → `\\`, `'` → `''`. Переносы `\n\r` — без экранирования.
 Колонка SelectUsers: `Value: '+[{9:{"First":"{\\"Users\\":{\\"Added\\":[2122]}}"}}]'
`
 Колонка Выпадающий список: `"First"` = текст, `"Second"` = значение (обязателен для выпадающих).

При двусторонней синхронизации (1Ф ↔ GitLab, 1Ф ↔ Telegram) скрипт может триггерить сам себя.
 Решение: текстовый маркер в комментариях.
 `var MARKER = "[auto:gitlab]";

// Входящий скрипт — маркирует свои комментарии
SMART.execute_action_in_task_context("PostComment", taskId, {
    CommentAuthor: ROBOT_USER_ID,
    Task: taskId,
    CommentText: MARKER + " MR !42 принят",
    SilentComment: false,
    ForcedEmail: false,
    CommentSMS: false,
    NoSubscription: true,
    TextAsHTML: false
});

// Исходящий скрипт — проверяет маркер и пропускает свои
if (commentText.indexOf("[auto:gitlab]") >= 0) return;
` Два маркера: `[auto:gitlab-sync]` (вход) и `[auto:1forma-sync]` (выход) — чтобы различать направление.
 `gitlab-1forma-integration-spec.md:427-434`

Telegram и GitLab гарантируют at-least-once delivery. Повторный webhook → дубль комментария.
 Решение через CACHE:
 `function isProcessed(deliveryId) {
    if (!deliveryId) return false;
    var key = "gitlab_delivery_" + deliveryId;
    if (CACHE.get(key)) return true;
    CACHE.set(key, "1", 86400);  // 24ч TTL
    return false;
}

var deliveryId = params["X-Gitlab-Delivery"] || "";
if (isProcessed(deliveryId)) {
    RESULT = { status: "duplicate" };
    return;
}
` `gitlab-1forma-integration-spec.md:437-454` — паттерн идемпотентности.

`SQL.query` возвращает список (нумерация с 0), а не словарь:
 `var rows = SQL.query("SELECT TaskID, Description FROM Tasks WHERE SubcatID = @sub", { sub: 5641 });

// НЕПРАВИЛЬНО:
// Object.keys(rows) → ["Capacity", "Count"] — НЕ индексы строк

// ПРАВИЛЬНО:
for (var i = 0; i < rows.Count; i++) {
    var row = rows[i];
    var taskId = row.TaskID;       // PascalCase, как в SQL
    var desc = row.Description;
}
`

Важно: `SQL.query_one()` возвращает `null` даже для существующих строк в контексте публикаций (OnCallPublishedObject, eventId=106). Причина — Jint interop: метод существует, но маршаллинг результата ломается.
 Рабочая замена — `SQL.query()` + `rows[0]`:
 `// НЕ работает: var task = SQL.query_one("SELECT ...", { id: taskId });
// Работает:
var rows = SQL.query("SELECT TOP 1 TaskID, SubcatID, IsClosed FROM Tasks WHERE TaskID = " + parseInt(taskId));
var task = rows.Count > 0 ? rows[0] : null;
if (!task) return;
var subcatId = task.SubcatID;
` `SQL.query()` возвращает .NET `List` — используй `.Count` и `[0]`, не `.length`.
 Работает корректно, включая публикации.
 `var count = SQL.scalar("SELECT COUNT(*) FROM Tasks WHERE SubcatID = @sub", { sub: 5641 });
var name = SQL.scalar("SELECT Value FROM SettingsCustom WHERE [Key] = @n", { n: "my_key" });
` Важно: колонка в SettingsCustom — `[Key]`, не `Name`. Квадратные скобки обязательны (зарезервированное слово).

Устаревший модуль (с 2026-05-08). В новых скриптах не используется; раздел оставлен для совместимости со старыми скриптами.
 `VECTORDB.query()` возвращает `Dict<int, Dict<string, object>>` с ключами от 1, не от 0.
 `var rows = VECTORDB.query("SELECT task_id, score FROM vector_search.chunks LIMIT 10");
var rowNum = 1;
while (rows[rowNum]) {
    var row = rows[rowNum];
    var taskId = row.task_id;    // lowercase — PG convention
    rowNum++;
}
` VECTORDB доступен не на всех площадках. Проверить наличие:
 `var hasVectordb = typeof VECTORDB !== "undefined";
`

Важно. `AddWithValue("$1", value)` в Npgsql НЕ биндит позиционные `$N`. Ошибка: `42P02 — parameter $1 does not exist` или silent mismatch.
 Причина: `AddWithValue` ожидает имя параметра, Npgsql автоматически добавляет `@` при связывании. SQL должен использовать `@pN`, AddWithValue — `"pN"` (без `@`).
 `// НЕПРАВИЛЬНО — $N через AddWithValue
pgExec(
  "INSERT INTO table (col1, col2) VALUES ($1, $2)",
  { "$1": val1, "$2": val2 }
);
// → 42P02: parameter $1 does not exist

// ПРАВИЛЬНО — @pN в SQL, pN в AddWithValue
pgExec(
  "INSERT INTO table (col1, col2) VALUES (@p1, @p2)",
  { "p1": val1, "p2": val2 }
);
` Cast-синтаксис: `@p1::text[]`, `@p1::halfvec(1024)` — работает корректно с `@pN`.
 Null-значения: `AddWithValue("pN", null)` в Npgsql интерпретируется как «нет параметра», не как DBNull. CLR interop из SmartScript не поддерживает `System.DBNull.Value`. Вместо null передавать safe defaults: `"{}"` (пустой PG array), `0`, `""`.
 Применимо к функциям-обёрткам над PG-запросами в скриптах.
 `docs/domains/ai/anfisa-qa-memory-spec.md` — секция 10.2, полный разбор.

Имена колонок в 1Форме контринтуитивны. Не угадывай — проверь DDL (`DB_MSSQL/dbo.{Table}.Table.sql`).
 Типичные ошибки:
 Таблица Ожидаешь На самом деле Примечание `Subcategories` `Name` `Description` (varchar 140) Нет колонки Name вообще `ExtParams` `Name` `ExtParamName` (varchar 900) Нет колонки Name `SettingsCustom` `Key`, `Value` `[Key]`, `[Value]` Зарезервированные слова — квотировать через `qi()` `SmartExpressions` `Public` `[Public]` (bit) Зарезервированное слово — `qi("Public")` `Categories` — Это Разделы, не Категории Категории = `Subcategories` Зарезервированные слова (требуют `qi()` или квадратных скобок): `Name`, `Order`, `Key`, `Value`, `Public`, `Type`, `Description`, `Default`. Если колонка — зарезервированное слово, используй `qi("ColumnName")`.
 Правило при написании SS с SQL: перед деплоем — для каждого SQL-запроса проверить все колонки по DDL в `DB_MSSQL/`. Не полагаться на «очевидные» имена.
 Правило при создании нового SS через API:

- `language` — enum-строка: `"Lua"`, `"JavaScript"`, `"Python"` (не `"JS"`, не `"Js"`)

- `contextType` — строка `"None"` (не null, не 0)

- `isLibrary` — boolean (true/false)

Ограничения песочницы выполнения скриптов:
 Параметр Значение Timeout 5 минут MaxStatements 1 000 000 CLR-типы Заблокированы по умолчанию — только whitelist CLR exceptions Перехватываются как JS Error (`ExceptionHandler = _ => true`) `js-scripting-jint.md:63-82`
 `print()` пишет в консоль смарт-скрипта (вывод движка), но не в комментарии и не в журналы. На рабочей площадке его не видно никому.
 Паттерн:
 `var _perfLog = [];

function logPerf(label, startMs) {
    _perfLog.push(label + ": " + (Date.now() - startMs) + "ms");
}

// В конце скрипта — постим лог в служебный тред
if (_perfLog.length > 0) {
    SMART.execute_action_in_task_context("PostComment", taskId, {
        Task: taskId,
        CommentAuthor: CONFIG.ANFISA_USER_ID,
        CommentText: "[perf] " + _perfLog.join(", "),
        ThreadCommentID: serviceThreadId,   // тред служебных сообщений (ThreadCommentID, не ThreadCommentId)
        Recipients: [CONFIG.ANFISA_USER_ID],
        ForcedEmail: false,
        CommentSMS: false,
        NoSubscription: true,
        TextAsHTML: false
    });
}
` Для постоянного логирования — `AutomationScriptsLog` (таблица БД):
 `// v4: замена print → logToAutomation
// Доступен через SMART или SQL в зависимости от версии
SQL.query(
    "INSERT INTO AutomationScriptsLog (ScriptId, Message, TaskId, CreatedAt) VALUES (@sid, @msg, @tid, GETDATE())",
    { sid: 492, msg: "Tool: qmd_search, 230ms", tid: taskId }
);
` `SESSION_USER.Id` (не `.UserId`). Работает даже при тестовом запуске (id=0, contextType=5).
 CONTEXT и EVENTPARAMS при тестовом запуске = undefined.
 `task.Result` для CLR Task-ов безопасен в ASP.NET Core (нет SynchronizationContext). Deadlock невозможен.
 ВАЖНО: `await` НЕ работает в рекуренсах (синхронный запуск). `editor/execute` использует `ExecuteAsync` и поддерживает `await`, но рекуренс запускает скрипт синхронно — top-level `await` вызовет ошибку парсинга: `"await is only valid in async functions"`. Всегда использовать `.Result`.

Формат запроса к API редактора скриптов:
 `{
    "script": {
        "id": 0,
        "scriptCode": "...",
        "language": 1
    }
}
` НЕ `{"id":0, "languageId":1, "code":"..."}` — вызовет NullReferenceException (`editor.Script = null`).
 Поля SmartScriptDto: `language` (ScriptLanguage enum: 0=Lua, 1=JavaScript, 2=Python), `eventId` (106 для публикаций), `isLibrary`, `contextType`, `description`. Поле `languageId` не существует — будет проигнорировано, язык останется 0 (Lua). При обновлении без `eventId` — значение обнулится → EVENTPARAMS undefined.
 Save: `{"script": scriptDto, "context": {"subcatId": ..., "eventId": ..., "contextType": ...}}`.

`HTTP.send_http_request` не поддерживает `Content-Type: multipart/form-data; boundary=...` — `StringContent` в .NET 8+ отклоняет boundary в MediaTypeHeaderValue. `HTTP.post_multipart` с `Value` (без FileId) создаёт StringContent без filename в Content-Disposition.
 Решение: создать MultipartFormDataContent через CLR:
 `var multipart = UTILS.create_instance(
  "System.Net.Http", "System.Net.Http.MultipartFormDataContent", []);
var modelContent = UTILS.create_instance(
  "System.Net.Http", "System.Net.Http.StringContent", [modelJson]);
multipart.Add(modelContent, "modelData");
var fileContent = UTILS.create_instance(
  "System.Net.Http", "System.Net.Http.StringContent", [fileText]);
multipart.Add(fileContent, "file", fileName);  // 3-й аргумент = filename

var httpClient = UTILS.create_instance(
  "System.Net.Http", "System.Net.Http.HttpClient", []);
httpClient.DefaultRequestHeaders.TryAddWithoutValidation("1F-Pat", pat);
var response = httpClient.PostAsync(url, multipart).Result;
var body = response.Content.ReadAsStringAsync().Result;
multipart.Dispose();
httpClient.Dispose();
` `docs/domains/integrations/docs-disk-sync-gitlab.js` — рабочий пример (SS 510).

Табличные ДП (Table ExtParam) — запись/обновление/удаление строк из JS SS.
 Методы API для табличных ДП в мобильном приложении:
 Версия Endpoint Описание v1.0 `POST /app/v1.0/api/mobile/tasks/{taskId}/extparamtable` Устаревший, тело: `{id: epId, rows: string[]}` v1.2 `POST /app/v1.2/api/task/{taskId}/ep/table/{epId}/update` Актуальный, формат тела ниже

Структура тела запроса:
 `// Добавить строку в Table EP
var body = {
  rows: [{
    id: 0,                    // 0 = новая строка
    modifyType: "create",     // "create" | "update" | "delete"
    cols: [
      { id: 35911, value: "Валидация" },       // Фаза (Text column)
      { id: 35921, value: "OK" },               // Статус (Text column)
      { id: 35931, value: "YAML parsed" },      // Детали (Text column)
      { id: 35941, value: "2026-03-13T12:00:00" } // Timestamp (DateTime column)
    ]
  }]
};

// Вызов из SS через HTTP.fetch (v1.2 endpoint)
var resp = HTTP.fetch(
  BASE_URL + "/app/v1.2/api/task/" + taskId + "/ep/table/" + epId + "/update",
  { method: "POST", body: JSON.stringify(body), headers: {"Content-Type": "application/json"} }
);
` Формат значений в колонках:
 Тип колонки Формат value Text строка: `"some text"` DateTime ISO: `"2026-03-13T12:00:00"` Number строка числа: `"42.5"` Select/Combobox ID значения: `"12755"` Lookup taskId: `"98765"` Чтение данных табличного ДП:
 `POST /app/v1.2/api/task/{taskId}/ep/table/{epId}
Body: { take: 100, skip: 0 }
` Ответ: `data.data[]` — массив строк. Значения в полях `c{columnId}`, например:

- `c35911.stringValue` — текст

- `c35941.dateTimeValue` — дата
  Обновление и удаление строк:
 `// Обновить существующую строку
{ id: 12345, modifyType: "update", cols: [{ id: 35921, value: "Failed" }] }

// Удалить строку
{ id: 12345, modifyType: "delete", cols: [] }
` Частые ошибки:

- id=0 для новой строки — обязательно, `null` или отсутствие → ошибка

- modifyType — строка, не enum-число. `"create"`, не `0`

- При `"delete"` массив `cols` можно передать пустым, но поле должно присутствовать

- v1.0 endpoint принимает `rows` как `string[]` (JSON-строки), v1.2 — как объекты

Проблема: `if (chatId == -123456789) { ... } else if (chatId == -987654321) { ... }` — каждый новый клиент/чат требует правки кода.
 Решение: конфигурационная таблица или SettingsCustom с JSON.
 `// Плохо
if (subcatId == 5641) { ... }

// Хорошо
var config = UTILS.json_decode(
    SQL.scalar("SELECT Value FROM SettingsCustom WHERE [Key] ='gitlab_state_mapping'")
);
var mapping = config[String(subcatId)];
` Проблема: три копии одного скрипта (SS 7/274/133 в Telegram). Рассинхронизация гарантирована.
 Решение: один параметризованный скрипт + `include()` библиотек.
 Проблема: прямой `UPDATE Comments SET Content = '...'` — обходит валидацию, ACL, логирование.
 Решение: `SMART.execute_action` или `UTILS.resolve_instance` для вызова сервисов.
 Проблема: без ограничений скрипт может забить внешний API (Anthropic: 2K RPM, OpenAI: лимиты на org). Блокировка ключа необратима.
 Решение: rate limit через CACHE (см. секцию 1).
 Проблема: повторный webhook → дубль комментария, дубль задачи, дубль перехода.
 Решение: delivery ID tracking через CACHE (см. секцию 7).
 Проблема: `print()` пишет в никуда. 30+ вызовов `var_dump` в SS 7 (Telegram) засоряют output.
 Решение: `_perfLog` + служебный тред, или logToAutomation.
 Проблема: `{ФамилияИмя}Temp123456` — зная алгоритм, можно войти под любым автосозданным юзером.
 Решение: случайный пароль или invite-flow без пароля.
 Проблема: побочный эффект при каждом incoming webhook. Мутация в read-path.
 Решение: отдельный scheduled cleanup.
 Смежные разделы:

- Jint — JS-интерпретатор — справочник методов JS API

- Python в смарт-скриптах

- FAQ: обработка ошибок Lua и pcall

- Уведомления и публикации
