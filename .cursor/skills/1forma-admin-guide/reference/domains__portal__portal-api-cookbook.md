# Порталы — API Cookbook: создание портала и добавление виджетов

Источник: https://help.1forma.ru/domains/portal/portal-api-cookbook/

Полная инструкция по программному управлению порталами через API администрирования. Все примеры — рабочие, проверены на продакшне.
 ⚠️ Если виджету нужен SmartScript (источник данных через публикацию, custom-action, обработчик) — создавайте его штатными средствами: при ручном создании учитывайте, что `contextType` должен быть строкой `"None"` (а не null), а тело запроса оборачивается в `{script, context}`.
 Аутентификация: все маршруты администрирования требуют прав администратора (God). Авторизация через PAT: `-H "1F-Pat: {token}"
`

Маршрут: `POST /app/v1.0/api/portal/addAdminTemplate`
 `curl -s -X POST "$HOST/app/v1.0/api/portal/addAdminTemplate" \
  -H "1F-Pat: $PAT" -H "Content-Type: application/json" \
  -d '{
    "Name": "Dev Burn-up Dashboard",
    "isCustomerZone": false,
    "ShowHeader": true,
    "ModuleId": null
  }'
` Ответ: объект `PortalGridTemplateDto` с присвоенным `id`: `{
  "id": 5071,
  "name": "Dev Burn-up Dashboard",
  "gridType": "Flex",
  "isFixedMode": true,
  "showHeader": true
}
`
 Портал создаётся с GridType=Flex. Для CssGrid (современная раскладка) — обновить через `updateAdminTemplate` (шаг 4).
 Поле Тип Описание `Name` string Название портала `isCustomerZone` bool? Клиентская зона (обычно false) `ShowHeader` bool Показывать заголовок портала `ModuleId` int? Привязка к модулю (null = без привязки)

Маршрут: `POST /api/portals/block/add`
 `curl -s -X POST "$HOST/api/portals/block/add" \
  -H "1F-Pat: $PAT" -H "Content-Type: application/json" \
  -d '{
    "Name": "Burn-up спринта",
    "TypeId": 33,
    "IsRequired": false,
    "IsEnabled": true,
    "ShowFooter": false,
    "TemplateId": 5071,
    "ContextTypeId": 0,
    "IsFilterHidden": true,
    "IsHideableByUser": false,
    "IsHeaderHidden": false,
    "CanExpand": false,
    "WidgetNameType": 0,
    "WidgetColorMode": 0,
    "GroupsIds": [1620],
    "SectionsIds": []
  }'
` Ответ: `13461` (int — ID созданного блока).

Основные поля запроса:
 Поле Тип Описание `Name` string Название виджета `TypeId` int Тип блока (см. таблицу ниже) `TemplateId` int ID портала (не portalId!) — поле названо `templateId` в DTO `GroupsIds` int[] Группы доступа — обязательно, иначе виджет не виден `SectionsIds` int[] Секции портала (для группировки) `ContextTypeId` byte 0 = Портал `IsEnabled` bool Включён ли виджет `IsHideableByUser` bool Может ли пользователь скрыть `IsHeaderHidden` bool Скрыть заголовок виджета `CanExpand` bool Разворачивается на весь экран `WidgetNameType` enum 0=Fixed, 1=Smart `WidgetColorMode` enum 0=Default, 1=Transparent

Частые проблемы и их решения:
 Проблема Причина Решение Над виджетом показывается старое/неправильное название `Name` блока отображается пользователям как заголовок виджета. Это НЕ внутреннее имя — оно видно в интерфейсе Обновить `Name` через `POST /api/portals/block/{blockId}/update` с `{"Name": "Новое имя"}` Хочу убрать заголовок виджета совсем По умолчанию `IsHeaderHidden: false` Установить `IsHeaderHidden: true` при создании или обновлении блока Два заголовка: портальный ShowHeader + виджетный Name `ShowHeader` портала и `Name` блока — независимые. Оба могут отображаться одновременно Выбрать один: либо `ShowHeader: false` на портале, либо `IsHeaderHidden: true` на блоке Дублирование: Name блока + заголовок в SmartHtml template Если SmartHtml содержит свой `<h1>` и блок показывает Name — будет два заголовка Убрать заголовок из SmartHtml template — использовать Name блока как единственный заголовок

Идентификаторы типов блоков (`TypeId`):
 TypeId Тип Описание 0 Buttons Блок кнопок 1 HTML Блок HTML 2 Subcat Выборка задач (категория) 5 SmartBlock Смарт-блок 18 Chart График 19 TasksSearch Поиск задач 20 Feed Лента событий 21 Menu Навигационное меню 22 Table Таблица 29 Pivot Сводная таблица 30 Gantt Диаграмма Ганта 33 SmartHtml Smart HTML (самый гибкий) 34 CalendarSmartHtml Календарь Smart HTML 38 Calendar Календарь 40 Counters Счётчики 44 Filter Виджет фильтров 45 Navigation Навигация 50 Poll Мини-опрос

