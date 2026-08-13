# Контролы формы — Администрирование

Источник: https://help.1forma.ru/domains/form-controls/admin/

Документ описывает администрирование поведения контролов дополнительных параметров в MTF/NTF: настройки ДП и категории, влияющие на рендер, ввод, валидацию и сохранение. Аудитория — администраторы конфигурации. Внутри — обзор домена, механизмы AdminSPA/API, карта групп контролов, runtime-совместимость MTF/NTF, диагностика БД и типичные ошибки настройки.

`form-controls` описывает администрирование поведения контролов дополнительных параметров в MTF/NTF: какие настройки ДП и категории влияют на рендер, ввод, валидацию, сохранение и обратное чтение значения.
 Администратор настраивает:

- глобальный тип и type-specific настройки ДП;

- привязку ДП к категории, порядок, обязательность, скрытие, readonly и размещение в блоках;

- параметры конкретных групп контролов: File/MultiFile, Lookup/MultiLookup, SelectUsers, Table EP;

- совместимость сценария со старым MTF, новым MTF и NTF;

- диагностику сохранения значения через таблицы настроек, runtime-значений и link-таблицы.
  Граница домена:

- `form-controls` — поведение конкретного контрола ДП на форме;

- `../ext-params/admin.md` — жизненный цикл ДП как сущности, Admin API и глобальные настройки;

- `../categories/admin.md` — включение ДП в категорию и category-level флаги;

- `../task-forms/admin.md` — layout формы, блоки, порядок элементов, тулбар и JS/CSS-вставки.



Отдельной формы автоадминки только для домена `form-controls` не выявлено. Администрирование распределено между общими контурами ДП и категории.
 Механизм Где настраивается Что контролирует AdminSPA / Admin API ДП `../ext-params/admin.md` глобальная карточка ДП, тип, type-specific настройки, связи ДП Настройки категории -> ДП `../categories/admin.md` привязка ДП к категории, порядок, обязательность, скрытие, подсказки, блоки Настройки формы задачи `../task-forms/admin.md` контейнеры MTF/NTF, layout, порядок блоков и JS/CSS-вставки Runtime MTF/NTF `frontend.md`, `data-flow.md` фактический рендер, сохранение, refresh и realtime-обновления `EntityEditor`-схем для `form-controls` как самостоятельного домена в прочитанных источниках не выявлено.

Профильные административные операции выполняются через контур `ext-params`.
 Контроллер / маршрут Назначение для form-controls `/api/admin/extparams` CRUD глобального ДП и базовой модели контрола `/api/admin/subcat/extparams` привязка ДП к категории, порядок, category-level настройки `/api/admin/extparams/settings` чтение и сохранение type-specific настроек ДП `/api/admin/subcat/extparams/blocks` блоки и группы блоков, в которых размещаются контролы `/api/admin/ext-params/links` master-slave связи ДП, влияющие на Lookup/MultiLookup `/api/admin/extparam/options` опции list/combobox и смежных выборов `/api/admin/ext-params/table` шаблон и рендер ДП типа `Table` Подробный список методов и административных сценариев находится в `../ext-params/admin.md`.

Таблица показывает, к каким таблицам настроек относится каждая группа контролов.
 Группа Тип ДП Таблицы настроек Основной справочник File / MultiFile `File` `ExtParamsFileSettings` `controls-reference.md` Lookup `LookUpField` `LookupParamSettings` `controls-reference.md` MultiLookup `MultiSlctSubcatTasks` `LookupParamSettings`, `ExtParamValueSelectedTask*` `controls-reference.md` SelectUsers `SelectUsers` `ExtParamSelectUserSettings`, `ExtParamSelectUserSettingsOrgUnitTypes` `controls-reference.md` Table EP `Table` `ExtParamTableCommonSettings`, `ExtParamTableSettings`, `ExtParamTableSections` `controls-reference.md` ⚠️ Этот файл не дублирует полный перечень type-specific параметров. Для выбора конкретной настройки использовать `controls-reference.md`, а для ограничений старый MTF / новый MTF / NTF — `compatibility-matrix.md`.

Фактическое поведение контрола на форме определяется не только глобальным типом ДП, но и привязкой к категории.
 Настройка Где владелец Эффект для form-controls Порядок ДП в категории `../categories/admin.md` порядок рендера контролов в MTF/NTF Блок ДП и группа блоков `../task-forms/admin.md` контейнер и визуальная группировка контролов Количество занимаемых колонок `../categories/admin.md`, `../task-forms/admin.md` ширина контрола в форме Скрытый / скрыть при постановке / скрыть в карточке `../categories/admin.md` наличие контрола в NTF и MTF Обязательный / readonly / права по статусам `../ext-params/admin.md`, `../categories/admin.md` валидация и доступность ввода Подсказка и smart-подсказка `../categories/admin.md` текст или smart-результат рядом с контролом Master-slave связи ДП `../ext-params/admin.md` фильтрация зависимых Lookup/MultiLookup контролов ⚠️ Если поле не отображается на форме, сначала проверять category-level привязку и блоки. Если поле отображается, но не сохраняется или сохраняется не тем форматом, переходить к type-specific настройкам и `data-flow.md`.

