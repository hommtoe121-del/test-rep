# C# (Roslyn) — типизированные смарт-скрипты

Источник: https://help.1forma.ru/domains/smart-actions/csharp-scripting-roslyn/



Реализовано. Доступно с 2026-04-20.

Пятый язык смарт-скриптов. Выполняется через Roslyn Scripting API — тот же компилятор C#, что и у ядра Первой Формы, внутри платформы и без внешних зависимостей.
 Ключевое отличие от Lua, JavaScript и OneScript: те выполняются на стороннем движке с прослойкой преобразования данных, а C#-скрипт — это тот же C#, что и сама платформа. Скрипту напрямую доступны контекст сессии и сервисы платформы, без промежуточных преобразований.
 Аспект JS (Jint) C# (Roslyn) Типизация Нет Полная, на этапе компиляции Асинхронность Ограниченная Нативные `async/await`, `Task.WhenAll` Преобразование данных Через обёртки между JS и платформой Нет — прямой вызов сервисов Ошибки Во время выполнения На этапе компиляции (до запуска) Первый запуск ~5 мс ~100–200 мс (первая компиляция) Повторный запуск Средняя скорость (интерпретация) Высокая скорость (после компиляции) Изоляция Встроенная (лимит операций, таймаут) Таймаут (полная изоляция — в планах)

То же правило что и для JS. Каждый C# SmartScript обязан содержать комментарий с версией и датой в начале скрипта:
 `// v1 | 2026-04-21 10:30 | Начальная версия
// v2 | 2026-04-22 15:45 | Добавлена проверка прав
`

Компиляция и выполнение разделены. На этапе компиляции из текста скрипта строится исполняемый код — это дорогая операция (~100–200 мс на первый запуск, до 1–3 с на крупный скрипт ~200 КБ). На этапе выполнения готовый код вызывается с подготовленным контекстом — это уже быстро (нативная скорость).
 Результат компиляции кешируется: для сохранённых скриптов — по идентификатору скрипта, для черновиков из редактора — в отдельном ограниченном кэше. При повторном запуске того же скрипта компиляция не повторяется.

При дочернем вызове сохранённого C# SmartScript платформа проверяет, соответствует ли переданный код сохранённой компиляции. Сначала из записи скрипта выбирается только `CompiledSourceHash` — SHA-256 исходника, сохранённый при компиляции. SHA-256 переданного кода вычисляется потоково и сравнивается с этим значением.
 Если хэши совпали, используется runner из кэша сохранённой компиляции: исходный код `ScriptCode` для такой проверки не загружается. Это уменьшает объём чтения и выделения памяти при повторных дочерних вызовах крупных скриптов.
 Полное сравнение исходного кода сохранено как резервная ветка. Она применяется, если хэш ещё отсутствует у скрипта, сохранённого до появления precompile-колонок, или не совпал с переданным кодом — например, при запуске несохранённого текста из редактора либо после гонки параллельного сохранения. Если резервное сравнение подтверждает совпадение, обычный путь наполнения кэша повторно сохраняет актуальный хэш; если код различается, выполняется ad-hoc компиляция переданного текста.

При дочернем вызове сохранённого C# SmartScript (`RunScriptSync`, `RunScriptBackground`) исходный код скрипта из базы не загружается вовсе — runner берётся из кэша сохранённой компиляции по `scriptId`.
 Как это работает:

- `SmartScriptService.GetSmartScript(id)` возвращает проекцию `GetScriptForRun` — те же поля, что и раньше, но для C# `ScriptCode` приходит как `null` (маркер «код опущен, исполнить по id»). Сам исходник в базе не меняется.

- `RoslynScriptEngine.ResolveRunner` на этом пути делает `CompiledSmartScriptCache.Get(scriptId)` — без чтения `ScriptCode` и без SHA-256 по нему. Актуальность кэша обеспечивается при его наполнении (проверка версии/хеша) и кластерной инвалидацией при сохранении.

- Если runner в кэше отсутствует (язык переключили, строку удалили параллельно) — исходник дочитывается через `GetScriptForCompile` и компилируется ad-hoc; следующий вызов наполнит кэш обычным путём.
  Если дочернему вызову передаётся сам код (например, несохранённый черновик из редактора), применяется сверка по `CompiledSourceHash` (см. выше). Source-движки (JS, OneScript, Lua, Python) этой оптимизацией не затронуты — их проекция по-прежнему отдаёт `ScriptCode`. Поведение скриптов (исключения, ошибки компиляции, проверки редакторских черновиков) не изменилось.

При резолве SmartScript по имени или идентификатору (Include, `RunScriptSync`, `RunScriptBackground`) платформа загружает из базы только необходимые поля, а не полную запись скрипта:

- C# Include — только идентификационные поля (`Id`, `Description`, `LanguageId`, `IsLibrary`, `SubcatId`). Исходный код (`ScriptCode`) и скомпилированные байты (`CompiledAssembly`) не читаются: C#-библиотеки исполняются из кеша скомпилированных сборок (`CompiledSmartScriptCache`).

- Разрешение имени в идентификатор (name→id в JS, OneScript, `SmartScriptService`) — только `Id`.