Частые проблемы и их решения:
 Проблема Причина Решение Блок создан, но не виден Нет `GroupsIds` Передать `GroupsIds: [1620]` при создании TemplateId при создании ИГНОРИРУЕТСЯ Известный баг: при создании блок попадает на случайный портал После создания вызвать `POST /api/portals/block/{blockId}/update` с правильным `TemplateId` Блок на портале, но не рендерится Блок не добавлен в mesh См. шаг 4 — добавление в mesh. TemplateId — для группировки в админке, mesh определяет рендеринг

После создания блока SmartHtml — настроить HTML-шаблон и (опционально) смарт-выражения.
 Маршрут: `POST /api/admin/portals/blocks/advsets/smart-html?blockId={blockId}`
 `curl -s -X POST "$HOST/api/admin/portals/blocks/advsets/smart-html?blockId=13461" \
  -H "1F-Pat: $PAT" -H "Content-Type: application/json" \
  -d '{
    "template": "<div id=\"burnup-chart\" style=\"width:100%;min-height:400px;\"></div>",
    "smarts": ""
  }'
` Ответ: HTTP 204 No Content.
 Поле Тип Описание `template` string HTML-шаблон (mustache-совместимый) `smarts` string JSON-конфиг смарт-выражений (пустая строка если не нужны) Важно: `blockId` — query parameter, не в body.

Один из типов источника данных для наполнения SmartHtml-виджета (`type: smartFilter`, конфигурация с полем `smartFilterId`) — вывод задач по сохранённому смарт-отбору.
 Такой источник автоматически пересекает результат отбора с правами текущего пользователя (row-level ACL, аналогично основному гриду задач) — пользователь видит только те задачи, к которым у него есть доступ, даже если отбор технически возвращает более широкий набор. Если у пользователя есть право «просмотр всех задач категории», отбор отдаётся целиком.
 Без определённого пользователя сессии источник возвращает пустой результат, а не полный отбор (fail-closed).
 Для агрегатных табло, которые намеренно показывают весь отбор категории под групповым доступом к виджету (не завязываясь на права конкретного смотрящего), в конфигурации источника предусмотрен параметр `SkipRowPermissions` (по умолчанию выключен) — отключает наложение построчных прав.

SPA загружает блоки из `mesh` JSON — поля `PortalGridTemplates.Mesh`. Просто привязка блока к порталу через `TemplateId` недостаточна — блок должен быть явно перечислен в mesh.

Запрос текущей конфигурации портала:
 `curl -s -X POST "$HOST/app/v1.0/api/portal/adminTemplates" \
  -H "1F-Pat: $PAT" -H "Content-Type: application/json" \
  -d '{
    "targetPortalId": 5071,
    "WithTemplateContent": true,
    "WithTemplateBlockIncludes": true
  }'
` Из ответа извлечь: `result[0].template.{id, name, mesh, dashboard, gridType, isFixedMode, showHeader, moduleId}`.

Формат mesh — JSON с массивом `blocks[]` и режимом размещения:
 `{
  "blocks": [
    {
      "guid": "уникальный-uuid",
      "blockIds": [13461],
      "type": "widget",
      "containerRules": {},
      "sectionRules": {"type": "1/1"},
      "sizeRules": {
        "13461": {
          "13461": {"type": "custom", "col": 2, "row": 2},
          "type": "custom",
          "col": 12,
          "row": 4
        }
      }
    }
  ],
  "portalPlacingMode": "row"
}
` Структура sizeRules — двухуровневая: `"sizeRules": {
  "13461": {                    ← ключ = blockId (строка)
    "13461": {                  ← вложенный ключ = blockId (мин. размеры / fallback)
      "type": "custom",
      "col": 2, "row": 2
    },
    "type": "custom",           ← основной тип размера
    "col": 12,                  ← ширина в колонках (из 12)
    "row": 4                    ← высота в рядах
  }
}
` `col: 12, row: 4` = полная ширина, 4 ряда. Внешний уровень — размер блока на портале, внутренний — минимальные/свёрнутые размеры.
 Дополнительные свойства блока в mesh:
 Поле Тип Описание `guid` string Уникальный UUID записи в mesh `blockIds` int[] ID блоков (обычно один) `type` string `"widget"` или `"section"` `startNewRow` bool Начать с новой строки `sizeRules` object Размеры: col (1-12), row

Маршрут: `POST /app/v1.0/api/portal/updateAdminTemplate`
 `curl -s -X POST "$HOST/app/v1.0/api/portal/updateAdminTemplate" \
  -H "1F-Pat: $PAT" -H "Content-Type: application/json" \
  -d '{
    "id": 5071,
    "name": "Dev Burn-up Dashboard",
    "isFixedMode": false,
    "mesh": "{\"blocks\":[...],\"portalPlacingMode\":\"row\"}",
    "dashboard": null,
    "GridType": 2,
    "ShowHeader": true,
    "ModuleId": null
  }'
` Ответ: `true` (HTTP 200).

Все поля обязательны — маршрут перезаписывает ВСЕ поля сущности:
 Поле Тип Описание `id` int ID портала `name` string Название (перезапишет!) `isFixedMode` bool? Фиксированный режим `mesh` string JSON mesh layout (строка!) `dashboard` string JSON dashboard layout (null для CssGrid) `GridType` int 0=Flex, 1=Dashboard, 2=CssGrid `ShowHeader` bool Показывать заголовок `ModuleId` int? Модуль

