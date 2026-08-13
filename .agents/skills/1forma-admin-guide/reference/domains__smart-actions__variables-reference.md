# Справочник переменных смарт-действий

Источник: https://help.1forma.ru/domains/smart-actions/variables-reference/

Переменные вида `@ContextID`, `@EventParam0`, `@cycle_value` и т.д., которые подставляются в SmartExpressions (T-SQL / ESQL), смарт-фильтрах, CustomFlex SQL. Одно место для поиска вместо обхода трёх разных доменных файлов.
 Переменные `@authorId`, `@subcatId` в таблице ниже — это SQL-параметры хранимых процедур и JS-скриптов, а не подстановочные переменные платформы. Они включены со сноской для полноты.

Подстановочные переменные платформы, доступные в смарт-выражениях:
 Переменная Регистр Тип Значение Где доступна `@ContextID` регистронезависим (`@ContextId` = то же) int ID текущей задачи (TaskID) SmartExpression T-SQL, ESQL-обёртка `WrapCommandTextInContext()` `@EventParam0` регистронезависим зависит от события Параметр 0 текущего события; для события подписи — ID задачи-акцептанта SmartExpression с привязкой к событию (`eventId != null`) `@EventParam1` регистронезависим зависит от события Параметр 1 события; для события «открытие задачи» — контекст открытия; для ДП Lookup — ID текущей категории SmartExpression с привязкой к событию; ДП Lookup смарт-фильтр `@EventParam2` регистронезависим зависит от события Параметр 2 события; для ДП Lookup — ID задачи-источника; для портала (каскадный dropdown) — выбранное значение родительского параметра SmartExpression с привязкой к событию; каскадные dropdown в портале `@EventParam3` регистронезависим string (JSON) Параметр 3 события; для ДП Lookup, а также SelectUsers/SelectGroups/SelectOrgUnits — JSON значений строки таблицы (если ДП находится в колонке ДП Таблица) ДП Lookup, SelectUsers, SelectGroups, SelectOrgUnits — смарт-фильтр `@EventParam4` регистронезависим int Параметр 4 события; для ДП Lookup — ID блока ДП Lookup смарт-фильтр `@EventParam{N}` регистронезависим зависит от события Параметр с индексом N (от 0); именуются `@EventParam0`, `@EventParam1`, ... SmartExpression с eventId; T-SQL — через `AddNeededQueryParams()` по тексту `@eventParamRaw{N}` регистронезависим любой (TVP-compatible) «Сырой» вариант параметра события N; поддерживает TVP для `IEnumerable<int/string/DateTime>` SmartExpression T-SQL (только в режиме T-SQL, не в визуальном конструкторе) `@cycle_value` строчный string Текущее значение итератора в цикле пакета действий SmartExpression внутри циклического пакета (`IsCyclic = true`) `@cycle_idx` строчный int Индекс текущей итерации (от 0) SmartExpression внутри циклического пакета `@CurrentSessionUserID` строчный int ID текущего пользователя (сессионный); в ShowTasksFeed текстово заменяется на `@UserID` до исполнения SmartExpression T-SQL; CustomFlex SQL в ShowTasksFeed ### Примечание: переменные в JS-скриптах и SP (не SMART-переменные платформы) Следующие идентификаторы встречаются в SQL-параметрах SP и JS-скриптах Анфисы — это обычные SQL-параметры, не «подстановочные» переменные платформы SmartExpressions:
 Идентификатор Контекст `@authorId` SQL-параметр в JS-скриптах Анфисы (agent-tools-user.js, agent-persist.js) `@subcatId` SQL-параметр SP и JS-фильтров Анфисы `@SubcatID` SQL-параметр большинства системных SP (ShowTasksFeed, tc_SubcategoryDenormalize и др.) `@UserID` SQL-параметр ShowTasksFeed; итоговое имя после замены `@CurrentSessionUserID` ---

T-SQL ветка (TSqlContent)
 Платформа сканирует текст T-SQL на наличие известных имён параметров и подставляет их значения. Распознаются:

- `@EventParam0`..`@EventParamN` — параметры события;

- `@eventParamRaw0`..`@eventParamRawN` — «сырые» параметры события;

- `@cycle_value`, `@cycle_idx` — переменные цикла;

- именованные переменные из результатов предыдущих действий пакета.
  ESQL ветка (XML Content)
 Из XML визуального конструктора формируется ESQL-строка, которая оборачивается в контекст текущей задачи:
 `SELECT value (<expr>) FROM Tasks as t WHERE t.TaskID = @ContextID
` XML-узлы и соответствующие провайдеры:
 XML-узел Результат в SQL `<eventparameter index="N">` `@EventParamN` `<cyclevariable name="X">` `@cycle_value` / `@cycle_idx` `<variable name="X">` `@X_` (результат действия пакета)

