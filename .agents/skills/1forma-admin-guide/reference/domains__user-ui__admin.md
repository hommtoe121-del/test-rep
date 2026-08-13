# Лента в шапке — диагностика

Источник: https://help.1forma.ru/domains/user-ui/admin/

Диагностический документ для случаев, когда у пользователя пропала «Лента» в интерфейсе 1Формы. Охватывает два разных элемента (панель навигации и блок корп. сети), порядок приоритетов сборки интерфейса, SQL-запросы для проверки рабочих мест групп, глобальные ключи `SettingsCustom`, API-снимок для обращения в поддержку, мини-чеклист тикета и справочник query-параметров SPA. Для инженеров поддержки и администраторов.

В интерфейсе обычно путают два разных элемента:

- `Лента` в основной панели навигации (верхняя/боковая панель).

- `Блок "Лента корп. сети"` в левом меню.
  Это разные настройки и разные поля в БД.
 Логика видимости строится в таком порядке:

- Глобальные настройки приложения (`SettingsCustom`), включая ключи интерфейса.

- Рабочее место группы (`UserGroupSettings`) с учётом `Priority` при нескольких группах.

- Персональные пользовательские настройки (`UserSettings` / `UserUISettings`) — для части параметров интерфейса.

- Дефолты новых пользователей (`UsersNewDefaultSettings`) для параметров, отмеченных как «только для новых».
  Ключевой нюанс: если пользователь состоит в нескольких группах, решает рабочее место с более высоким приоритетом.



Основное поле для проверки: `dbo.UserGroupSettings.LentaVisible`. Сопутствующие поля (элемент может «исчезнуть визуально», если меняется способ показа панели): `NavigationPanelVisible`, `NavigationPanelPosition`, `NavigationPanelColor`.
 `declare @user_id int = 0; -- подставить user id

select
        ug.UserID,
        ug.GroupID,
        g.Name as GroupName,
        ugs.ID as GroupWorkspaceId,
        ugs.Priority,
        ugs.LentaVisible,
        ugs.NavigationPanelVisible,
        ugs.NavigationPanelPosition,
        ugs.NavigationPanelColor
from dbo.UserGroups ug
join dbo.Groups g
        on g.GroupID = ug.GroupID
left join dbo.UserGroupSettings ugs
        on ugs.GroupId = ug.GroupID
where ug.UserID = @user_id and
      isnull(ug.IsDeleted, cast(0 as bit)) = cast(0 as bit)
order by
        case when ugs.Priority is null then -1 else ugs.Priority end desc;
` Что смотреть: - есть ли вообще РМГ у группы; - какое РМГ победит по `Priority`; - какое значение `LentaVisible` у победившего РМГ.

Проверяемые поля: `dbo.UserGroupSettings.CorporateNetworkFeedVisibleType`, `dbo.UsersNewDefaultSettings.IsCorporateNetworkFeedVisible`. Важно: для этого блока действует правило «применяется только для новых пользователей»; для существующих используется отдельный сброс через интерфейс.
 `declare @user_id int = 0; -- подставить user id

select
        ug.UserID,
        ug.GroupID,
        g.Name as GroupName,
        ugs.Priority,
        ugs.CorporateNetworkFeedVisibleType
from dbo.UserGroups ug
join dbo.Groups g
        on g.GroupID = ug.GroupID
left join dbo.UserGroupSettings ugs
        on ugs.GroupId = ug.GroupID
where ug.UserID = @user_id and
      isnull(ug.IsDeleted, cast(0 as bit)) = cast(0 as bit)
order by
        case when ugs.Priority is null then -1 else ugs.Priority end desc;

select top (1)
        unds.IsCorporateNetworkFeedVisible
from dbo.UsersNewDefaultSettings unds;
`

`TopMenuItemsHidingSettings` скрывает: - `Contacts`; - `Create`; - `History`; - `ProfileLinks`; - `Reports`; - `SearchPanel`.
 Пункта `Lenta` в этом ключе нет.
 Проверить актуальные значения:
 `select
        sc.[Key],
        sc.[Value]
from dbo.SettingsCustom sc
where lower(sc.[Key]) in (
        'topmenuitemshidingsettings',
        'socialnetworkssettings',
        'spacesettings'
);
`