Значения типа раскладки портала:
 Значение Тип Описание 0 Flex Старый тип, через templateJson 1 Dashboard Dashboard layout 2 CssGrid Современный, через mesh JSON

Частые проблемы и их решения:
 Проблема Причина Решение HTTP 500 Не все поля переданы (name=null) Передать ВСЕ поля, включая name, GridType и т.д. Портал стал пустым mesh не содержит blocks Проверить JSON mesh, убедиться что blockIds заполнены Название портала изменилось name перезаписывается Передавать текущее имя



Маршрут: `POST /api/includes/add`
 `curl -s -X POST "$HOST/api/includes/add" \
  -H "1F-Pat: $PAT" -H "Content-Type: application/json" \
  -d '{
    "name": "Burn-up Sprint Chart",
    "type": 1,
    "content": "(function() { /* JS code */ })()"
  }'
` Ответ: `{data: 5840}` (ID include).
 Поле Тип Описание `name` string Описание (в DTO — `description`) `type` int 1=Js, 2=Css, 3=JsUrl `content` string Содержимое (JS/CSS код или URL для JsUrl)

Частые проблемы и их решения:
 Проблема Причина Решение Перезаписан чужой Include `POST /api/includes/add` возвращает ID в ответе, но если ID не извлечён — разработчик ищет «последний ID» через SQL и попадает на чужой Include. `POST /api/includes/{id}/update` перезаписывает content без проверки владельца Всегда извлекать ID из ответа `POST /api/includes/add` (`{data: 5840}`). Никогда не угадывать ID по SQL. Перед `update` — проверить `description` через `SELECT description FROM Includes WHERE IncludeId = N` Include обновлён, но код не изменился Баг: `POST /api/includes/{id}/update` иногда не обновляет `Content` Создать новый include через `POST /api/includes/add`, перепривязать к блоку, удалить старый Тип сбросился на CSS (code не выполняется) При `update` не передан `Type: 1` — сбрасывается на 0 (CSS) Всегда передавать `Type: 1` (JS) при обновлении Include привязан к нескольким блокам Один Include может быть на N блоках. Обновление затрагивает ВСЕ Перед update проверить: `SELECT PortalId FROM PortalIncludes WHERE IncludeId = N` — убедиться что Include используется только вашим блоком

Маршрут: `POST /api/portals/block/{blockId}/includes/add?includeId={includeId}`
 `curl -s -X POST "$HOST/api/portals/block/13461/includes/add?includeId=5840" \
  -H "1F-Pat: $PAT"
` Ответ: HTTP 200.
 Важно: `includeId` — query parameter, не body. Запрос требует пустой body `{}` (без body — HTTP 400).

Маршрут: `POST /api/includes/{includeId}/update`
 `curl -s -X POST "$HOST/api/includes/5840/update" \
  -H "1F-Pat: $PAT" -H "Content-Type: application/json" \
  -d '{
    "Description": "Burn-up Sprint Chart",
    "Type": 1,
    "Content": "(function() { /* updated JS */ })()"
  }'
` КРИТИЧНО: всегда передавать `Type` при обновлении. Маршрут перезаписывает все поля. Если не передать `Type: 1` (JS), поле сбрасывается на `0` (CSS) — include перестаёт выполняться как JavaScript. SmartHtml рендерит CSS-includes через `<style>`, а JS — через `<script>`. При сбросе типа код попадёт в `<style>` и будет проигнорирован браузером.
 Минимальный безопасный вызов: `curl -s -X POST "$HOST/api/includes/5840/update" \
-H "1F-Pat: $PAT" -H "Content-Type: application/json" \
-d '{"Content": "..", "Type": 1}'
`
 Баг (март 2026): `Content` игнорируется при update. `POST /api/includes/{id}/update` возвращает 200, но `Content` не перезаписывается — `GET /api/includes/include/{id}` отдаёт старое содержимое. Обновляется только `Description`. Workaround: создать новый include через `POST /api/includes/add`, привязать к блоку, отвязать старый (`DELETE /api/portals/block/{blockId}/includes/delete?includeId={includeId}` — includeId в query, НЕ в path).

реализовано — используйте `spaApi.loadApexCharts` вместо CDN-загрузки. CDN-загрузка (`loadApexCharts`/`ensureApexCharts`) устарела. Существующие виджеты с guard `if (window.ApexCharts) return cb` продолжат работать.
 Канонический паттерн для ApexCharts в JS Include:
 `spaApi.loadApexCharts().then(function(ApexCharts) {
    var chart = new ApexCharts(el, opts);
    chart.render();
});
` JS-код виджетов обязан соответствовать дизайн-системе 1Формы.
 Запрещено:

- Хардкод цветов (`#005eff`, `rgba(0,0,0,0.6)`) — только CSS-токены

- Инлайн `font-size`, `font-weight` — только классы `vh-*`