SmartExpression T-SQL: расчёт срока по контекстной задаче
 Срок рассчитывается по полям текущей задачи (`@ContextID`):
 `SELECT
   CASE
       WHEN Extparam84NativeValue IS NOT NULL
            AND Extparam71NativeValue > GETDATE()
       THEN dbo.tc_AddWorkingDays(Extparam71NativeValue, 1)
       ELSE dbo.tc_AddWorkingDays(GETDATE(), 1)
   END
FROM TasksInSubcat21Denormalized
WHERE Taskid = @ContextID
` SmartExpression T-SQL: проверка акцептантов подписи
 Для акцептантов контекст задачи передаётся через `@eventParam0` (не `@ContextId`):
 `if exists (
    select u.UserID
    from TasksInSubcat22Denormalized t22 (nolock)
    join ExtParamSelectUsersValues epvu (nolock)
        on t22.TaskID = epvu.TaskID and epvu.ExtParamID = 76
    join Users u (nolock)
        on u.UserID = epvu.UserID
    where t22.TaskID = @ContextId      -- ← обычный контекст
        and u.IsFired_2 = 0
    )
  select 1
else
  select 0
` Для случая подписи: параметр задачи — `@eventParam0`, не `@ContextId`.
 SmartExpression T-SQL: циклический пакет
 В циклическом пакете доступна переменная `@cycle_value`:
 `select ExtParam1567NativeValue
from TasksInSubcat1725Denormalized
where taskid = try_cast(@cycle_value as int)
` `@cycle_value` — текущий элемент итератора (string). Используется `try_cast`, так как тип итератора — string.
 ESQL ветка (XML Content): сравнение параметра события с полем задачи
 Условие из визуального конструктора преобразуется в ESQL:
 `<?xml version="1.0" encoding="utf-16"?>
<smartfilter>
  <property name="TaskID" />
  <operator key="eq" />
  <eventparameter index="1" />
</smartfilter>
` Результат ESQL: `t.[TaskID] = @EventParam1`, обёрнутый в:
 `SELECT value (t.[TaskID] = @EventParam1) FROM Tasks as t WHERE t.TaskID = @ContextID
` ДП Lookup и SelectUsers/SelectGroups/SelectOrgUnits: смарт-фильтр с контекстными параметрами
 При открытии карточки задачи SmartFilter ДП Lookup и ДП «Выбор пользователей» (SelectUsers/SelectGroups/SelectOrgUnits) получает:

- `@eventParam1` — ID текущей категории (SubcatId)

- `@eventParam2` — ID задачи-источника (TaskId)

- `@eventParam3` — JSON значений строки таблицы (если ДП находится в колонке ДП Таблица)

- `@eventParam4` — ID блока
  Событие «Открытие задачи»: нетипичное использование @eventParam
 Событие `WhenOpenTask` передаёт контекст открытия в `@eventParam1` (не `@eventParam0`). `@eventParam0` у данного события не используется — нетипичное поведение.

Что нужно учитывать при использовании переменных:

-  `@ContextID` в фильтре расписания. Фильтр расписания — T-SQL, возвращающий список задач (не boolean). `@ContextID` в нём не имеет смысла: нет «текущей задачи». Если `WHERE TaskID = @ContextID` остался от конвертации смарт-выражения в TSQL — убрать.


-  `@EventParam{N}` доступен только при наличии eventId. В выражении без привязки к событию параметры `@EventParamN` не подставляются.


-  `@cycle_value` и `@cycle_idx` — только в циклическом пакете. В обычном пакете эти переменные недоступны.


-  `@cycle_value` — строка, не int. Тип `string`; при использовании как ID задачи применять `try_cast(@cycle_value as int)`.


-  Пустой элемент итератора не роняет цикл. Если итератор возвращает пустой элемент, `@cycle_value` приходит пустым и итерация отрабатывает штатно.


-  `@CurrentSessionUserID` — текстовая замена в ShowTasksFeed. В смарт-выражении это SQL-параметр; в процедуре `ShowTasksFeed` он заменяется на `@UserID` до выполнения запроса.


-  `generateDummyParams: true` подставляет `@ContextId = -1`. При тестировании выражения через Admin API с этим флагом JOIN на контекстную задачу вернёт пустой результат, но синтаксическая проверка пройдёт.


-  Регистр переменных. В T-SQL имена распознаются без учёта регистра (`@ContextID`, `@ContextId`, `@contextid` — одно и то же). На практике — можно писать как угодно, платформа распознает.

  Связанные документы
 Смежные разделы:

- Паттерны smart-автоматизации — практические примеры выражений и T-SQL

- Смарт-действия — администрирование — настройка событий и пакетов

- Смарт-фильтры — администрирование — Admin API, тестирование выражений

- ДП «Ссылка» — справочник настроек — контекстные `@eventParam` в ДП Lookup