Собственные `SettingsCustom` или `sys_general_settings` ключи для `form-controls` как отдельного домена в прочитанных источниках не выявлены.
 Связанные настройки находятся у доменов-владельцев:
 Ключ / настройка Где описано Что влияет `HideEmptyEpOnNtf` `../categories/admin.md` скрытие пустых ДП на форме создания задачи `UseNewMTF`, `UseNewMTFStyle` `../task-forms/admin.md` выбор runtime MTF и стиль карточки задачи type-specific настройки ДП `../ext-params/admin.md`, `controls-reference.md` поведение конкретных контролов

При настройке контролов учитывать режим формы.
 Режим Что важно администратору Старый MTF часть сценариев поддерживается ограниченно, особенно для табличного ДП и realtime Новый MTF основной runtime для интерактивных контролов и полного поведения Table EP NTF create-flow до создания задачи; часть сценариев File/Table/Lookup зависит от контекста будущей задачи Полная матрица поддержки по группам контролов вынесена в `compatibility-matrix.md`.
 ⚠️ Для инцидентов "работает в MTF, но не работает в NTF" сначала проверять, требует ли сценарий уже существующий `TaskID`, строки таблицы или link к файлу. Эти цепочки описаны в `data-flow.md`.

Таблицы базы данных, задействованные при диагностике контролов, настроек и runtime-значений ДП.
 Объект Назначение `ExtParams` глобальная карточка ДП и тип контрола `ExtParamsInSubcat` привязка ДП к категории и category-level настройки `ExtParamValues` базовое значение ДП в задаче `ExtParamsFileSettings` настройки File/MultiFile `LookupParamSettings` настройки Lookup/MultiLookup `ExtParamSelectUserSettings`, `ExtParamSelectUserSettingsOrgUnitTypes` настройки SelectUsers `ExtParamTableCommonSettings`, `ExtParamTableSettings`, `ExtParamTableSections` настройки Table EP `ExtParamTableValues` значения ячеек табличного ДП `ExtParamValueSelectedTasks`, `ExtParamValueSelectedTaskFolders` выбранные задачи и папки MultiLookup `ExtParamSelectUsersValues*` значения SelectUsers по пользователям, группам и оргструктуре `FileStorageFileToExtParamLinks` связь файлов с ДП File/MultiFile и файловыми колонками таблицы

SQL-запросы для быстрой диагностики значения, привязки и настроек ДП в задаче.
 `-- Глобальная карточка ДП
select *
from ExtParams
where ExtParamID = @extParamId;

-- Привязка ДП к категории
select *
from ExtParamsInSubcat
where ExtParamID = @extParamId
  and SubcatID = @subcatId;

-- Runtime-значение ДП в задаче
select *
from ExtParamValues
where ExtParamID = @extParamId
  and TaskID = @taskId;

-- File/MultiFile: есть ли link к ДП
select *
from FileStorageFileToExtParamLinks
where ExtParamID = @extParamId
  and TaskID = @taskId;
` Для Lookup/MultiLookup, SelectUsers и Table EP использовать специализированные таблицы из `data-flow.md`, потому что один `ExtParamValues` не показывает всю модель значения.

Таблица связывает симптом с наиболее вероятной точкой проверки и документом.
 Симптом Что проверить первым Где подробности Контрол не отображается привязка в `ExtParamsInSubcat`, скрытие, блоки, режим NTF/MTF `../categories/admin.md`, `../task-forms/admin.md` Контрол виден, но readonly права ДП, права по статусам, smart-access, режим формы `../ext-params/admin.md` Lookup/MultiLookup пустой источник, smart-фильтр, master-slave связи, доступ к задачам источника `controls-reference.md`, `data-flow.md` SelectUsers показывает не тот набор `AllowUsers` / `AllowGroups` / `AllowOrgUnits`, фильтрации по группе/оргструктуре/smart-фильтру `controls-reference.md` File загружается, но не отображается link в `FileStorageFileToExtParamLinks`, ограничения размера/расширений, источник файла `data-flow.md` Table EP не сохраняет строки настройки колонок, readonly/required/key column, denorm-view и reload `data-flow.md`, `backend.md` Сценарий работает в MTF, но не в NTF наличие `TaskID`, create-flow ограничения, матрица совместимости `compatibility-matrix.md`

Документы, покрывающие смежные аспекты поведения контролов в форме задачи.

- `business.md` — бизнес-границы и типовые пользовательские сценарии.

- `backend.md` — контроллеры, сервисы и таблицы настроек по группам контролов.

- `frontend.md` — SPA-компоненты, модели и состояния контролов.

- `data-flow.md` — сквозные цепочки чтения, сохранения, upload, Table EP и refresh.

- `controls-reference.md` — справочник групп контролов и ключевых type-specific настроек.

- `compatibility-matrix.md` — совместимость старый MTF / новый MTF / NTF.

- `../ext-params/admin.md` — администрирование ДП как сущностей и Admin API.

- `../categories/admin.md` — привязка ДП к категории и category-level поведение.

- `../task-forms/admin.md` — layout MTF/NTF, блоки, порядок и JS/CSS-вставки.