- Неполный `fontFamily` — только полный системный стек
  Как читать токены в JS (для ApexCharts, Chart.js и т.д.): `function token(name) {
    return getComputedStyle(document.documentElement).getPropertyValue(name).trim();
}

// Графики: серии данных
var primary   = token('--surfacedata-primary');    // синий
var secondary = token('--surfacedata-secondary');  // мятный

// Текст на осях, легенда
var axisText = token('--onsurface-secondary');

// Сетка графика
var gridLine = token('--outline-shallow');

// Бейджи: container + oncontainer пара
var bgInfo    = token('--container-info');
var textInfo  = token('--oncontainer-info');
`
 Типографика в HTML, генерируемом из JS: `// ПРАВИЛЬНО — класс vh-label-md задаёт 12px/16px, weight 500
'<span class="vh-label-md" style="background:' + token('--container-success') + '">'

// НЕПРАВИЛЬНО — инлайн font-size/weight
'<span style="font-size:12px; font-weight:500; background:#e0ffea">'
`
 Полный системный шрифт (для chart-библиотек): `-apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Ubuntu, Oxygen, 'Fira Sans', 'Droid Sans', 'Helvetica Neue', sans-serif
`
 Ключевые токены для виджетов:
 Назначение Токен Серии графика `--surfacedata-primary`, `--surfacedata-secondary`, `--surfacedata-dataset-*` Основной текст `--onsurface-primary` Вторичный текст (оси, мета) `--onsurface-secondary` Сетка графика `--outline-shallow` Бейдж info (фон/текст) `--container-info` / `--oncontainer-info` Бейдж success `--container-success` / `--oncontainer-success` Бейдж default `--container-default` / `--oncontainer-default` Ошибка `--onsurfaceextra-danger`

Частые проблемы и их решения:
 Проблема Причина Решение Include не выполняется в SmartHtml Include привязан к порталу, а не к блоку Привязать к блоку: `POST /api/portals/block/{blockId}/includes/add?includeId=N`. SmartHtml выполняет только block-level includes (`jsIncludesIds`) При SPA-навигации между вкладками — пустота JS Include (IIFE) выполняется один раз при первом рендере блока. SPA client-side navigation не перезапускает скрипт Использовать MutationObserver (паттерн ниже) Двойная инициализация виджетов MutationObserver срабатывает на уже инициализированные элементы Маркировать атрибутом (`data-hm-init`), искать `:not([data-hm-init])` ApexCharts heatmap дёргается при клике `dataPointSelection` переключает selected state и перерисовывает серии Использовать `chart.events.click` + `animations: {enabled: false}` + `states: {active: {filter: {type: 'none'}}}` Include обновлён, но не работает `POST /api/includes/{id}/update` без `Type: 1` сбрасывает тип на CSS (0). Include инжектится как `<style>` вместо `<script>` Всегда передавать `"Type": 1` при обновлении JS-includes (§5c) CDN-библиотеки (d3, dagre) не грузятся Корпоративный firewall блокирует CDN. Или `loadScript` вызывается до DOM ready Fallback: проксировать через свой сервер. Проверять `typeof d3 !== 'undefined'` перед использованием Portal-level include не выполняется в SmartHtml Portal-level includes загружаются через `PortalIncludes`, но SmartHtml-блок использует только `PortalBlockIncludes` (block-level) Привязать include к блоку, не к порталу. Portal-level includes работают для других типов виджетов Шрифты/лейблы в d3-виджете мелкие, особенно на большом мониторе `d3.zoom().scaleExtent([fitScale, ...])` с `fitScale = viewportH / totalHeight` < 1. Когда контент не помещается по высоте — весь SVG (включая `<text>`) скейлится вниз через `transform: scale()`. Чем больше lanes/rows — тем меньше fitScale, тем мельче шрифт. На большом мониторе кажется парадоксом, но lanes тоже растут (`N × rowHeight`) и не помещаются → scale всё равно < 1 Не использовать auto-fit по высоте. SVG рендерить в полную `totalHeight`, контейнер — `overflow-y: auto` (вертикальный scroll). d3.zoom оставить только для X (если нужен), `scaleExtent([1, 8])` без сжатия вниз. Шрифты тогда абсолютные, lanes scrollable. Паттерн ниже Паттерн: SPA-safe инициализация виджетов через MutationObserver
 JS Include в SmartHtml — IIFE, который выполняется один раз. Если пользователь переключается между порталами через вкладки (SPA navigation), новые DOM-элементы появляются без повторного вызова скрипта. Решение:
 `(function() {
    function discoverAndInit() {
        // Ищем только НЕинициализированные контейнеры
        var containers = document.querySelectorAll('[data-source]:not([data-hm-init])');
        if (!containers.length) return;

        containers.forEach(function(c) {
            c.setAttribute('data-hm-init', '1'); // маркер против двойной инит
            initWidget(c);
        });
    }

    // Первый проход — текущий DOM
    discoverAndInit();

    // SPA navigation — ловим появление новых элементов
    new MutationObserver(function(mutations) {
        for (var i = 0; i < mutations.length; i++) {
            var added = mutations[i].addedNodes;
            for (var j = 0; j < added.length; j++) {
                var node = added[j];
                if (node.nodeType !== 1) continue;
                if ((node.hasAttribute && node.hasAttribute('data-source')
                        && !node.hasAttribute('data-hm-init'))
                    || (node.querySelector
                        && node.querySelector('[data-source]:not([data-hm-init])'))) {
                    discoverAndInit();
                    return; // один вызов на batch мутаций
                }
            }
        }
    }).observe(document.body, {childList: true, subtree: true});

    function initWidget(container) { /* ... */ }
})();
` Ключевые моменты:

- `data-hm-init` — атрибут-маркер, предотвращает двойную инициализацию

- `MutationObserver` с `{childList: true, subtree: true}` — ловит вставку элементов на любом уровне

- `return` после `discoverAndInit` — один вызов на batch мутаций (оптимизация)

- Паттерн универсален: заменить `[data-source]` на свой селектор контейнера
  Паттерн: d3 swimlane/gantt/timeline — НЕ скейлить по высоте, использовать scroll
 Антипаттерн (мелкий шрифт на большом мониторе): `// ❌ Auto-fit по высоте — весь SVG (включая <text>) сжимается через transform: scale()
var totalHeight = lanes.length * rowHeight;       // напр. 30 × 60 = 1800
var fitScale    = container.clientHeight / totalHeight;  // 800 / 1800 = 0.44
svg.attr('height', container.clientHeight);
g.attr('transform', 'scale(' + fitScale + ')');   // labels тоже × 0.44 → 5px вместо 11px
d3.zoom().scaleExtent([fitScale, 8]).on('zoom', ...);
`
 Фикс (шрифты абсолютные, lanes scrollable): `// ✅ SVG в полную высоту контента, контейнер скроллится вертикально
var totalHeight = lanes.length * rowHeight;
svg.attr('height', totalHeight);                  // полная высота — НЕ обрезаем
container.style.overflowY = 'auto';               // вертикальный scroll
container.style.maxHeight = '70vh';               // ограничение viewport, не SVG
container.style.overflowX = 'hidden';

// d3.zoom — только горизонтальный (если нужен таймлайн с zoom по X)
d3.zoom()
  .scaleExtent([1, 8])                            // НЕ ниже 1 — никакого сжатия вниз
  .translateExtent([[0, 0], [totalWidth, totalHeight]])
  .on('zoom', function(ev) {
      xAxis.attr('transform', 'translate(' + ev.transform.x + ',0) scale(' + ev.transform.k + ',1)');
      // Y не трогаем — lanes имеют фиксированную геометрию, шрифты не плывут
  });
`
 Чек-лист для d3-виджета на портале:

- `font-size` у `<text>` задавать через CSS-класс `vh-label-md`/`vh-body-sm` (12px/14px) или абсолютным значением в `style.fontSize` — не полагаться на наследование от scaled `<g>`

- Если контент по высоте больше viewport — это нормально, делай scroll контейнера, не сжимай SVG

- d3.zoom с `scaleExtent` начинающимся ниже 1 (`[0.4, 8]`, `[fitScale, 8]`) — почти всегда баг: пользователь не должен видеть контент сжатым меньше 100%

- Проверить на двух мониторах разных размеров — если на большом всё мельче (а должно быть наоборот), значит auto-fit ещё где-то сидит

Получение и управление привязками includes:
 `# Получить привязанные
GET /api/portals/block/{blockId}/includes

# Доступные для добавления
GET /api/portals/block/{blockId}/includes/available-for-add

# Удалить привязку
DELETE /api/portals/block/{blockId}/includes/delete?includeId={includeId}
`

Includes можно привязать к порталу целиком (работают на всех блоках портала):
 `# Привязать к порталу
POST /api/admin/portals/{portalId}/includes/add?includeId={includeId}

# Список includes портала
GET /api/admin/portals/{portalId}/includes

# Доступные для добавления
GET /api/admin/portals/{portalId}/includes/available-for-add

# Удалить
DELETE /api/admin/portals/{portalId}/includes/delete?includeId={includeId}
` Разница: portal-level includes загружаются через `PortalIncludes`, block-level — через `PortalBlockIncludes`.
 Для SmartHtml (TypeId=33) — только block-level. SmartHtml-блок при рендеринге читает `jsIncludesIds` из `PortalBlockIncludes`. Portal-level includes (`PortalIncludes`) не попадают в SmartHtml-блоки. Если include привязан к порталу, но не к блоку — SmartHtml его не увидит.

Маршрут: `POST /api/portals/block/{blockId}/update`
 `curl -s -X POST "$HOST/api/portals/block/13461/update" \
  -H "1F-Pat: $PAT" -H "Content-Type: application/json" \
  -d '{
    "Name": "Burn-up спринта",
    "TypeId": 33,
    "IsEnabled": true,
    "IsRequired": false,
    "ShowFooter": false,
    "TemplateId": 5071,
    "ContextTypeId": 0,
    "IsFilterHidden": true,
    "IsHideableByUser": false,
    "IsHeaderHidden": false,
    "CanExpand": false,
    "WidgetNameType": 0,
    "WidgetColorMode": 0,
    "GroupsIds": [1620],
    "SectionsIds": []
  }'
`

Маршрут: `POST /app/v1.0/api/portal/rename`
 `curl -s -X POST "$HOST/app/v1.0/api/portal/rename" \
  -H "1F-Pat: $PAT" -H "Content-Type: application/json" \
  -d '{
    "PortalId": 5071,
    "NewName": "Dev Burn-up Dashboard",
    "ShowHeader": true
  }'
`