- Source-движки (JS, OneScript, Lua) — загружается `ScriptCode` (нужен для интерпретации), но не `CompiledAssembly` (это поле использует только C#-движок).
  Резолв крупных библиотечных наборов (например, регистрация десятков модулей через Include на каждый запрос) не тянет из базы тяжёлые `ScriptCode` и `CompiledAssembly`. Семантика, синтаксис и ошибки резолва (неоднозначное имя, NotFound, несовпадение `SubcatId`) не изменились.

Компиляция выполняется в следующих случаях:
 Сценарий Что происходит Первый запуск сохранённого скрипта после перезапуска приложения Компиляция, результат кэшируется по идентификатору скрипта Повторный запуск того же скрипта Берётся из кэша — компиляция не повторяется Сохранение скрипта в редакторе На узле, где сохранили, выполняется компиляция и кэш обновляется. На остальных узлах кластера запись из кэша удаляется, новая версия компилируется при первом следующем запуске Запуск несохранённого черновика из редактора Поиск в кэше черновиков по содержимому скрипта. Если не найдено — компиляция и запись в кэш; если найдено — используется готовый код `LIB.Include("name")` / `Include(id)` внутри скрипта Тот же кэш по идентификатору библиотечного скрипта; первый вызов после изменения компилирует заново

При сохранении или удалении скрипта на одном узле остальные узлы кластера должны узнать, что кешированная компиляция устарела. На остальных узлах из кэша только удаляется запись, без немедленной перекомпиляции. Перекомпиляция выполняется лениво — при следующем обращении к этому скрипту.
 Это сделано намеренно: после массового обновления набора скриптов событие об устаревании приходит сразу на все узлы. Если бы каждый узел в ответ немедленно компилировал все затронутые скрипты, это вызвало бы всплеск параллельных компиляций по всему кластеру. Ленивая схема растягивает нагрузку во времени: компиляция выполняется только тогда, когда конкретный скрипт реально понадобился.

Ключ `SmartScriptRoslynTimeoutMinutes` ограничивает время ожидания результата C#-скрипта. Лимит имеет минутную гранулярность и срабатывает по внешнему дедлайну: вызывающий освобождается по истечении таймаута независимо от того, пробрасывает ли сам скрипт `CancellationToken` в свои операции. Бесконечный цикл внутри скрипта больше не удерживает вызывающего до перезапуска процесса.
 Скрипт при этом не прерывается: задача помечается как брошенная (`AbandonRunawayScript` с записью в лог) и продолжает выполняться, пока не завершится сама — прервать выполняющийся `Task` извне нельзя. Таймаут защищает вызывающего, но не освобождает ресурсы: зациклившийся скрипт продолжает занимать поток до рестарта процесса.

С версии 2.268.350 у C#-смарт-скриптов появился необязательный режим асинхронного сохранения: можно включить флаг и получить ответ сразу, не дожидаясь компиляции. По умолчанию поведение не меняется — сохранение выполняет компиляцию синхронно, как раньше.

Режим включается параметром строки запроса:
 `POST /api/admin/smart/script/editor?async=true
`
- `async=false` (по умолчанию) — прежнее синхронное сохранение: запись скрипта, компиляция и ответ `200 OK`.

- `async=true` — запись скрипта без ожидания компиляции, ответ `202 Accepted`; компиляция уходит в фон, скомпилированные байты (колонки `CompiledAssembly`, `CompiledSourceHash`, `CompiledRuntimeVersion`) появляются в базе позднее.
  Сценарий применения — инструмент развёртывания, который загружает пакет скриптов после релиза: при `async=true` массовое сохранение не выстраивает компиляции в синхронную очередь, и развёртывание ускоряется в разы. Для ручного редактирования через интерфейс разницы нет — подходит стандартное `async=false`.

После асинхронного сохранения статусы запрашиваются по списку идентификаторов:
 `GET /api/admin/smart/script/compile-status?ids=10,11,12
` Ответ — словарь вида `{ идентификатор: статус }`:
 Поле Тип Описание `Status` `string` `"pending"` — в очереди или компилируется; `"ok"` — успешно скомпилировано; `"error"` — ошибка компиляции (см. `Diagnostics`); `"unknown"` — записи нет (статус истёк или скрипт не передавался в фон) `Diagnostics` `string[]?` Сообщения компилятора при `Status == "error"`, иначе `null` `CompiledAt` `DateTime?` Момент успешной компиляции при `Status == "ok"`, иначе `null` При `error` некорректный код не публикуется, прежняя рабочая версия (если была) продолжает выполняться (см. ниже). Инструмент развёртывания при статусе `error` выводит диагностику.

Ключевая особенность: если платформа обнаруживает, что исходник скрипта в базе изменился, а новые скомпилированные байты ещё не готовы (скрипт отредактирован, но фоновая компиляция не завершена) — платформа:

- Отдаёт прежние корректные байты (последнюю удачную версию) — вызов скрипта не блокируется;

- Ставит перекомпиляцию в очередь — фоновый процесс выполнит её следующей.
  До версии 2.268.350 в такой ситуации выполнялась синхронная перекомпиляция на потоке первого вызова, который и платил всю стоимость (~100–200 мс и больше). Поведение управляется настройкой `SmartScriptCompileServeLastGood` (по умолчанию `true`); установка `false` возвращает прежнее поведение с синхронной перекомпиляцией.

Поведение фоновой компиляции настраивается через CustomSettings:
 Ключ Тип Default Назначение `SmartScriptCompileParallelism` `int` `min(ProcessorCount - 1, 4)`, не меньше 1 Число одновременных фоновых компиляций. Пусто или 0 — значение по умолчанию. Требует перезапуска пула `SmartScriptCompileBackstopScanSeconds` `int` `0` (выключен) Период (в секундах) дополнительного фонового сканирования скриптов, требующих компиляции, в дополнение к стартовому. Требует перезапуска `SmartScriptCompileStatusTtlSeconds` `int` `3600` Время жизни (в секундах) записей статуса `ok`/`error`. По истечении статус становится `unknown` `SmartScriptCompileServeLastGood` `bool` `true` См. выше. `false` — прежнее поведение (синхронная перекомпиляция при изменённом исходнике)

Самый короткий рабочий C#-скрипт:
 `// v1 | 2026-04-23 12:00 | hello world
LOG.AddAutomationLog("hello from C#");
RESULT = new { ok = true, message = "hello" };
` `RESULT` — запасной вариант, совместимый с Lua и JavaScript. Нативный вариант — `return`:
 `return new { ok = true, message = "hello" };
` Приоритет: нативный `return` выигрывает у `RESULT`. `RESULT` используется только если скрипт не вернул значение через `return`.
 Скрипт пишут и запускают в редакторе смарт-скриптов (раздел администрирования), выбрав в нём язык C#.

Перечисленные ниже имена доступны в скрипте глобально — импортировать ничего не нужно.
 `// Execution context
IContext Context              // сессия, userId, кэш
dynamic CONTEXT                // бизнес-сущность запуска: задача, письмо, пользователь, очередь
                                // сообщений или документ; null без контекста. Не путать с Context
                                // (IContext) выше — тот про сессию/права/культуру исполнения.
ILifetimeScope Scope           // Autofac scope — для ad-hoc Resolve<T>()
IReadOnlyDictionary<object,object> EVENTPARAMS  // параметры события
object SESSION_USER            // текущий пользователь
string DB_TYPE                 // "MSSQL" | "PG"
object SYSTEM_INFO

// Platform API (те же имена что в JS/Lua — для консистентности)
CSharpSqlApi SQL
CSharpHttpApi HTTP
CSharpCacheApi CACHE
CSharpUtilsApi UTILS
CSharpLibApi LIB
CSharpLogApi LOG
CSharpSmartApi SMART

CSharpFilesApi FILES
CSharpMetaApi META
CSharpRegistryApi REGISTRY
CSharpVectorDbApi VECTOR_DB

// Параметры события (паритет с JS top-level globals)
IReadOnlyDictionary EXTRAPARAMS

// SSE streaming для агентских/потоковых сценариев
Action STREAM_EMIT
Func STREAM_CANCELLED

// Запасной вариант в стиле Lua/JS (когда нативный return неудобен)
object RESULT { get; set; }

// Output для редактора и логов
void print(object message)
`

`CONTEXT` и `Context` — разные сущности, различаются регистром:
 Имя Тип Что это Когда null `Context` `IContext` Сессия: текущий пользователь, impersonation, культура, access levels Не бывает null в нормальном запуске `CONTEXT` `dynamic` Бизнес-объект: задача, письмо, пользователь, очередь сообщений, документ При запуске без контекста (например, из редактора) Обращение к `CONTEXT.Id` без null-check в contextless-запуске приведёт к `Microsoft.CSharp.RuntimeBinder.RuntimeBinderException`. Скрипт, который может запускаться и с контекстом, и без, должен начинаться с проверки:
 `if (CONTEXT == null) {
    RESULT = new { skipped = true, reason = "no context" };
    return;
}
var taskId = (int)CONTEXT.Id;
` DI-шлюз для продвинутых случаев:
 `// Резолв произвольного сервиса из Autofac-скоупа
var provider = Resolve<IHostingFacade>();

// Property-injection в собственные классы (декларируем в скрипте)
public class MyHelper {
    public IDbService Db { protected get; set; }
    public int CountTasks() =>
        Convert.ToInt32(Db.GetScalarQueryResult<object>("SELECT COUNT(*) FROM Tasks"));
}
var helper = InjectProperties(new MyHelper());
var n = helper.CountTasks();
` Поиск MCP-инструментов. C# SmartScript может найти подходящий MCP-инструмент платформы через `Resolve<TCClassLib.Mcp.IMcpToolSearch>()`. Контракт возвращает структурированный список `McpToolSearchHit` (поля: `ServerPath`, `ToolName`, `Description`, `Score`) и делегирует ранжирование в `McpToolSearchIndex`. Подробнее — domains/ai/backend.md, раздел «Нативный доступ из SmartScript: `IMcpToolSearch`».

Полный паритет с JS SMART API — те же методы, типизированные.
 `// Выполнить action
object ExecuteAction(string action, int? contextId, string contextType,
    Dictionary<string, object> @params = null, bool async = false)
object ExecuteActionInTaskContext(string action, int? contextTaskId, ...)
object ExecuteActionInUserContext(string action, int? contextUserId, ...)
object ExecuteActionInEmailContext(string action, int? contextEmailId, ...)

// Запустить смарт-скрипт в фоне (без возврата результата, RESULT теряется)
void RunScriptBackground(object smartScriptIdOrName, int? contextId = null,
    object[] eventParams = null, Dictionary<string, object> extraParams = null,
    bool ignoreScriptEvent = false)

// Запустить SmartScript асинхронно с возвратом RESULT (рекомендуется)
async Task<object> RunScriptSyncAsync(object smartScriptIdOrName, int? contextId = null,
    string contextType = "task", object[] eventParams = null,
    Dictionary<string, object> extraParams = null, bool ignoreScriptEvent = false)

// Запустить SmartScript синхронно с возвратом RESULT (блокирующая обёртка над RunScriptSyncAsync)
object RunScriptSync(object smartScriptIdOrName, int? contextId = null,
    string contextType = "task", object[] eventParams = null,
    Dictionary<string, object> extraParams = null, bool ignoreScriptEvent = false)
` `contextType`: `"task"|"user"|"email"|"edocumentlink"|"edocumentlinksbis"`. Default `"task"`.
 `RunScriptSyncAsync` — асинхронный вариант с нативным `await`: не блокирует поток на время выполнения дочернего скрипта (включая его внутренние HTTP- и LLM-вызовы). `RunScriptSync` — блокирующая обёртка над ним, оставлена для обратной совместимости. Каскад вложенных вызовов в обоих случаях ограничен 3 уровнями. Между скриптами возможен типизированный обмен: вызывающий C#-скрипт может объявить `record` для результата и сериализовать/десериализовать его без ручного разбора JSON.
 `// Пример: вызов job-скрипта с получением типизированного результата
var raw = SMART.RunScriptSync("anfisa-job", taskId, "task", null,
    new Dictionary<string, object> { ["Prompt"] = "суммаризируй", ["UserId"] = 28 },
    ignoreScriptEvent: true);

var result = JsonConvert.DeserializeObject<JobResult>(JsonConvert.SerializeObject(raw));
return new { ok = result.Ok, answer = result.Answer };

public record JobResult(bool Ok, string Answer, int TokensUsed);
` Когда что использовать: | Нужно | Метод | |-------|-------| | Выполнить SS в фоне, RESULT не нужен | `SMART.RunScriptBackground` | | Получить RESULT синхронно, передать extraParams (блокирующий контекст / migration-обёртка) | `SMART.RunScriptSync` | | Получить RESULT, не блокируя поток на время выполнения child (LLM/HTTP вызовы) | `SMART.RunScriptSyncAsync` | | Выполнить в том же контексте (общие глобальные имена) | `LIB.Include(name)` / `Include(id)` | | Выполнить action (не SmartScript) | `SMART.ExecuteAction*` |

Типизированный доступ к кэшу настроек через `Resolve<T>()` — не через прямой SQL. Прямой SQL-запрос к `SettingsCustom` обходит кэш (TTL 1 час) и приводит к лишнему обращению к базе на каждый вызов.

Готовый шаблон для чтения настроек в начале скрипта:
 `// Одна инициализация в начале скрипта
var _csCache = Resolve<TCDataAccess.Caching.InMemory.CustomSettingsCache>();

// Helper — обёртывает cold-cache quirk (см. ниже)
string GetSettingValue(string key)
{
    var k = new Valhalla.Integration.Dto.Settings.CustomSettingKey(key, null);
    var entity = _csCache.Get(k) ?? _csCache.Invalidate(k);
    return entity?.Value;
}

// User-scope вариант
string GetSettingValueForUser(string key, int userId)
{
    var k = new Valhalla.Integration.Dto.Settings.CustomSettingKey(key, userId);
    var entity = _csCache.Get(k) ?? _csCache.Invalidate(k);
    return entity?.Value;
}

// Использование
var apiKey = GetSettingValue("anfisa_anthropic_key");
var userTheme = GetSettingValueForUser("user_theme", 28);
`

`CustomSettingsCache` имеет `InvalidateIfGetIsEmpty = false` — при первом `Get(key)` без прогрева возвращает `null` без lookup в БД. В прод-ядре кэш прогревается через API-записи (`POST /api-core/data-source-form/72/action` → `Invalidate` ключа), поэтому `Get(key)` обычно уже горячий.
 На стендах без фоновых заданий (например, при `IsMainServer=false`) кэш холодный — `Get(key)` возвращает null даже если запись в БД есть. Fallback `Get ?? Invalidate` делает force-load из БД при пустом кэше и сам кэширует — один лишний DB round-trip только на первом обращении.

Антипаттерны при работе с настройками и их замена:
 ❌ ✅ `SQL.scalar("SELECT Value FROM SettingsCustom WHERE Key=@k", ...)` `GetSettingValue("key")` `UTILS.resolve_instance("TCDataAccess", "...CustomSettingsCache")` (JS-паттерн) `Resolve<CustomSettingsCache>()` Цикл с повторными `Resolve<CustomSettingsCache>()` на каждый `Get` Один раз в начале + переиспользовать `_csCache`

Имена методов snake_case — симметрично `JsSqlApi` (чтобы порт JS↔C# шёл один-в-один). Все возвращают `object` (не generic), типизируй через `Convert.ToXxx` или cast.
 `// object scalar(string sql, IDictionary<string, object> params)
var count = Convert.ToInt32(SQL.scalar(
    "SELECT COUNT(*) FROM Tasks WHERE UserID = @id",
    new Dictionary<string, object> { ["id"] = 28 }));

// object query(string sql, IDictionary<string, object> params) → List<Dictionary<string, object>>
var rows = SQL.query("SELECT TaskID, Description FROM Tasks WHERE UserID = @id",
    new Dictionary<string, object> { ["id"] = 28 });
// Каждая строка — Dictionary<string, object>. Парсить вручную или прогнать через JsonConvert:
var typed = JsonConvert.DeserializeObject<List<TaskRow>>(JsonConvert.SerializeObject(rows));

// object query_one(string sql, params) — первая строка или null
var one = SQL.query_one("SELECT Description FROM Tasks WHERE TaskID = @id",
    new Dictionary<string, object> { ["id"] = 2078773 });

// int exec(string sql, params) — для INSERT/UPDATE/DELETE, возвращает affected rows
var affected = SQL.exec("UPDATE Tasks SET PriorityID = @p WHERE TaskID = @id",
    new Dictionary<string, object> { ["p"] = 3, ["id"] = 2078773 });

public record TaskRow(int TaskID, string Description);
` Почему не generic `scalar<int>()`: `CSharpSqlApi` зеркалит контракт `JsSqlApi` для единообразия имён при портировании. Для типизации используй `Convert.ToXxx` или явный cast. Если хочется generic-API — резолви `TCMain.DB` напрямую через `Resolve<IDbService>()` + используй `GetScalarQueryResult<T>(sql, params)`.
 DB-агностичность обязательна (как для всех языков SS). `CSharpSqlApi` не делает автоматической подмены MS↔PG — пиши DB-совместимый SQL или используй SQL Dialect Helpers (см. ниже). Нельзя raw `[Key]`, `GETDATE()`, `LEN()`, `ISNULL()`, `TOP N`, `N'...'` без помощников.

Время ожидания SQL-запросов в смарт-скрипте ограничивается таймаутом (в секундах). Работает одинаково в C#, JS и Lua.

- Дефолт на весь скрипт — свойство `SQL.DefaultTimeoutSec` или метод `SQL.set_timeout(N)` (в Lua — `SQL:set_timeout(N)`). Задаёт таймаут по умолчанию для всех последующих запросов скрипта.

- Таймаут на один вызов — необязательный третий аргумент методов `scalar`, `query`, `query_one`, `exec`: `SQL.scalar(sql, params, 60)` — не более 60 секунд для этого запроса.
  Приоритет: явный таймаут вызова перебивает `DefaultTimeoutSec`; если ни то ни другое не задано — действует драйверный дефолт 600 секунд. Значение `0` означает «без лимита», отрицательное запрещено.

`SQL.*` методы возвращают готовый SQL-фрагмент для подстановки в `$"..."`. Логика диалекта задана в одном месте и не дублируется. Идентификаторы для PG автоматически приводятся к нижнему регистру (`SQL.Qi("Value")` → `"value"`).
 `// Пагинация (предпочитай ANSI-вариант — одинаковый фрагмент для MS/PG):
var sql = $@"SELECT TaskID FROM Tasks WHERE UserID = @id
             ORDER BY TaskID DESC
             {SQL.FetchFirst(10)}";

// Идентификатор + IfNull + текущее время:
var sql2 = $@"SELECT {SQL.Qi("Value")}, {SQL.IfNull("Comment", "''")}, {SQL.Now()} FROM SettingsCustom";

// Lateral / APPLY (парный с LateralOn):
var sql3 = $@"SELECT t.TaskID, c.Color
              FROM Tasks t
              {SQL.OuterApply()} dbo.fn_TaskColor(t.TaskID, @userId) c {SQL.LateralOn()}
              WHERE t.OwnerID = @userId";

// INSERT … OUTPUT/RETURNING (парный):
var newId = Convert.ToInt32(SQL.scalar($@"
    INSERT INTO Logs (Msg) {SQL.OutputInsertedId("Id")}
    VALUES (@m) {SQL.ReturningId("Id")}",
    new Dictionary<string, object> { ["m"] = "test" }));

// DateAdd с типом единицы (предпочтительный вариант):
var sql4 = $"SELECT * FROM Tasks WHERE Created > {SQL.DateAdd(SqlDateUnit.Hour, -2)}";

// Длинный текст:
var sql5 = $"SELECT {SQL.CastText("Body", 400)} FROM Comments";
`

Полный перечень методов и их раскрытие для каждой СУБД:
 Группа Метод MS PG Идентификаторы `Qi(name)` `[name]` `"name"` (lowercase!) Время `Now()` / `NowFn()` `GETDATE()` `NOW()` / `now()` UUID `NewId()` `NEWID()` `gen_random_uuid()` Пагинация `Top(n)` `TOP n` `` `Limit(n)` `` `LIMIT n` `FetchFirst(n)` `OFFSET 0 ROWS FETCH NEXT n ROWS ONLY` то же Hint `Nolock()` `WITH (NOLOCK)` `` Скаляры `IfNull(e, d)` `ISNULL(e, d)` `COALESCE(e, d)` `Len(e)` `LEN(e)` `LENGTH(e)` `Concat(a, b, ...)` `a + b + ...` `a \|\| b \|\| ...` Даты `DateAdd(SqlDateUnit, amount, baseExpr=null)` `DATEADD(UNIT, amount, base)` `(base ± INTERVAL 'amount units')` `DateAddParam(SqlDateUnit, "@p", baseExpr=null)` `DATEADD(UNIT, @p, base)` `(base + (@p) * INTERVAL '1 unit')` `DateDiffSeconds(a, b)` `DATEDIFF(SECOND, a, b)` `EXTRACT(EPOCH FROM (b - a))` `FormatDate(e, SqlDateFormat)` `CONVERT(VARCHAR(N), e, style)` `to_char(e, 'pattern')` Касты `CastText(e, maxLen?)` `CAST(e AS NVARCHAR(maxLen\|MAX))` `CAST(e AS TEXT)` (или `LEFT(...)`) `CastVarchar(e)` `e` `e::varchar` `CastInt(e)` `CAST(e AS INT)` `e::integer` `CastUuid(e)` `e` `e::uuid` LATERAL `CrossApply()` `CROSS APPLY` `CROSS JOIN LATERAL` `OuterApply()` `OUTER APPLY` `LEFT JOIN LATERAL` `LateralOn()` `` `ON TRUE` Recursive CTE `WithRecursive(name)` `WITH name` `WITH RECURSIVE name` `RecursiveKeyword()` `` `RECURSIVE` INSERT `OutputInsertedId(col)` `OUTPUT inserted.col` `` `ReturningId(col)` `` `RETURNING col` `SqlDateUnit`: `Second`, `Minute`, `Hour`, `Day`, `Month`, `Year`. `SqlDateFormat`: `Iso` (`yyyy-MM-dd HH:mm:ss`), `IsoDate` (`yyyy-MM-dd`), `RuDate` (`dd.MM.yyyy`), `RuDateTime` (`dd.MM.yyyy HH:mm:ss`).

Существующие методы `SQL.Qi`, `SQL.QuoteIdent`, `SQL.NowFn`, `SQL.IfNull`, `SQL.CastText(expr)`, `SQL.DateAdd(string part, ...)`, `SQL.NewId` оставлены без изменений — старые скрипты продолжают работать. Новый код используй через таблицу выше; для `DateAdd` предпочитай `SqlDateUnit`-перегрузку.

В прикладном коде ядра доступны через extension methods на `IValhallaDataConnection` (`IValhallaDataConnection.Extensions.cs`, `db.IsPostgreDB`) и на `IDBHandler` (`IDBHandlerSqlDialectExtensions.cs`, `TCMain.DB.IsPostgre`). Все три точки входа делегируют на единый `SqlDialect`.

Имена методов тоже snake_case (паритет с Jint):
 `// object send_http_request(method, url, params, headers, rawBody, options)
// или async-версия:
var resp = await HTTP.send_http_request_async("POST", "https://...",
    @params: null,
    headers: new Dictionary<string, object> { ["Authorization"] = "Bearer …" },
    rawBody: JsonConvert.SerializeObject(new { q = "test" }),
    options: new Dictionary<string, object> { ["Timeout"] = 30000 });

// Типизированный альтернативный путь:
var typed = await HTTP.post_async("https://...", new ScriptHttpRequestOptions {
    RawBody = JsonConvert.SerializeObject(new { q = "test" }),
    Timeout = 30000
});
// typed — ScriptHttpResponse, с полями .StatusCode, .Body, .Headers
` Полный паритет с Jint-версией для snake_case; дополнительно — типизированный `request`/`get`/`post`/`request_async` для удобства C#.

«Простые» запросы (без авторизации, прокси и учётных данных) по умолчанию идут через общий пул HTTP-клиентов — общее хранилище cookie запоминает `Set-Cookie` и подставляет cookie на следующих запросах к тому же домену в течение ~2 минут. Если внешний сервис должен отдавать новую сессионную cookie на каждый вызов (ротация сессий, лимит параллельных соединений на учётку), включите `DisableCookieContainer`:
 `var resp = await HTTP.send_http_request_async("POST", url,
    @params: null, headers: headers, rawBody: rawBody,
    options: new ScriptHttpRequestOptions { DisableCookieContainer = true });
` Опция исключает запрос из общего пула и направляет его через одноразовый HTTP-клиент без хранения cookie. Полное описание и сравнение с Lua/JS-Jint ключами: `admin.md` § «HTTP-опции: изоляция cookie-контейнера».

Имена методов в snake_case. Кэш локален для процесса и общий для всех языков смарт-скриптов (Lua, JS, C#).
 `CACHE.set("anfisa:sess:123", "payload", lifetime: 3600);  // lifetime в секундах
var v = CACHE.get("anfisa:sess:123");                      // → string или null
CACHE.delete("anfisa:sess:123");
` Используй для кэша внутри одного SS или share между SS. Не для SettingsCustom — там свой кэш через `CustomSettingsCache` (см. §SettingsCustom API).

`CSharpUtilsApi` намеренно тонкий. В C# полный BCL доступен напрямую, не нужно wrapper'ов для стандартных вещей.
 `// Всё что есть в CSharpUtilsApi
UTILS.Print("progress");        // → output-буфер, виден в /editor/execute response
var empty = UTILS.IsEmpty(value); // null / "" / IEnumerable с Count=0 → true

// Глобальный shortcut: print(msg) на SmartScriptGlobals = UTILS.Print без префикса
print("hello");
`

Соответствие JS-функций `UTILS.*` и средств C#:
 JS C# `UTILS.new_guid()` `Guid.NewGuid().ToString()` `UTILS.json_decode(s)` `JsonConvert.DeserializeObject<T>(s)` или `JObject.Parse(s)` `UTILS.json_encode(o)` `JsonConvert.SerializeObject(o)` `UTILS.trim_to_tokens(text, n)` нет стандартного аналога — писать руками по нужде `UTILS.resolve_instance(asm, type)` `Resolve<T>()` (typed) или `Scope.Resolve(Type.GetType($"{type}, {asm}"))` (reflection fallback) `UTILS.create_instance(asm, type)` `new T(...)` (typed) или `Activator.CreateInstance(Type.GetType(...))`

Единственный метод, паритет с `JsLogApi.AddAutomationLog`:
 `LOG.AddAutomationLog("heartbeat: alive");
LOG.AddAutomationLog("failed", parameters: $"ex={ex.Message}, stack={ex.StackTrace}");
` Пишет в `AutomationScriptsLog` (form 91). `objectKey` = ID текущего SS, `objectName` = Description. В JS было `SS.logError`/`SS.logInfo` — в C# один метод, разделения по severity нет (использовать префикс в `message` если нужно: `"[ERROR] ..."`).

В C# SS есть четыре способа «подключить что-то» — с разной семантикой. Не путать.
 Форма Пример Что даёт `using Namespace;` `using System.Text.RegularExpressions;` Стандартный C#-импорт пространства имён. Требует что сборка содержащая этот namespace уже в `TrustedAssemblies` (или `TrustedImports` уже содержит namespace) `#r "Assembly"` `#r "System.Xml"` Директива Roslyn — добавить сборку на уровне конкретного скрипта. Например, `#r "System.Xml"; using System.Xml; var d = new XmlDocument();` компилируется без изменения списка разрешённых сборок. Полезно для редко используемых стандартных сборок `#load "file.csx"` `#load "helper.csx"` Не работает — default-resolver ищет в файловой системе, а у нас скрипты в БД (CS1504 при попытке). Для подключения другого скрипта использовать `Include(name)` `Include(name\|id)` / `SMART.RunScriptSync(...)` `await Include("my-lib")` Наш механизм кросс-скриптовых вызовов — см. ниже

`#r "AssemblyName"` работает с стандартными .NET сборками в GAC/runtime (типа `System.Xml`, `System.IO.Compression`). Для custom сборок платформы путь только через `TrustedAssemblies` или `SmartScriptRoslynAdditionalAssemblies` — `#r "TCDataAccess"` не даст runtime-обход, т.к. это не GAC-resolvable имя.
 Это полезно знать: для BCL-сборок которые не в текущем TrustedAssemblies можно взять через `#r` один раз в конкретном скрипте вместо модификации ядра. Но для платформенных сервисов — только нормальный путь (trusted + `Resolve<T>`).

Два разных механизма с разной семантикой. Не путать.

Подключение библиотечного скрипта по имени или идентификатору:
 `var result = await LIB.Include("helper-script-name");    // by Description
var result = await LIB.Include(642);                      // by Id
var result = await Include("helper");                     // shortcut на SmartScriptGlobals
` Что это: подключить C#-скрипт как библиотеку. Глобальные имена (в т.ч. `RESULT`, `SQL`, `HTTP`, `CACHE`, `Context`) общие с вызывающим скриптом. Библиотечный скрипт компилируется один раз и выполняется в том же контексте, что и вызывающий.
 Требования к целевому скрипту:

- `IsLibrary = true` — иначе `TCLogicException("Скрипт ... не является библиотечным")`

- `LanguageId = 4` (CSharp) — вызов между разными языками (JS ↔ C#) через `Include` пока не поддерживается (ошибка `TCLogicException("Cross-language include is not supported in MVP")`)

- Существует (искомый по name/id) — иначе `EntityNotFoundException` / `TCLogicException("не найден")`
  Возвращает: значение `return` целевого скрипта (или `null`, если `return` нет). Во время выполнения это число, строка, анонимный объект или словарь.

Что фактически доступно целевому скрипту:
 Что Общее? `RESULT` ✅ да — целевой скрипт может перезаписать, вызывающий увидит `SQL` / `HTTP` / `CACHE` / … ✅ да — тот же экземпляр `Context`, `Scope` ✅ да Локальные переменные целевого скрипта (`var libraryVariable = 10;`) ❌ нет — это разные блоки компиляции, при попытке доступа будет ошибка CS0103 Иллюстрация: `// вызывающий скрипт
RESULT = "caller-initial";
var retVal = await Include("lib");     // внутри целевого: RESULT = "from lib"; return "ret val";
// После Include:
// retVal = "ret val"
// RESULT = "from lib"  (библиотека перезаписала!)
`
 Вывод: использовать библиотечный скрипт как чистую функцию с возвращаемым значением, не полагаясь на общее состояние через RESULT (если это явно не требуется).

Ограничения на вложенные и рекурсивные вызовы:

- Loop detection: если один script id встречается в стеке 5 раз подряд — `TCLogicException("Рекурсия в include() глубже 5: <trace>")`. Именно loop (same id), не numeric depth — линейная цепочка A→B→C→D→E→F→... не упадёт по этому правилу.

- Таймаут выполнения — 15 минут по умолчанию (настраивается, см. раздел «Изоляция»). Линейная цепочка упрётся сюда, если скрипты медленные.

Альтернативный механизм:
 `var raw = SMART.RunScriptSync(
    "anfisa-job",                                    // id или name
    contextId: taskId,
    contextType: "task",                              // task|user|email|...
    eventParams: null,
    extraParams: new Dictionary<string, object> {
        ["Prompt"] = "суммаризируй",
        ["UserId"] = 28
    },
    ignoreScriptEvent: true);

// raw — это RESULT целевого скрипта; для типизации десериализуйте:
var typed = JsonConvert.DeserializeObject<JobResult>(JsonConvert.SerializeObject(raw));
` Отличия от `Include`:
 Aspect `Include(name)` `SMART.RunScriptSync(name, ...)` Язык целевого скрипта Только C# Любой (JS, Lua, OneScript, C#) `IsLibrary=true` обязательно ✅ да ❌ нет Контекст Общий с вызывающим (его RESULT перезаписывается!) Дочерний, изолированный — собственный DI и собственные глобальные имена Параметры Нет — через общие глобальные имена `eventParams` + `extraParams` (→ `EXTRA_PARAMS` в целевом скрипте) Возвращаемое значение Значение `return` `RESULT` или `return` целевого скрипта Асинхронность Нативный `await Include(...)` Внутри выполняется синхронно (но вызывающий C#-скрипт может писать `var x = SMART.RunScriptSync(...)` без await) Рекурсия Защита от цикла на глубине 5 (один и тот же id) Ограничение глубины вложенности — 3 Применение Вспомогательные функции, общие и кросс-СУБД SQL-хелперы Типизированный вызов между модулями, мост между языками (вызывающий C#, целевой JS), изоляция ошибок

Include — когда:

- Целевой скрипт — чистая утилита на C#, переиспользуемая несколькими скриптами

- Нужна скорость (без накладных расходов на дочерний контекст)

- Допустимо, что целевой скрипт перезапишет `RESULT`
  RunScriptSync — когда:

- Целевой скрипт на другом языке (например, вызвать JS из C#)

- Нужна изоляция (дочерний контекст с собственными глобальными именами)

- Нужны параметры через `EXTRA_PARAMS`

- Нужен типизированный контракт «вызывающий → `JobResult`»
  RunScriptBackground — когда:

- Запуск без ожидания результата (результат не нужен)

- Делегирование подзадач, сценарии с очередями сообщений

- Не блокировать вызывающий скрипт

Какие сочетания языков поддерживаются для каждого механизма:
 From → To `Include` / `include()` `SMART.run_script_sync` / `SMART.RunScriptSync` JS → JS ✅ ✅ JS → C# ❌ Jint парсит C# как JS (`Unexpected token`) ✅ C# anonymous record → JS object C# → JS ❌ `Cross-language include is not supported in MVP` ✅ JS object → C# `ExpandoObject` (dynamic) C# → C# ✅ общий контекст ✅ дочерний контекст Правило: `Include`/`include()` работает только в пределах одного языка. Для вызовов между разными языками используйте `SMART.run_script_sync` — он связывает все 5 языков смарт-скриптов.

Возможность вызывать скрипты на другом языке через `run_script_sync` означает, что перенос не блокирует интеграцию:
 Место в JS-коде Новая стратегия при миграции этого скрипта на C# `include("x-js-lib")`, где `x-js-lib` тоже переносим Замена на `await Include("x-js-lib")` (оба на C#, общий контекст) `include("x-js-lib")`, где целевой скрипт остаётся на JS Замена на `SMART.RunScriptSync("x-js-lib", null, "task", null, null, true)` (дочерний контекст) `SMART.run_script_background("y-js", ...)` `SMART.RunScriptBackground("y-js", ...)` — между языками тоже работает (без ожидания результата) И наоборот — вызывающий JS-скрипт может обращаться к новым C#-хелперам через `SMART.run_script_sync` ещё до того, как сам будет перенесён. Это и позволяет переносить модули на C# по одному, не ломая существующие JS-вызовы.

Первый вызов — платформа компилирует скрипт, ~100–200 мс. Последующие вызовы — берут готовый код из кэша (ключ — содержимое скрипта), выполнение на скорости обычного C#.
 При изменении текста скрипта он компилируется заново, прежняя версия вытесняется из кэша.

Сборки (whitelist) — 16 assemblies:
 Assembly Кому/зачем `System.Private.CoreLib` `object`, всё System.* primitive `System.Linq` LINQ — `Enumerable`, `Queryable`, `Where/Select/…` `System.Threading.Tasks` `Task`, `async/await`, `Task.WhenAll` `System.Net.Http` `HttpClient` (если нужен bypass HTTP API) `Newtonsoft.Json` `JsonConvert`, `JObject` `Valhalla.Interfaces` / `TCInterfaces` `IContext`, `IDependency`, сигнатуры `Valhalla.Scripting` `SmartScriptGlobals` + все `CSharpApi/*` `TCClassLib` Классы бизнес-логики `TCClassLib` (`Roles` и др.) `Valhalla.Integration` `ScriptLanguage`, `CustomSettingKey`, DTO `TCDataAccess` (+2026-04-23) `CustomSettingsCache`, `UsersCache`, `CategoriesCache`, `SubcategoriesCache`, etc — все кэши, сервисы сущностей Utils assembly (+2026-04-23) Общие helpers (`TypeExtensions` и др.) TCClassLib.Orm (+2026-04-23) `TcContext` (EF) — прямая работа с ORM, `Tasks`/`Users`/`Comments` DbSet `Valhalla.ExternalServices` (+2026-04-23) `ExternalServicesConfigurationServiceBase<,>` и т.п. `Valhalla.Configuration` `Configuration` — доступ к конфигурации приложения `Microsoft.CSharp` Поддержка `dynamic` keyword и runtime-биндинга `System.Linq.Expressions` `DynamicAttribute`, `System.Dynamic.*` — нужна в паре с `Microsoft.CSharp` для `dynamic` Imports по умолчанию (не надо писать `using`):
 `System
System.Linq
System.Threading.Tasks
System.Collections.Generic
Newtonsoft.Json
System.Text              (+2026-04-23 — StringBuilder, Encoding)
Valhalla.Scripting.Engine
` Runtime-расширения — администратор может добавить дополнительные сборки через `SettingsCustom.SmartScriptRoslynAdditionalAssemblies` (CSV путей). Ограничено директорией `{AppDir}/SmartScriptExtensions/` — чтобы нельзя было подгрузить произвольный DLL с диска.
 Прогрев компилятора — `SettingsCustom.SmartScriptRoslynWarmUpScriptIds` (CSV из ID скриптов): перечисленные скрипты компилируются заранее при старте приложения, чтобы их первый вызов не платил холодный старт Roslyn.

Текущий режим (доверенный) — таймаут плюс список разрешённых сборок. Таймаут по умолчанию — 15 минут; переопределяется настройкой `SettingsCustom.SmartScriptRoslynTimeoutMinutes` (лимит выполнения C#-скрипта в минутах).
 Строгая изоляция (в планах) — анализатор кода будет запрещать обращения к файловой системе, процессам, рефлексии и загрузке сборок. Понадобится, когда возможность писать C#-скрипты получат внешние пользователи и внедренцы.

Где искать причину типичных ошибок выполнения:
 Симптом Где смотреть Ошибка компиляции `CS****` Ответ редактора скриптов / журнал `AutomationScriptsLog` Таймаут Запись о прерывании в журнале `AutomationScriptsLog` `TCLogicException("Превышена максимальная глубина синхронных вызовов SmartScript (3)")` Каскад `SMART.RunScriptSync` → подрезать глубину, использовать `RunScriptBackground` для async-путей Неожиданный `null` от `RunScriptSync` Целевой скрипт не выставил RESULT и не сделал `return`. Проверить целевой скрипт

Ориентир по выбору языка под задачу:
 Критерий JS (Jint) C# (Roslyn) Много операций со строками, перекладывание JSON ✅ ─ Частые правки (промпты, маршрутизация, экспериментальные сценарии) ✅ ─ Короткий (<100 строк) ✅ ─ (накладные расходы на типизацию не оправданы) Часто меняющаяся по структуре логика ✅ ─ (типы приходится менять вместе с логикой) Типизированные контракты (LLM response, mailbox) ─ ✅ Async/параллельность (`Task.WhenAll`) ─ ✅ Прямой DI к сервисам платформы ─ ✅ Сложная state machine ─ ✅ 5+ вызовов платформенных сервисов ─ ✅ (подсказки IDE по именам)

Частые ошибки компиляции C#-скриптов и их причины:
 CS-код Причина Fix CS0234 `Тип не существует в пространстве имён` Использован тип из assembly вне TrustedAssemblies Проверить список разрешённых сборок в разделе «Что доступно из коробки» CS1061 `Тип не содержит определения X` Неверное имя метода на платформенном API Проверить точное имя метода в справочнике API (snake_case или PascalCase) CS4014 `Вызов не ожидается (missing await)` `Include(...)` / `SQL.scalar(...)` без await Добавить `await` или явно игнорировать Task CS0103 `Имя не существует в текущем контексте` Попытка обратиться к переменной из другого скрипта (библиотечного из вызывающего) Передавать через RESULT или через параметры `RunScriptSync` CS0246 `Тип не найден` Часто с `Resolve<T>()` где T из non-trusted asm То же что CS0234

Перед написанием SQL проверяйте фактические имена колонок в структуре таблиц — они часто отличаются от ожидаемых:
 Таблица Не существует Реально `SmartScripts` `IsDisabled`, `Disabled` Нет такой колонки — SS не disabled'ятся на уровне таблицы `Comments` `CommentId` (camelCase) `CommentID` (CAP) `Comments` `UserId` `UserID` `Tasks` `Name`, `Title`, `TaskText` `Description` (varchar max) `Subcategories` `Name` `Description` (иногда) или `CategoryName` `SettingsCustom` `SettingKey`, `SettingValue` `Key` и `Value`