Дополнительные ключи CustomSettings, влияющие на интерфейс:
 Ключ Тип / По умолчанию Эффект `HelperCustomLink` string / `""` Если задан непустой URL — иконка Help (правый верхний угол) ведёт на эту ссылку вместо стандартной справки `HideOldGantt` bool Скрывает в проектной задаче кнопку перехода к старому интерфейсу проектного управления (остаётся только переход к новому) `HideUserInfoButton` bool / `false` Скрывает пункт «Инфо пользователя» в контекстном меню при клике на имя в комментариях, в ДП «Выбор пользователя» и «Адресаты Email» `HideUserVoipToken` bool Скрывает `VoipToken` в API-ответах данных о пользователях (для интеграций без необходимости передавать токен) `HideDefaultTags` bool / `false` Скрывает в ленте комментариев теги по умолчанию (название категории и статус задачи) `HideEmptyEpOnNtf` bool / `true` Включает на всю систему настройку «Скрыть на NTF при пустом значении» для ДП. При `false` — управление переходит на уровень категории `highCharts` bool Если `true` — графики и диаграммы рендерятся через HighCharts; иначе — через ApexCharts `BrandSettings` JSON Настройки корпоративного стиля. Генерируется автоматически из общих настроек приложения при сохранении логотипов и параметров (горизонтальный/вертикальный, светлая/тёмная тема, ширина панели). Прямое редактирование не рекомендуется — обновлять через интерфейс `CKEditorCustomConfigPath` string Путь к пользовательскому конфигу CKEditor для ДП «Большой текст с форматированием» (своя панель кнопок, плагины и т. п.) `IsFeedsViewOnly` bool Если `true` — в Избранном по клику на категорию открывается Лента (Таблица скрыта); в дереве «Мои задачи» отображаются только Заказчик/Исполнитель/Подписка, скрываются Согласования и Задачи подчинённых. Используется для упрощённых рабочих мест `EmptyFeedPlaceholderURL` string (URL) Картинка-заглушка пустой ленты (задач, чатов, категории). Если не задана — стандартная иконка `EnableAllTabsNotifications` bool / `null` Управление push-уведомлениями: `true` — во все вкладки браузера, включая фоновые; `false` — только в активную; не указано — показывать всегда (поведение по умолчанию) `ToDoListSettings` JSON Конфигурация ДП Multilookup со схемой «Чек-лист»: соответствие ID категории шаблонов и ID ДП с шаблоном (`{"TaskNotesExtParamId":N,"TemplateTasksExtParamId":N,"TemplatesSubcatId":N}`)

Чтобы быстро локализовать причину, при обращении в поддержку зафиксируйте ответы следующих API-вызовов:

- `GET /api/user/settings`

- `GET /api/user/group-workspace-inserts`

- `POST /api/favorite/menu`

- (для админ-валидации) `GET /api/admin/user-group-settings/{id}/user-ui-settings`
  Мини-чеклист для тикета «не видна Лента в верхней шапке»:

- Уточнить, какая именно лента пропала: основная `Лента` или `Лента корп. сети`.

- Собрать effective РМГ пользователя по `Priority`.

- Проверить `LentaVisible` (для основной ленты) или `CorporateNetworkFeedVisibleType` (для корп. ленты).

- Проверить, не изменено ли положение/видимость панели (`NavigationPanelVisible`, `NavigationPanelPosition`).

- Для корп. ленты подтвердить, применялся ли сброс настроек для текущих пользователей.

- Приложить API-снимок и SQL-выгрузку при обращении в поддержку 1Ф.

При инициализации SPA загружается файл `ui.json`. Затем система генерирует CSS-переменные и устанавливает их в `:root`. Имена переменных соответствуют токенам из `ui.json` и макетов; при переключении темы значения переменных перезаписываются.
 Группа Переменные / правило Назначение Текст `--onsurface-primary`, `--onsurface-secondary`, `--onsurface-tertiary` Основной, вторичный текст и подсказки Контейнеры `--container-primary`, `--oncontainer-primary` Фон контейнера и текст на нём Разделители `--outline-base`, `--outline-shallow`, `--outline-deep` Рамки и разделители Transition `--vh-transition-duration`, `--vh-transition-timing-function`, `--vh-transition-all`, `--vh-transition-bg`, `--vh-transition-colors` Плавность анимаций Тени `--shadow-{color}` Генерируются, если для цвета в `ui.json` задан `shadowMode` Состояния `{color}-hover`, `{color}-click`, `{color}-selected` Генерируются через `color-mix()` из базового цвета и цветов состояний Отступы и радиусы tokens `spacing` и `radius` Значения считаются от базового `space-100` / `radius-100` = `8px` ⚠️ Если у элемента при изменении состояния меняются размеры или отступы, не используйте общий `--vh-transition-all`: задавайте transition на конкретные свойства, иначе анимация может выглядеть некорректно.