Маршрут: `DELETE /api/admin/portals/{portalId}`
 `curl -s -X DELETE "$HOST/api/admin/portals/5071" \
  -H "1F-Pat: $PAT"
`

Маршрут: `DELETE /api/portals/block/{blockId}/delete`
 `curl -s -X DELETE "$HOST/api/portals/block/13461/delete" \
  -H "1F-Pat: $PAT"
` Удаление блока не проверяет mesh — блок удалится, но останется «призраком» в mesh JSON. SPA проигнорирует несуществующий blockId, но mesh лучше почистить вручную.

Доступ настраивается на уровне виджета (блока), не портала. Без `GroupsIds` виджет не виден обычным пользователям.
 При создании: передать `GroupsIds: [1620]` в `PortalBlockCreateRequestDto`.
 При обновлении: передать `GroupsIds: [1620]` в `PortalBlockUpdateRequestDto` через `POST /api/portals/block/{blockId}/update`.
 Проверка: пользователь видит виджет, если состоит в одной из групп блока (напрямую или через ассистирование).

При создании публикации автоматически создаётся ExternalObject (GUID возвращается в ответе). Без настройки доступа — 403 Forbidden при вызове.

Запрос текущих настроек доступа:
 `GET /app/v1.2/api/externalObjects/{guid}/full
` Ответ: `{
  "id": 1164,
  "guid": "acc8ef91-...",
  "description": "Публикации: gitlab-dashboard",
  "visibleToAnyone": false,
  "isAnonymous": false,
  "externalObjectsViewRights": [],
  "externalObjectsSpecialRights": [],
  "externalObjectsSmartExpressionRight": null
}
`

Открыть публикацию всем авторизованным пользователям:
 `POST /app/v1.2/api/externalObjects/{guid}
{"visibleToAnyone": true}
`

Ограничить доступ выбранными группами:
 `curl -s -X POST "$HOST/app/v1.2/api/externalObjects/{guid}" \
  -H "1F-Pat: $PAT" -H "Content-Type: application/json" \
  -d '{
    "visibleToAnyone": false,
    "isAnonymous": false,
    "description": "Публикации: gitlab-dashboard",
    "externalObjectsViewRights": [
      {"externalObjectId": "{guid}", "groupId": 1620}
    ]
  }'
` Ответ: HTTP 204 No Content.
 GUID — из ответа на создание публикации (`POST /app/v1.2/api/publishedObjects` → `data.externalObjectId`). Если GUID утерян — найти через `GET /app/v1.2/api/publishedObjects/gridData` (поле `externalObjectId`).

Частые проблемы и их решения:
 Проблема Причина Решение 403 при вызове публикации ExternalObject access не настроен `visibleToAnyone: true` или добавить группы 500 при обновлении Неправильный формат body. `groupIds` не существует Использовать `externalObjectsViewRights: [{externalObjectId, groupId}]` GET возвращает 404 Путь без `/full` Использовать `GET ../externalObjects/{guid}/full` Пользователь видит виджет, но данных нет Блок доступен (группы блока), но публикация закрыта Настроить ExternalObject с теми же группами `visibleToAnyone` = true по умолчанию (DDL) Новые публикации видны всем Для привилегированных операций — явно снять флаг, назначить группу

Ожидаемые упрощения: - шаг 3 исчезает (TemplateId будет работать при создании) - P2 (add-to-mesh) → шаг 7 сокращается до одного вызова - P4 (quick publication) → цепочка Script→Pack→Action→Publication сокращается до одного вызова
 `1. Создать портал
   POST /app/v1.0/api/portal/addAdminTemplate
   → portalId

2. Создать блок SmartHtml
   POST /api/portals/block/add
   {TemplateId: portalId, TypeId: 33, GroupsIds: [1620], ...}
   → blockId
   ⚠️ TemplateId игнорируется при создании (баг, ждём фикс)

3. Привязать блок к правильному порталу (уйдёт после исправления)
   POST /api/portals/block/{blockId}/update
   {TemplateId: portalId, ...все остальные поля...}

4. Настроить HTML шаблон
   POST /api/admin/portals/blocks/advsets/smart-html?blockId={blockId}
   {template: "<div>...</div>", smarts: ""}

5. Создать JS Include
   POST /api/includes/add
   {name: "...", type: 1, content: "..."}
   → includeId

6. Привязать Include к блоку
   POST /api/portals/block/{blockId}/includes/add?includeId={includeId}

7. Добавить блок в mesh портала (упростится после P2: add-to-mesh)
   a. GET текущий: POST /app/v1.0/api/portal/adminTemplates
   b. Добавить blockId в mesh.blocks[]
   c. Сохранить: POST /app/v1.0/api/portal/updateAdminTemplate
      {id: portalId, GridType: 2, mesh: JSON.stringify(mesh), ...}
` Результат: `https://{host}/spa/portal/{portalId}`

Один блок можно использовать на нескольких порталах — достаточно добавить его `blockId` в mesh нужного портала. SmartHtml-настройки, includes, группы доступа — всё сохраняется на уровне блока и работает везде, где он подключён.
 `1. Загрузить mesh целевого портала
   POST /app/v1.0/api/portal/adminTemplates

2. Добавить существующий blockId в mesh.blocks[]

3. Сохранить
   POST /app/v1.0/api/portal/updateAdminTemplate
`

