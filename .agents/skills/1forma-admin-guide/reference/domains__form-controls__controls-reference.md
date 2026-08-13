# Контролы формы — Справочник

Источник: https://help.1forma.ru/domains/form-controls/controls-reference/

Справочник типовых контролов формы (ДП): карта типов, ключевые настройки по группам, быстрый выбор документа для диагностики и важные ограничения. Для инженеров поддержки при разборе инцидентов с формами задач.

Таблица связывает группу контрола с типом ДП, таблицей настроек и основным документом.
 Группа Тип ДП Ключевая таблица настроек Основной документ Деньги `Money` `ExtParamNumericSettings` (общие для числовых) `docs/domains/ext-params/types-reference.md`; триггеры сохранения: blur, Enter, Tab (v2.267+), pointerdown Число `NumericValue` `ExtParamNumericSettings` `docs/domains/ext-params/types-reference.md`; триггеры сохранения: blur, Enter, Tab, pointerdown File / MultiFile `File` `ExtParamsFileSettings` `docs/domains/ext-params/file/settings-reference.md` Lookup `LookUpField` `LookupParamSettings` `docs/domains/ext-params/lookup/settings-reference.md` MultiLookup `MultiSlctSubcatTasks` `LookupParamSettings` + `ExtParamValueSelectedTask*` `docs/domains/ext-params/lookup/settings-reference.md` SelectUsers `SelectUsers` `ExtParamSelectUserSettings` `docs/domains/ext-params/select-users/settings-reference.md` Table EP `Table` `ExtParamTableCommonSettings`, `ExtParamTableSettings` `docs/domains/ext-params/table/settings-reference.md`

File / MultiFile: `IsMultifile`, `RenameMethod`, `RenameOnUpload`, `SmartRenameOnUpload`, `MaxFileSizeKb`, `ExtParamFileSource`, `ProtectFileView`, `IsLogFileReadsAction`, `FileProviderId`.
 Lookup / MultiLookup: `SubcatId` / `UnionId` (источник), `SmartFilterId` (фильтр источника), `ShowAsRadioButtons`, `DisplayAsText`, `IsHierarchical`, каскадные связи (через `ExtParamLink`).

 SelectUsers: `SingleUserMode`, `AllowUsers`, `AllowGroups`, `AllowOrgUnits`, `TypeFiltration`, `GroupId`, `SmartFilterId`, `OrgUnitTypeFiltration`, `OrgUnitId`, `OrgUnitSmartFilterId`.
 Table EP: общие настройки таблицы (`DisplayMode`, `AutosaveEnabled`, `ImportEnabled`, `ExportEnabled`), настройки колонок (`Type`, required/readonly/hidden, default/smart value), секции (`ExtParamTableSections`), специализированные типовые настройки колонок (Lookup/File/SelectUsers/etc.).

Быстрый выбор документа для диагностики конкретного контрола или известного ограничения.
 Вопрос Документ Не загружается файл / ограничения по размеру `docs/domains/ext-params/file/settings-reference.md` Lookup показывает пустой список `docs/domains/ext-params/lookup/settings-reference.md` SelectUsers отображает «не тех» пользователей `docs/domains/ext-params/select-users/settings-reference.md` Таблица не сохраняет/не валидирует строки `docs/domains/ext-params/table/settings-reference.md` Нужна серверная трассировка конкретного типа `docs/domains/form-controls/{type}/backend.md` Нужен фронтовый слой конкретного типа `docs/domains/form-controls/{type}/frontend.md` Иерархический Lookup показывает задачи в неправильных статусах `docs/domains/ext-params/lookup/settings-reference.md` Важные различия между типами контролов:

- `Lookup` и `MultiLookup` имеют общую таблицу настроек, но разные модели хранения значений.

- `File` и `MultiFile` — один тип ДП с переключением через `IsMultifile`.

- `SelectUsers` хранит три независимые группы значений (users/groups/orgunits).

- `Table EP` одновременно использует и общие, и типоспецифические настройки колонок.
  Связанные документы: `docs/domains/form-controls/compatibility-matrix.md`, `docs/domains/form-controls/backend.md`, `docs/domains/form-controls/frontend.md`, `docs/domains/form-controls/data-flow.md`.