В настройках, где используется строковый стиль, поддерживаются типовые значения:
 Значение CSS-эффект `Default` Нет стиля `Success` `background-color:#dff0d8` `Info` `background-color:#d9edf7` `Warning` `background-color:#fcf8e3` `Danger` `background-color:#f2dede` `TotalsRow` `font-weight: bold` `BackRed` `background-color: pink` `BackGreen` `background-color: lightgreen` `BackBlue` `background-color: lightblue` `BackYellow` `background-color: gold` `TextRed` `color: red` `TextGreen` `color: green` Дополнительно можно задавать цвет текста и фон в формате `Color#ХХХХХХ` и `Background#ХХХХХХ`, а высоту строки — в формате `HeightXXXpx`.
 `Color#ff0000,Background#00ff00
Color#ff0000,Height40px
` ⚠️ Если нужно задать несколько CSS-атрибутов, перечисляйте их через пробел. Если нужно задать несколько snippet-значений, перечисляйте их через запятую.

Используется для построения ссылок из писем, виджетов, отчётов, публикаций и сторонних систем в конкретные представления 1Формы.

Базовый формат deep-link:
 `https://{адрес_1Формы}/{модуль}?{parameters}
https://{адрес_1Формы}/{endpoint_with_params}?{parameters}
` Несколько параметров соединяются `&`. Все основные параметры обязательны (исключение — взаимоисключающие `TasksForExtParamID` и `TasksForLookupColumnId` для lookup'а). Дополнительные параметры можно задавать выборочно.
 ⚠️ Если ссылка строится для контекстов, где `?`, `&` и пр. служебные символы интерпретируются интерфейсом (например, `~/spa/link?url=...`), их нужно URL-кодировать:
 Символ Код Символ Код `?` `%3F` `=` `%3D` `&` `%26` пробел `%20` `$` `%24` `[` `%5B` `/` `%2F` `]` `%5D` `%` сам по себе передаётся как `%25` (используется в фильтрах вида `f_task=%25{текст}%25`).
 Переадресация на API-метод: `~/spa/entry/signin?fromUrl={api_url}` — выполняет переадресацию на API после авторизации. Используется, например, в выгрузках Excel: ссылка из ДП «Файл» имеет вид `https://адрес_1Формы/api/files/download/?redirect=1`. К URL добавляется `?auth=true` — при ошибке 401 автоматически открывается страница логина (с v2.257 адрес страницы — в ключе `AuthTokenLoginUrl` в `appsettings.json`).

Ссылки на списки задач, ленты и иерархические представления:
 Назначение URL Ключевые параметры Список задач категории / раздела `~/spa/tasks/subcat/{subcatId}/grid` `~/spa/tasks/category/{categoryId}/grid` — Смарт-отбор в списке `~/spa/tasks/subcat/{subcatId}/grid?SmartQueryId={id}` `SmartQueryId` Отбор по статусам `?ActiveMode=isOpened` / `isClosed` / `isRejected` / пусто (все) Совмещается с `SmartQueryId` Фильтр по колонке (`%25...%25`) `?f_{columnName}=%25текст%25` `task` (текст), `exception` (лог) и т.д. Сводный раздел `~/spa/tasks/summary/{summaryId}/grid` — «Я исполнитель» / «Я заказчик» / «Я подписчик» `~/spa/tasks/{YouPerformerTasks\|FromYou\|Subscribed}/grid?all=false&feedType={Performer\|Owner\|All}` — «На подписи» / «Согласованные» / «Отклонённые» / «Запрошенные» `~/spa/tasks/{ToSign\|Signed\|Declined\|Requested}/grid` или `~/spa/resolutions` — Задачи подчинённых `~/spa/tasks/YourGroups/grid?OrgUnitId={all\|id}` Также: `YourGroupsOverdue`, `YourGroupsActiveSigns`, `YourGroupsSignSigns`, `YourGroupsOverdueSigns`, `YourGroupsRefusedSigns` Главная лента `~/1FMain.aspx` `~/spa/feeds/{extp}` `extp`: `default`, `tasks`, `comments`, `unread`, `lenta`, `questions` Лента категории/раздела `~/spa/tasks/subcat/{id}/feeds` `~/spa/tasks/category/{id}/feeds` — Иерархия задач `~/spa/task-hierarchy/{id}` См. Иерархия задач Таймлайн `~/spa/tasks/{taskId}/timeline` `~/spa/noframe/tasks/{taskId}/timeline` — Гант категории/раздела `~/spa/tasks/subcat/{id}/gantt` `~/spa/tasks/category/{id}/gantt` См. Гант — руководство Календарное представление категории `~/spa/tasks/subcat/{id}/calendar` `~/spa/tasks/category/{id}/calendar` iframe-режим: `~/Syndicate.aspx?forceURL=Scheduler.aspx?SubcatId={id}&HideHeader=true` Канбан категории `~/spa/tasks/subcat/{id}/kanban` См. Канбан — настройка Канбан исполнителя `~/spa/user/kanban/performer/{userId}` До 100 задач каждого типа (на сегодня / на завтра / по сроку) Канбан по публикации `~/spa/kanban/{publicationAlias}` — Чат-вид категории `~/spa/subcat/{id}/chat` — ⚠️ Параметр `f_*` применяется поверх ранее сохранённых фильтров грида и не сбрасывает их. Если результат пустой — очистите фильтры через «Очистить всё». Совместимость со старым `newcustomgrid.aspx?TaskTextFilter=…` — через `f_task=…%25`.

Ссылки на карточку задачи и форму создания:
 Назначение URL Параметры Открыть карточку (SPA) `~/spa/tasks/{TaskID}` `layoutMode`: `mtf` / `mtf-thread` / `mtf-only-thread` Открыть карточку (старый) `~/MainTaskForm.aspx?TaskID={id}` `TemplateID` — пользовательскый шаблон Без шапки и меню `~/spa/noframe/tasks/{TaskID}` + `uiTheme=light\|dark` (только при `layoutMode=mtf-only-thread`) Создание задачи (SPA) `~/spa/newtask/{SubcatId}` `~/spa/noframe/newtask/{SubcatId}` См. ниже Создание задачи (старый) `~/NewTask.aspx?SubcatID={id}` См. ниже Дополнительные параметры создания (`newtask`, `NewTask.aspx`):
 Параметр Описание `OrderedTime` Срок в формате `dd.MM.yyyy%20HH:mm` `TemplateID` Пользовательский шаблон карточки новой задачи `SourceTaskID` Родительская задача `CommentID` Первый комментарий в задаче `UserID` Заказчик (от пользователя) `LinkedID` Связанная задача `TaskText` Текст задачи `ExtParamString` Значения ДП (см. ниже) Формат `ExtParamString`:

- Скаляр: `$Ext{ID_ДП}${значение}` (несколько ДП склеиваются подряд).

- Multilookup: `$Ext{ID_ДП}$[ID_1,ID_2,...]`.

- Таблица (JSON): `$Ext{ID_ДП}${"rows":[{"c{ID_колонки}":"значение",...}]}`. Применяется только к колонкам с режимом ≠ `ReadOnly` и ≠ `Invisible`; smart-колонки игнорируются и считаются автоматически; виртуальные — заполняются; URL имеет приоритет над `DefaultValue` категории; допускается комбинация скалярных ДП и таблицы.

`~/spa/meeting/new/1507` (или `~/spa/noframe/meeting/new/1507`) — `1507` — ID системной категории «Календарь».
 Параметр Формат / значения `Title` Текст `RequiredAttendees` ID пользователей через запятую `Start`, `End` `YYYY-MM-DDTHH:mm:ss.sss` `IsAllDayEvent` `0` / `1` `Location` Текст `FreeBusyState` `Free` / `Tentative` / `Busy` / `OOF` / ~~`WorkingElsewhere`~~ `LinkedTaskId` Номер задачи `Description` Текст `Attachment` ID файлов через запятую Прочее URL Календарь пользователя `~/spa/user/profile/{userId}?tab=calendar` Планировщик встреч `~/spa/scheduling-assistant?users={ID,ID,...}` (или `~/spa/noframe/scheduling-assistant`)

Ссылки на просмотр файлов, Диск и опросы:
 Назначение URL Параметры Просмотр файла Office (`doc/docx/ppt/pptx/xls/xlsx`) `~/spa/file/{fileID}?mode=view` `~/spa/noframe/file/{fileID}?mode=view` `mode`: `view` / `edit`; `taskId`, `extParamId`, `folderId`, `versionId` Просмотр графики/PDF/MP4 (`jpg/png/pdf/mp4`) `~/spa/file/{fileID}/1` `~/spa/noframe/file/{fileID}/1` — Документ в Р7 `~/r7/?fileId={id}&mode=edit&taskId={id}&extParamId={id}` — Главная Диска (полный / краткий) `~/spa/disk` / `~/spa/disk/short` + `noframe`-варианты Диск задачи `~/spa/disk/Task/{TaskID}` — Редактор опроса `~/spa/survey/creator/?surveyTaskId={id}` + `noframe` Прохождение опроса `~/spa/survey?assignmentTaskId={id}` + `noframe` Файловое хранилище (устаревшее) `~/spaex.aspx/file-storage/{FolderType}/{FolderId}` `FolderType`: `Root`, `Folder`, `UserRoot`, `UserFolder`, `Shared`, `TaskRoot`, `Category`, `Subcategory`, `Task`, `LinkedToTask`. Параметры: `view=gallery`, `onlySpaStyles=1`, `showlefttree=1`

Чаты:
 Назначение URL Параметры Создать чат `~/spa/chat/new?with={ID,ID,...}&name={имя}&msg={текст}` Несколько `with=...`; в групповом чате `msg` задаёт название чата, а не сообщение Приватный чат с пользователем `~/spa/chat/new?with={userId}&private=true&msg={текст}` `msg` — текст в поле ввода созданного чата, не публикуется Подписи, почта, пользователи, оргструктура:
 Назначение URL Запрошенные подписи (список) `~/spa/tasks/ToSign/grid` Рабочее место акцептанта `~/spa/resolutions` Почтовая папка `~/Emails/EmailList.aspx?FolderID={id}` Просмотр письма `~/Emails/EmailView.aspx?EmailID={id}` Справочник сотрудников `~/spa/org/list/workers` (publicationAlias) Профиль пользователя `~/spa/user/profile/{userId}` Орг.структура `~/spa/org/chart2/` или `~/spa/org/chart/{publicationAlias}`

Ссылки на порталы, виджеты и страницы пространств:
 Назначение URL Портал Flex `~/spaex.aspx/portal/{portalID}?{filter_params}` Портал Dashboard `~/spa/portal/{portalID}?{filter_params}` или `~/spa/noframe/portal/{portalID}` (без задвоения панели индикаторов изнутри SPA) Виджет (вне портала) `~/spa/portal/block/{BlockID}` или `~/SinglePortalBlock.aspx?BlockID={id}` Виджет в контексте задачи `~/SinglePortalBlock.aspx?BlockID={id}&TaskID={taskID}` Новость в портале `~/spa/portal/{portalId}?widgetId={widgetId}&taskId={taskId}` Настройки виджета `~/administration/widget-settings/{id}` Краткая статья пространства (noframe) `~/spa/noframe/spaces/subcat/{SubcatID}/short?pageId={PageId}` Параметры фильтров портала / виджета (одинаковый формат):
 `f{paramID}_v={value}        # значение
f{paramID}_r=1              # сделать read-only (0 — нет)
` Тип параметра Пример (`paramID=11`) строка `f11_v=test` число `f11_v=2` дата `f11_v=25.06.2018` период `f11_v=from:25.06.2018;to:20.06.2018` пользователь / орг.ст. / выпадающий / группа `f11_v=[1,2,3]` категория `f11_v=cat:[1,2,3];subcat:[11,22,33]` ⚠️ Не поддерживается для виджета Smart Html и для Таблицы с настроенными кнопками. Значения из URL имеют приоритет над значениями по умолчанию фильтра.

Ссылки на просмотр и экспорт отчётов:
 Назначение URL Просмотр отчёта `~/spa/report/new/{ReportID}` или `~/MVC/ReportView/{ReportID}?{params}` Экспорт отчёта `~/MVC/ReportExport/{ReportID}?format={fmt}&{params}` Форматы экспорта: `pdf`, `html`, `csv`, `image` (jpg), `dbf`, `xml`, `json`, `mht`, `odf`, `ods`, `odt`, `xls`, `docx`, `rtf`, `pptx`, `xaml`, `txt`, `svg`, `ps`, `ppml`.
 Параметры фильтра отчёта (отличаются от портального — добавляется имя):
 `f{paramID}_{Name}_v={value}      # имя регистрозависимо
f{paramID}_r=1                   # read-only
` ⚠️ В фильтре «категория» нельзя указать ID раздела — только список категорий раздела. Для просмотра ID параметра — настройки фильтра.

См. также:

- Пользователи и группы — администрирование

- Системные настройки

- Социальная сеть — администрирование