Сводка основных маршрутов API порталов:
 Операция Метод Маршрут Создать портал POST `/app/v1.0/api/portal/addAdminTemplate` Обновить портал (mesh, GridType) POST `/app/v1.0/api/portal/updateAdminTemplate` Переименовать портал POST `/app/v1.0/api/portal/rename` Удалить портал DELETE `/api/admin/portals/{portalId}` Список порталов GET `/api/admin/portals` Загрузить портал с блоками POST `/app/v1.0/api/portal/adminTemplates` Создать блок POST `/api/portals/block/add` Получить блок GET `/api/portals/block/{blockId}` Обновить блок POST `/api/portals/block/{blockId}/update` Удалить блок DELETE `/api/portals/block/{blockId}/delete` SmartHtml: получить GET `/api/admin/portals/blocks/advsets/smart-html?blockId=N` SmartHtml: сохранить POST `/api/admin/portals/blocks/advsets/smart-html?blockId=N` Include: создать POST `/api/includes/add` Include: обновить POST `/api/includes/{id}/update` Include → блок: привязать POST `/api/portals/block/{blockId}/includes/add?includeId=N` Include → блок: список GET `/api/portals/block/{blockId}/includes` Include → блок: удалить DELETE `/api/portals/block/{blockId}/includes/delete?includeId=N` Include → портал: привязать POST `/api/admin/portals/{portalId}/includes/add?includeId=N` Include → портал: список GET `/api/admin/portals/{portalId}/includes` Include → портал: удалить DELETE `/api/admin/portals/{portalId}/includes/delete?includeId=N` Секции блоков GET `/api/portals/block/sections` Типы блоков GET `/api/portals/block/types`

Navigation widget — таб-бар для переключения между связанными порталами.

Запрос текущих настроек:
 `curl -s "$HOST/api/portals/block/{blockId}/type-params" \
  -H "1F-Pat: $PAT"
` Ответ содержит `data.typeParams[]` с единственным параметром `NavigationWidgetsBlockSettings`, поле `value` — JSON-строка с конфигурацией табов.

Маршрут: `POST /api/portals/block/{blockId}/type-params/update`
 `curl -s -X POST "$HOST/api/portals/block/{blockId}/type-params/update" \
  -H "1F-Pat: $PAT" -H "Content-Type: application/json" \
  -d '{
    "typeParams": [{
      "typedType": "String",
      "type": "String",
      "name": "NavigationWidgetsBlockSettings",
      "title": "Navigation Widget Block Settings",
      "description": null,
      "required": true,
      "editable": true,
      "visible": false,
      "userParam": false,
      "userCanChange": false,
      "availableOptions": null,
      "value": "{JSON навигации — см. ниже}"
    }]
  }'
`

Структура JSON для навигации:
 `{
  "links": [
    {"title": "Таб 1", "portalId": "5371", "order": "1", "children": []},
    {"title": "Таб 2", "portalId": "5381", "order": "2", "children": []}
  ],
  "navType": "tabs",
  "stickyHeader": true
}
` Поле Тип Описание `links[].title` string Текст таба `links[].portalId` string ID целевого портала (строка!) `links[].order` string Порядок (строка) `links[].children` array Вложенные элементы (для dropdown) `navType` string `"tabs"` — горизонтальные табы `stickyHeader` bool Прилипание к верху при скролле

Частые проблемы и их решения:
 Проблема Причина Решение `portalId` должен быть строкой DTO ожидает string, не int `"portalId": "5371"`, не `"portalId": 5371` `POST /api/portals/block/{id}/update` с TypeParams → 500 Этот endpoint не обрабатывает TypeParams Использовать `POST /api/portals/block/{id}/type-params/update` value — строка, не объект value = JSON-encoded string внутри JSON Сериализовать навигацию в строку: `json.dumps(nav_config)`

Пример создания блока через API:
 `block_id = POST("/api/portals/block/add", {
    "Name": "Виджет",
    "TypeId": 33,              # SmartHtml
    "TemplateId": portal_id,   # Внимание: игнорируется при создании (баг)
    "GroupsIds": [1620],        # Обязательно, иначе блок не виден
    "ContextTypeId": 0,
    "IsEnabled": True
})
`

Не все типы блоков имеют выделенный advsets-контроллер:
 TypeId Тип Advsets-маршрут 33 SmartHtml `POST /api/admin/portals/blocks/advsets/smart-html?blockId=N` 21 Menu (Navigation) Настройки в `TypeParams` (JSON-строка внутри JSON-строки) 2 Subcat `POST /api/admin/portals/blocks/advsets/subcat?blockId=N` 18 Chart (График) Управление сериями — см. ниже

Пример настройки HTML-шаблона блока:
 `POST /api/admin/portals/blocks/advsets/smart-html?blockId={blockId}
{"template": "<div>...</div>", "smarts": ""}
# blockId — query parameter, не body!
# Ответ: 204 No Content
` SmartHtml — двойное хранение шаблона: template хранится в `TypeParams` (legacy) и `BlockTemplates` (через advsets API). При обновлении через advsets обновляется `BlockTemplates`.

В advsets добавлено поле `dataSources` — JSON-строка с массивом источников данных. Поддерживаются три типа:
 `POST /api/admin/portals/blocks/advsets/smart-html?blockId={blockId}
{
  "template": "...",
  "smarts": "",
  "dataSources": "[{\"alias\":\"myData\",\"type\":\"custom\",\"sourceKey\":\"source_key\"},{\"alias\":\"tasks\",\"type\":\"smartFilter\",\"smartFilterId\":123,\"fields\":[\"taskText\",\"stateDescription\",\"ep:42\"]},{\"alias\":\"kpi\",\"type\":\"script\",\"scriptId\":456,\"cacheTtlSec\":300,\"timeoutSec\":30}]"
}
` Имена полей конфига: `smartFilterId` / `cacheTtlSec` / `timeoutSec` / `maxRows` — не `filterId`/`ttl`/`timeout`. ДП в `fields` — `ep:{ExtParamID}` (двоеточие); `taskId` всегда в результате и в `fields` не указывается.
 Список системных полей для типа `smartFilter`: `GET /api/admin/portals/blocks/advsets/smart-html/smart-filter-fields
`
 Данные всех источников записываются в `window.__smartHtmlData[blockId]` (по алиасу) до выполнения JS Include. Если `dataSources` пуст или отсутствует — поведение прежнее (обратная совместимость).

Виджет «График» (TypeId 18) имеет выделенный набор маршрутов `/api/admin/portals/blocks/advsets/chart`:
 Метод Маршрут Назначение `GET` `/settings/common` Общие настройки графиков `GET` `?blockId=N` Данные графика блока `POST` `/add-new-chart?blockId=N` Добавить новую серию `DELETE` `/{chartIndex}?blockId=N` Удалить серию по индексу `POST` `?blockId=N` Сохранить настройки графика (тело — `ChartTypeParams`) Метод `DELETE` удаляет серию по индексу `chartIndex` (нумерация с 0), отдельного идентификатора у серии нет.
 В админке виджета каждая серия отображается отдельным блоком («График 1», «График 2» и т. д.). У всех серий, кроме первой, доступна кнопка «Удалить» (с подтверждением). Первая серия удалению не подлежит — удалить можно только добавленные вручную серии.

Создание: `POST("/api/includes/add", {
    "name": "Описание",    # при создании — "name"
    "type": 1,             # 1=Js, 2=Css, 3=JsUrl
    "content": "(function() { ... })()"
})  # → {"data": includeId}
`
 Обновление: `POST(f"/api/includes/{includeId}/update", {
    "description": "Описание",  # при обновлении — "description" (не "name"!)
    "type": 1,
    "content": "(function() { ... })()"
})
`
 Привязка к блоку: `POST /api/portals/block/{blockId}/includes/add?includeId={includeId}
# includeId — QUERY parameter, не body
# Требует пустой body {} — без body → 400
`

Частые ошибки при работе с API порталов и их решения:
 Проблема Причина Решение TemplateId игнорируется при создании Известный баг После создания: `POST /api/portals/block/{id}/update` с правильным TemplateId Блок не виден пользователям Нет `GroupsIds` Всегда передавать `GroupsIds: [1620]` Блок на портале, но не рендерится Блок не в mesh Добавить blockId в mesh JSON (формат в cookbook §4) `updateAdminTemplate` → 500 Не все поля переданы (`name=null`) Передать ВСЕ поля: id, name, mesh, GridType, ShowHeader, ModuleId, dashboard, isFixedMode Портал стал Flex (0) Забыли `GridType: 2` Всегда передавать `GridType: 2` (CssGrid) Удалённый блок в mesh `DELETE block` не чистит mesh Вручную удалить запись из mesh JSON `GET /api/admin/portals/{id}` → 405 Поддерживает только DELETE Читать портал: `GET /app/v1.0/api/portal/template/{id}` `addAdminTemplate` дубль имени → 500 Сервер возвращает 500 (не 409) Проверить существующие порталы `addAdminTemplate` — ответ без `data` Возвращает объект напрямую `response['id']`, не `response['data']['id']` TypeParams — упрощённый формат → NullRef `PUT /api/admin/portals/blocks/{id}` пишет Key/Value Обновлять через `POST /api/portals/block/{id}/type-params/update` `name` vs `description` для includes Разные имена одного и того же поля При создании — `name`. При обновлении — `description` 400 при привязке include Отправили без body Отправить пустой `{}` body Include не выполняется в SmartHtml Привязан к порталу, а не к блоку Привязать к блоку: `block/{blockId}/includes/add` Пустой виджет при SPA-навигации IIFE выполняется один раз MutationObserver + маркер `data-*-init` (паттерн в cookbook §5e) `Type: "JS"` → ошибка Значение `"Js"` PascalCase `"Js"`, `"Css"`, `"JsUrl"` — не UPPERCASE `POST /api/includes` → 404 Нет такого маршрута `POST /api/includes/add` (создание), `/api/includes/{id}/update` (обновление)

Смежные разделы:

- Порталы — администрирование — обзор администрирования порталов
