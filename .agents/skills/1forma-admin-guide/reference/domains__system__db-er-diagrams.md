# Схемы базы данных: связи таблиц

Источник: https://help.1forma.ru/domains/system/db-er-diagrams/

Карта ключевых связей между таблицами БД 1Формы по подсистемам. Каждая схема — костяк часто используемых связей (внешних ключей), чтобы понять устройство подсистемы, не разбирая её в SSMS. Полные перечни колонок каждой таблицы — в администрировании соответствующего домена. Имена таблиц и полей на схемах сверены с актуальной структурой БД.
 Во всех схемах: в заголовке таблицы — техническое имя БД и русское название; у каждого поля — тип, имя, ключ (`PK` — первичный, `FK` — поле-ссылка на другую таблицу) и русское описание. Закреплена ли ссылка ключом в самой базе, показывает тип линии: сплошная — внешний ключ есть, пунктирная — связь логическая, поле хранит идентификатор, но ключом не защищено (удаление «родителя» такую ссылку не проверяет).

Центральная таблица подсистемы — `TaskSignatures` (запрошенные подписи): каждая строка связывает задачу (`TaskID`), пользователя-обработчика (`SignatureUserID`), подпись из справочника (`SignatureID`), её статус (`SignatureStateID`), тип резолюции (`SignatureResolutionTypeID`) и переход маршрута, на котором подпись запрошена (`SignatureStepID`). Дополнительные акцептанты хранятся в `TaskSignatureUsers`. Какие подписи запрашиваются на переходах, задаёт `StatesRoutesSignatures`; какие резолюции доступны для подписи — `SignatureResolutionTypesSignatures`.
 `erDiagram
    TaskSignatures["TaskSignatures · Запрошенные подписи"] {
        int ID PK "идентификатор записи"
        int TaskID FK "задача"
        int SignatureUserID FK "пользователь-обработчик"
        int SignatureID FK "подпись из справочника"
        int SignatureStateID FK "статус подписи"
        int SignatureResolutionTypeId FK "тип резолюции"
        int SignatureStepID FK "переход маршрута (NULL — динамическая)"
        bit IsActive "активна"
        bit IsSignatureOverdue "просрочена"
    }
    TaskSignatureUsers["TaskSignatureUsers · Акцептанты"] {
        int TaskSignatureID FK "запрошенная подпись"
        int UserID FK "акцептант"
        int TaskID FK "задача"
    }
    Tasks["Tasks · Задачи"] {
        int TaskID PK "идентификатор задачи"
        int StepID FK "последний выполненный переход"
    }
    Users["Users · Пользователи"] {
        int UserID PK "идентификатор пользователя"
        varchar DisplayName "отображаемое имя"
    }
    Signatures["Signatures · Справочник подписей"] {
        int SignatureID PK "идентификатор подписи"
        varchar Description "название"
        bit CanBeDynamic "может быть динамической"
    }
    SignaturesStates["SignaturesStates · Статусы подписей"] {
        int SignatureStateID PK "идентификатор статуса"
        varchar Descr "название статуса"
    }
    SignatureResolutionTypes["SignatureResolutionTypes · Типы резолюций"] {
        int ID PK "идентификатор типа"
        varchar Name "название"
        tinyint SignatureActionId "действие резолюции"
    }
    SignatureResolutionTypesSignatures["SignatureResolutionTypesSignatures · Доступные резолюции"] {
        int SignatureID FK "подпись"
        int SignatureResolutionTypeID FK "тип резолюции"
        bit IsVisible "видимость"
    }
    StatesRoutesSignatures["StatesRoutesSignatures · Подписи на переходах"] {
        int ID PK "идентификатор"
        int StepID FK "переход маршрута"
        int SignatureID FK "подпись"
        bit IsNecessarily "обязательная"
        varchar SPName "SQL-маршрут акцептантов"
    }
    StatesRoutesInSubcat["StatesRoutesInSubcat · Переходы маршрута"] {
        int StepID PK "идентификатор перехода"
        int SubcatID FK "категория"
        int StateID FK "исходный статус"
        int StateNextID FK "целевой статус"
    }

    Tasks ||--o{ TaskSignatures : "TaskID"
    Users ||..o{ TaskSignatures : "SignatureUserID"
    Signatures ||--o{ TaskSignatures : "SignatureID"
    SignaturesStates ||--o{ TaskSignatures : "SignatureStateID"
    SignatureResolutionTypes ||--o{ TaskSignatures : "SignatureResolutionTypeId"
    StatesRoutesInSubcat ||..o{ TaskSignatures : "SignatureStepID"
    TaskSignatures ||--o{ TaskSignatureUsers : "TaskSignatureID"
    Users ||--o{ TaskSignatureUsers : "UserID"
    Tasks ||--o{ TaskSignatureUsers : "TaskID"
    Signatures ||--o{ StatesRoutesSignatures : "SignatureID"
    StatesRoutesInSubcat ||--o{ StatesRoutesSignatures : "StepID"
    Signatures ||--o{ SignatureResolutionTypesSignatures : "SignatureID"
    SignatureResolutionTypes ||--o{ SignatureResolutionTypesSignatures : "SignatureResolutionTypeID"
    StatesRoutesInSubcat ||--o{ Tasks : "StepID"` Настройка и диагностика подписей — в signatures/admin.md.

Подсистема подписания через криптопровайдеры. `EdsExternalUsers` — пользователи, зарегистрированные для внешней ЭП; тип задаёт `EdsType` (0 — CryptoPro RuToken, 1 — PayControl, 2 — CryptoPro DSS). Для облачных провайдеров (PayControl, CryptoPro DSS) подписание идёт транзакцией `EdsExternalRequests` — она привязана к запрошенной подписи (`TaskSignatures`), подписываемому файлу (`FileStorageFiles`) и подписанту. Итог подписания фиксирует `EdsFileInfo` (по каждой подписи: файл с подписью, сертификат, дата, при необходимости — машиночитаемая доверенность `Mchd*`). `EdsDSSFileInfo` — промежуточная запись для формирования и проверки подписанного файла в DSS. Связи `EdsFileInfo` и `EdsDSSFileInfo` внешними ключами БД не закреплены — они логические; закреплены только связи `EdsExternalUsers` и `EdsExternalRequests`.
 `erDiagram
    EdsExternalUsers["EdsExternalUsers · Пользователи внешней ЭП"] {
        int Id PK "идентификатор записи"
        int UserId FK "пользователь"
        varchar ExternalUserId "ID пользователя у провайдера"
        varchar RegistrationInfo "данные регистрации (JSON)"
        bit IsActive "сертификат активен"
        datetime DateActivated "активирован"
        datetime DateDisabled "деактивирован"
        int EdsType "тип ЭП (0-RuToken, 1-PayControl, 2-DSS)"
    }
    EdsExternalRequests["EdsExternalRequests · Транзакции подписания"] {
        int Id PK "идентификатор записи"
        int TaskSignatureId FK "запрошенная подпись"
        int EdsExternalUserId FK "подписант"
        int FileId FK "подписываемый файл"
        varchar ExternalId "ID транзакции у провайдера"
        bit IsActive "транзакция активна"
        bit IsWaitingCancel "ожидает отмены"
        varchar Result "результат (JSON)"
        int EdsType "тип ЭП"
        datetime DateCreated "создана"
    }
    EdsFileInfo["EdsFileInfo · Файлы с подписью"] {
        int Id PK "идентификатор записи"
        int TaskId FK "задача"
        int SignatureId FK "подпись из справочника"
        int SignedByUserId FK "подписавший пользователь"
        int TaskSignatureId FK "запрошенная подпись"
        int SignedFileId FK "подписанный файл"
        bit IsInnerSign "подпись встроена в файл"
        int SignatureFileId FK "отдельный файл подписи (NULL — встроена)"
        datetime SignDate "дата подписания"
        varchar CertificateSerialNumber "серийный номер сертификата"
        varchar CertificateOwnerName "владелец сертификата"
        int StepId FK "переход маршрута"
        bit IsCosign "соподпись"
        int MchdXMLFileId FK "машиночитаемая доверенность (XML)"
    }
    EdsDSSFileInfo["EdsDSSFileInfo · Проверка файла с подписью (DSS)"] {
        int Id PK "идентификатор записи"
        int EdsId FK "запись EdsFileInfo"
        int CertificateId "сертификат"
        varchar CertificateName "название сертификата"
        int SignedFileId "подписанный файл"
        int SignedVersionId "версия файла"
        int TaskSignatureId "запрошенная подпись"
        varchar Content "содержимое"
    }
    Users["Users · Пользователи"] {
        int UserID PK "идентификатор пользователя"
        varchar DisplayName "отображаемое имя"
    }
    TaskSignatures["TaskSignatures · Запрошенные подписи"] {
        int ID PK "идентификатор записи"
        int TaskID FK "задача"
    }
    Signatures["Signatures · Справочник подписей"] {
        int SignatureID PK "идентификатор подписи"
        varchar Description "название"
    }
    Tasks["Tasks · Задачи"] {
        int TaskID PK "идентификатор задачи"
    }
    FileStorageFiles["FileStorageFiles · Файлы"] {
        int FileId PK "идентификатор файла"
        varchar FileName "имя файла"
    }

    Users ||--o{ EdsExternalUsers : "UserId"
    EdsExternalUsers ||--o{ EdsExternalRequests : "EdsExternalUserId"
    TaskSignatures ||--o{ EdsExternalRequests : "TaskSignatureId"
    FileStorageFiles ||--o{ EdsExternalRequests : "FileId"
    TaskSignatures ||..o{ EdsFileInfo : "TaskSignatureId"
    Users ||..o{ EdsFileInfo : "SignedByUserId"
    Tasks ||..o{ EdsFileInfo : "TaskId"
    Signatures ||..o{ EdsFileInfo : "SignatureId"
    FileStorageFiles ||..o{ EdsFileInfo : "SignedFileId"
    EdsFileInfo ||..o{ EdsDSSFileInfo : "EdsId"
    TaskSignatures ||..o{ EdsDSSFileInfo : "TaskSignatureId"` Настройка провайдеров ЭП и подписания — в signatures/admin.md.

Определение руководителей, заместителей и ролей вокруг таблицы `Users`. Руководители — по ролям (`RoleUser` → `Role`, при `DirectorsDetectionMode = 0`) или явно в `UserDirectors` с привязкой к орг.единице (при `DirectorsDetectionMode = 1`); заместители — `UserAssistant`; членство в группах — `UserGroups` → `Groups`; орг.единицы образуют иерархию через `OrgStructureUnit.ParentId`. Полномочия роли перечисляет `RolePowers`: внешних ключей у неё нет, а `RolePowerID` — код полномочия из перечисления, не ссылка на справочник.
 `erDiagram
    Users["Users · Пользователи"] {
        int UserID PK "идентификатор пользователя"
        varchar DisplayName "отображаемое имя"
    }
    UserGroups["UserGroups · Членство в группах"] {
        int UserID FK "пользователь"
        int GroupID FK "группа"
    }
    Groups["Groups · Группы пользователей"] {
        int GroupID PK "идентификатор группы"
        varchar Descr "название группы"
        int GroupTypeID FK "тип группы"
    }
    RoleUser["RoleUser · Роли пользователей в группах"] {
        int ID PK "идентификатор записи"
        int UserID FK "пользователь"
        int GroupID FK "группа"
        int RoleID FK "роль"
        bit IsGlobal "глобальная роль"
    }
    Role["Role · Справочник ролей"] {
        int ID PK "идентификатор роли"
        varchar Description "название роли"
        int CustomerID FK "юридическое лицо"
    }
    RolePowers["RolePowers · Полномочия ролей"] {
        int RoleID "роль"
        int RolePowerID "код полномочия (перечисление, не ссылка на таблицу)"
    }
    UserAssistant["UserAssistant · Заместители"] {
        int ID PK "идентификатор записи"
        int UserId FK "замещаемый пользователь"
        int AssistantUserID FK "заместитель"
        datetime DTFrom "начало замещения"
        datetime DTTo "окончание замещения"
        bit IsActual "актуально"
    }
    UserDirectors["UserDirectors · Руководители"] {
        int DirectorId FK "руководитель"
        int UserId FK "подчинённый"
        int OrgUnitId FK "орг.единица"
        int Level "уровень (1 — непосредственный)"
        bit IsPrimary "основной"
    }
    OrgStructureUnit["OrgStructureUnit · Орг.единицы"] {
        int Id PK "идентификатор орг.единицы"
        varchar Name "название"
        int ParentId FK "родительская орг.единица"
        int OrgStructureTypeId FK "тип орг.единицы"
        int LinkedGroupId FK "связанная группа"
        bit IsActual "действующая"
    }

    Users ||--o{ UserGroups : "UserID"
    Groups ||--o{ UserGroups : "GroupID"
    Users ||--o{ RoleUser : "UserID"
    Groups ||--o{ RoleUser : "GroupID"
    Role ||--o{ RoleUser : "RoleID"
    Role ||..o{ RolePowers : "RoleID"
    Users ||--o{ UserAssistant : "UserId"
    Users ||--o{ UserAssistant : "AssistantUserID"
    Users ||--o{ UserDirectors : "DirectorId"
    Users ||--o{ UserDirectors : "UserId"
    OrgStructureUnit ||--o{ UserDirectors : "OrgUnitId"
    OrgStructureUnit ||--o{ OrgStructureUnit : "ParentId"
    Groups ||--o{ OrgStructureUnit : "LinkedGroupId"` Настройка — в org-structure/admin.md.

Центральная таблица — `Tasks`: ссылается на заказчика (`UserID`), статус (`StateID`), категорию (`SubcatID`) и родительскую задачу для подзадач (`ParentTaskID`). Исполнители — `TaskHelpers`; категории (`Subcategories`) группируются в разделы (`Categories`, дерево через `ParentCategoryID`). Завершённой задача считается при `IsClosed = 1` и `States.FinishWork = 1`.
 `erDiagram
    Tasks["Tasks · Задачи"] {
        int TaskID PK "идентификатор задачи"
        int UserID FK "заказчик"
        int StateID FK "статус"
        int SubcatID FK "категория"
        int ParentTaskID FK "родительская задача"
        varchar Description "текст задачи"
        datetime CreatedTask "создание"
        datetime OrderedTime "срок"
        datetime TaskStartTime "начало работы"
        datetime EndTime "завершение"
        bit IsClosed "завершена"
    }
    TaskHelpers["TaskHelpers · Исполнители"] {
        int UserID FK "исполнитель"
        int TaskID FK "задача"
        bit IsResponsible "ответственный исполнитель"
        bit WorkFinished "работа завершена"
    }
    Users["Users · Пользователи"] {
        int UserID PK "идентификатор пользователя"
        varchar DisplayName "отображаемое имя"
        varchar Nick "логин"
    }
    States["States · Статусы"] {
        int StateID PK "идентификатор статуса"
        varchar Description "название статуса"
        bit FinishWork "работа завершена"
        bit CloseTask "закрывающий статус"
    }
    Subcategories["Subcategories · Категории"] {
        int SubcatID PK "идентификатор категории"
        int CategoryID FK "раздел"
        varchar Description "название категории"
        int SubcatTypeID "тип категории"
    }
    Categories["Categories · Разделы"] {
        int CategoryID PK "идентификатор раздела"
        int ParentCategoryID FK "родительский раздел"
        varchar Description "название раздела"
    }

    Users ||--o{ Tasks : "UserID"
    States ||--o{ Tasks : "StateID"
    Subcategories ||--o{ Tasks : "SubcatID"
    Tasks ||--o{ Tasks : "ParentTaskID"
    Users ||--o{ TaskHelpers : "UserID"
    Tasks ||--o{ TaskHelpers : "TaskID"
    Subcategories ||--o{ TaskHelpers : "SubcatId"
    Categories ||--o{ Subcategories : "CategoryID"
    Categories ||--o{ Categories : "ParentCategoryID"` Настройка и диагностика — в tasks/admin.md.

Маршрут категории — переходы между статусами — задаёт `StatesRoutesInSubcat` (из `StateID` в `StateNextID`). Движение задачи по маршруту логируется в `StepLog`, принудительные смены статуса вопреки маршруту — в `TaskStatusForcedChangesLog`. У `TaskStatusForcedChangesLog.OldState`/`NewState` жёсткого внешнего ключа на `States` нет — связь логическая.
 `erDiagram
    StepLog["StepLog · Движение задачи по маршруту"] {
        int LogID PK "идентификатор записи"
        int TaskID FK "задача"
        int StepID FK "переход маршрута"
        int UserID FK "кто выполнил"
        int PreviousStateID FK "статус до"
        int NewStateID FK "статус после"
        datetime ChangeDate "дата перехода"
    }
    StatesRoutesInSubcat["StatesRoutesInSubcat · Маршрут задач в категории"] {
        int StepID PK "идентификатор перехода"
        int SubcatID FK "категория"
        int StateID FK "исходный статус"
        int StateNextID FK "целевой статус"
        varchar StepDescr "название перехода"
    }
    Tasks["Tasks · Задачи"] {
        int TaskID PK "идентификатор задачи"
        int SubcatID FK "категория"
        int StateID FK "текущий статус"
        varchar Description "текст задачи"
    }
    States["States · Статусы"] {
        int StateID PK "идентификатор статуса"
        varchar Description "название статуса"
    }
    TaskStatusForcedChangesLog["TaskStatusForcedChangesLog · Принудительные переходы"] {
        int TaskStatusForcedChangesLogId PK "идентификатор записи"
        int TaskID FK "задача"
        int UserID FK "кто выполнил"
        int OldState "исходный статус"
        int NewState "целевой статус"
        datetime ChangeDate "дата перехода"
    }
    Users["Users · Пользователи"] {
        int UserID PK "идентификатор пользователя"
        varchar DisplayName "отображаемое имя"
    }
    Subcategories["Subcategories · Категории"] {
        int SubcatID PK "идентификатор категории"
        varchar Description "название категории"
    }

    Tasks ||--o{ StepLog : "TaskID"
    StatesRoutesInSubcat ||--o{ StepLog : "StepID"
    Users ||--o{ StepLog : "UserID"
    States ||--o{ StepLog : "PreviousStateID"
    States ||--o{ StepLog : "NewStateID"
    Subcategories ||--o{ StatesRoutesInSubcat : "SubcatID"
    States ||--o{ StatesRoutesInSubcat : "StateID"
    States ||--o{ StatesRoutesInSubcat : "StateNextID"
    Subcategories ||--o{ Tasks : "SubcatID"
    States ||--o{ Tasks : "StateID"
    Tasks ||--o{ TaskStatusForcedChangesLog : "TaskID"
    Users ||--o{ TaskStatusForcedChangesLog : "UserID"
    States ||..o{ TaskStatusForcedChangesLog : "OldState"
    States ||..o{ TaskStatusForcedChangesLog : "NewState"
    StatesRoutesInSubcat ||--o{ Tasks : "StepID"` Настройка маршрутов — в categories/admin.md; диагностика — в categories/routing-troubleshooting.md.

Подсистема учёта трудозатрат. Ресурс — это задача из категории «Справочник ресурсов» (роль, оборудование, сотрудник), поэтому `ResourceId` и `PerformerTaskId` (карточка ресурса-исполнителя) ссылаются на `Tasks`. `TaskResources` — привязка ресурсов к задаче (план в минутах/процентах). Плановые трудозатраты по датам — `TaskResourcePlanEntries`, фактические — `TaskResourceFactEntries` (с утверждением: `ApprovedAmount`, `ApprovedBy`, резолюция из `TaskResourceFactResolutions`). `TaskResourceFactAggregated` — суммарный факт по задаче (денормализация для отчётов). `TaskResourceLockedFactEntries` — периоды, закрытые для изменения факта. `SessionUserId` — кто внёс запись. Связи `ResourceId` в `TaskResources` и `TaskResourceLockedFactEntries` внешним ключом БД не закреплены — они логические.
 `erDiagram
    TaskResources["TaskResources · Ресурсы по задаче"] {
        int Id PK "идентификатор записи"
        int TaskId FK "задача"
        int ResourceId "ресурс (задача из справочника)"
        int Amount "план, минуты"
        float Percent "план, проценты"
        int PlannedAmount "плановые трудозатраты"
        bit IsAutoCalculated "рассчитано автоматически"
    }
    TaskResourcePlanEntries["TaskResourcePlanEntries · Плановые трудозатраты"] {
        int Id PK "идентификатор записи"
        int TaskId FK "задача"
        int ResourceId FK "ресурс (задача из справочника)"
        int PerformerTaskId FK "карточка ресурса-исполнителя"
        int PerformerUserId FK "исполнитель"
        int SessionUserId FK "кто внёс"
        int Amount "трудозатраты, минуты"
        date Date "дата"
        bit IsAutoCalculated "рассчитано автоматически"
    }
    TaskResourceFactEntries["TaskResourceFactEntries · Фактические трудозатраты"] {
        int Id PK "идентификатор записи"
        int TaskId FK "задача"
        int ResourceId FK "ресурс (задача из справочника)"
        int PerformerTaskId FK "карточка ресурса-исполнителя"
        int PerformerUserId FK "исполнитель"
        int SessionUserId FK "кто внёс"
        int ResolutionId FK "резолюция утверждения"
        int ApprovedBy FK "кто утвердил"
        int Amount "трудозатраты, минуты"
        int ApprovedAmount "утверждённые трудозатраты"
        date Date "дата"
    }
    TaskResourceFactAggregated["TaskResourceFactAggregated · Суммарный факт"] {
        int Id PK "идентификатор записи"
        int TaskId FK "задача"
        int ResourceId FK "ресурс (задача из справочника)"
        int PerformerTaskId FK "карточка ресурса-исполнителя"
        int PerformerUserId FK "исполнитель"
        int Amount "суммарные трудозатраты, минуты"
    }
    TaskResourceLockedFactEntries["TaskResourceLockedFactEntries · Закрытые периоды факта"] {
        int Id PK "идентификатор записи"
        int TaskId FK "задача"
        int PerformerTaskId FK "карточка ресурса-исполнителя"
        int PerformerUserId FK "исполнитель"
        int SessionUserId FK "кто закрыл период"
        int ResourceId "ресурс (задача из справочника)"
        datetime DateFrom "начало периода блокировки"
        datetime DateTo "конец периода блокировки"
    }
    TaskResourceFactResolutions["TaskResourceFactResolutions · Резолюции утверждения"] {
        int Id PK "идентификатор записи"
        varchar Description "резолюция"
    }
    Tasks["Tasks · Задачи и ресурсы"] {
        int TaskID PK "идентификатор задачи"
    }
    Users["Users · Пользователи"] {
        int UserID PK "идентификатор пользователя"
    }

    Tasks ||--o{ TaskResources : "TaskId"
    Tasks ||..o{ TaskResources : "ResourceId"
    Tasks ||..o{ TaskResourceLockedFactEntries : "ResourceId"
    Tasks ||--o{ TaskResourcePlanEntries : "TaskId"
    Tasks ||--o{ TaskResourcePlanEntries : "ResourceId"
    Tasks ||--o{ TaskResourcePlanEntries : "PerformerTaskId"
    Tasks ||--o{ TaskResourceFactEntries : "TaskId"
    Tasks ||--o{ TaskResourceFactEntries : "ResourceId"
    Tasks ||--o{ TaskResourceFactEntries : "PerformerTaskId"
    Tasks ||--o{ TaskResourceFactAggregated : "TaskId"
    Tasks ||--o{ TaskResourceFactAggregated : "ResourceId"
    Tasks ||--o{ TaskResourceFactAggregated : "PerformerTaskId"
    Tasks ||--o{ TaskResourceLockedFactEntries : "TaskId"
    Tasks ||--o{ TaskResourceLockedFactEntries : "PerformerTaskId"
    Users ||--o{ TaskResourcePlanEntries : "PerformerUserId"
    Users ||--o{ TaskResourcePlanEntries : "SessionUserId"
    Users ||--o{ TaskResourceFactEntries : "PerformerUserId"
    Users ||--o{ TaskResourceFactEntries : "SessionUserId"
    Users ||--o{ TaskResourceFactEntries : "ApprovedBy"
    Users ||--o{ TaskResourceFactAggregated : "PerformerUserId"
    Users ||--o{ TaskResourceLockedFactEntries : "PerformerUserId"
    Users ||--o{ TaskResourceLockedFactEntries : "SessionUserId"
    TaskResourceFactResolutions ||--o{ TaskResourceFactEntries : "ResolutionId"` Настройка ресурсного планирования — в resources/admin.md.

Членство пользователя в группах — `UserGroups` (`Users` ↔ `Groups`), в орг.единицах — `UserOrgStructureUnit` (`Users` ↔ `OrgStructureUnit`). Орг.единицы образуют иерархию через `OrgStructureUnit.ParentId`. Наличие записи в `UserOrgStructureUnit` для орг.единицы означает должность, иначе — подразделение.
 `erDiagram
    Users["Users · Пользователи"] {
        int UserID PK "идентификатор пользователя"
        varchar DisplayName "отображаемое имя"
        varchar Nick "логин"
    }
    UserGroups["UserGroups · Членство в группах"] {
        int UserID FK "пользователь"
        int GroupID FK "группа"
    }
    Groups["Groups · Группы пользователей"] {
        int GroupID PK "идентификатор группы"
        varchar Descr "название группы"
    }
    UserOrgStructureUnit["UserOrgStructureUnit · Членство в орг.единицах"] {
        int UserID FK "пользователь"
        int OrgStructureUnitId FK "орг.единица (должность)"
        bit IsPrimary "основная должность"
        bit IsImported "импортирована из AD"
    }
    OrgStructureUnit["OrgStructureUnit · Орг.единицы"] {
        int Id PK "идентификатор орг.единицы"
        varchar Name "название"
        int ParentId FK "родительская орг.единица"
        bit IsActual "действующая"
    }

    Users ||--o{ UserGroups : "UserID"
    Groups ||--o{ UserGroups : "GroupID"
    Users ||--o{ UserOrgStructureUnit : "UserID"
    OrgStructureUnit ||--o{ UserOrgStructureUnit : "OrgStructureUnitId"
    OrgStructureUnit ||--o{ OrgStructureUnit : "ParentId"` Настройка — в users-and-groups/admin.md и users-and-groups/business.md.

Значения ДП хранятся в `ExtParamValues` (ключ — пара «задача + ДП»): простые типы — в типизированных колонках, Lookup — `SelectedTaskId`, адрес — ссылкой на `Address`. Сложные типы вынесены в отдельные таблицы (Таблица, Выбор задач/пользователей, Файл). ДП «Файл из файлохранилища» ссылается на `FileStorageFiles` не через прямой внешний ключ (колонки `TaskID`/`ExtParamId` там вычисляемые), поэтому файлохранилище на схеме не показано; колонки ДП «Таблица» задаёт справочник `ExtParamTableSettings`.
 `erDiagram
    ExtParams["ExtParams · Справочник ДП"] {
        int ExtParamID PK "идентификатор ДП"
    }
    ExtParamValues["ExtParamValues · Значения ДП в задачах"] {
        int TaskID PK "задача"
        int ExtParamID PK "ДП"
        varchar ExtParamValue "значение строкой"
        datetime DateTimeValue "дата"
        money MoneyValue "деньги"
        decimal DecimalValue "число"
        bit BoolValue "чекбокс"
        int SelectedTaskId FK "выбранная задача (Lookup)"
        int AddressId FK "адрес"
        int SelectUserValue "выбранный пользователь"
        bit IsEncrypted "зашифровано"
    }
    Tasks["Tasks · Задачи"] {
        int TaskID PK "идентификатор задачи"
    }
    Address["Address · Адресный классификатор"] {
        int Id PK "идентификатор адреса"
        varchar FullAddress "полный адрес"
        varchar City "город"
        decimal Latitude "широта"
        decimal Longitude "долгота"
    }
    ExtParamTableValues["ExtParamTableValues · Значение ДП «Таблица»"] {
        int TaskID PK "задача"
        int RowID PK "строка"
        int ColumnID PK "колонка (ExtParamTableSettings)"
        varchar Value "значение строкой"
    }
    ExtParamValueSelectedTasks["ExtParamValueSelectedTasks · ДП «Выбор задач»"] {
        int TaskID PK "задача"
        int ExtParamID PK "ДП"
        int SelectedTaskID PK "выбранная задача"
        int FolderId FK "папка"
    }
    ExtParamValueSelectedTaskFolders["ExtParamValueSelectedTaskFolders · Папки выбора задач"] {
        int Id PK "идентификатор папки"
        int TaskId FK "задача"
        int ExtParamId FK "ДП"
        varchar Name "название вкладки"
    }
    ExtParamSelectUsersValues["ExtParamSelectUsersValues · ДП «Выбор пользователей»"] {
        int Id PK "идентификатор записи"
        int TaskID FK "задача"
        int ExtParamID FK "ДП"
        int UserID FK "пользователь"
    }
    ExtParamSelectUsersValuesGroups["ExtParamSelectUsersValuesGroups · Выбор пользователей (группы)"] {
        int Id PK "идентификатор записи"
        int TaskID FK "задача"
        int ExtParamID FK "ДП"
        int GroupID FK "группа"
    }
    ExtParamsFiles["ExtParamsFiles · ДП «Файл»"] {
        int id PK "идентификатор записи"
        int TaskID FK "задача"
        int ExtParamID FK "ДП"
        int UserID FK "кто вложил"
        varchar FileName "имя файла"
        int FileSize "размер"
    }
    Users["Users · Пользователи"] {
        int UserID PK "идентификатор пользователя"
    }
    Groups["Groups · Группы"] {
        int GroupID PK "идентификатор группы"
    }
    OrgStructureUnit["OrgStructureUnit · Орг.единицы"] {
        int Id PK "идентификатор орг.единицы"
    }
    ExtParamSelectUsersValuesOrgUnits["ExtParamSelectUsersValuesOrgUnits · Выбор пользователей (орг.единицы)"] {
        int Id PK "идентификатор записи"
        int TaskID FK "задача"
        int ExtParamID FK "ДП"
        int OrgStructureUnitID FK "орг.единица"
    }

    Tasks ||--o{ ExtParamValues : "TaskID"
    ExtParams ||--o{ ExtParamValues : "ExtParamID"
    Tasks ||--o{ ExtParamValues : "SelectedTaskId"
    Address ||--o{ ExtParamValues : "AddressId"
    Tasks ||--o{ ExtParamTableValues : "TaskID"
    Tasks ||--o{ ExtParamValueSelectedTasks : "TaskID"
    ExtParams ||--o{ ExtParamValueSelectedTasks : "ExtParamID"
    Tasks ||--o{ ExtParamValueSelectedTasks : "SelectedTaskID"
    ExtParamValueSelectedTaskFolders ||--o{ ExtParamValueSelectedTasks : "FolderId"
    Tasks ||--o{ ExtParamValueSelectedTaskFolders : "TaskId"
    ExtParams ||--o{ ExtParamValueSelectedTaskFolders : "ExtParamId"
    Tasks ||--o{ ExtParamSelectUsersValues : "TaskID"
    ExtParams ||--o{ ExtParamSelectUsersValues : "ExtParamID"
    Users ||--o{ ExtParamSelectUsersValues : "UserID"
    Tasks ||--o{ ExtParamSelectUsersValuesGroups : "TaskID"
    ExtParams ||--o{ ExtParamSelectUsersValuesGroups : "ExtParamID"
    Groups ||--o{ ExtParamSelectUsersValuesGroups : "GroupID"
    Tasks ||--o{ ExtParamsFiles : "TaskID"
    ExtParams ||--o{ ExtParamsFiles : "ExtParamID"
    Users ||--o{ ExtParamsFiles : "UserID"
    Tasks ||--o{ ExtParamSelectUsersValuesOrgUnits : "TaskID"
    ExtParams ||--o{ ExtParamSelectUsersValuesOrgUnits : "ExtParamID"
    OrgStructureUnit ||--o{ ExtParamSelectUsersValuesOrgUnits : "OrgStructureUnitID"` Значения ДП «Выбор пользователей» хранятся раздельно по видам: пользователи — `ExtParamSelectUsersValues`, группы — `ExtParamSelectUsersValuesGroups`, орг.единицы — `ExtParamSelectUsersValuesOrgUnits`.
 Настройка — в ext-params/admin.md и ext-params/business.md.

Для быстрого чтения и фильтрации значений ДП «Таблица» система генерирует по одному представлению (`VIEW`) на каждый такой ДП: `ExtParamTable{ExtParamID}Denormalized` (например, `ExtParamTable6261Denormalized`). Представление разворачивает нормализованные строки `ExtParamTableValues` в плоскую форму: строка — запись таблицы, колонки — `Column{ID}Value` (значение строкой) и `Column{ID}NativeValue` (форматированное). Источник данных — `ExtParamTableValues`; сами представления пересобираются при изменении структуры ДП.
 `graph LR
    EPTV["ExtParamTableValues — значения ДП «Таблица»"]
    DNT["ExtParamTable{ExtParamID}Denormalized — VIEW, плоское представление"]
    T["Tasks — задачи"]

    EPTV -- "разворачивается в" --> DNT
    DNT -- "TaskID" --> T` Настройка ДП «Таблица» — в ext-params/table/settings-reference.md.

Письма — `Emails` (привязаны к ящику `EmailMailBoxes`). Получатели — `EmailRecipients`, раскладка по папкам — `EmailMailBoxesFoldersEmails` (папки `EmailMailBoxesFolders`, дерево через `ParentFolderId`), связь с задачей — `EmailTaskLinks` и поле `Emails.TaskId`. Доступ к ящикам — `EmailMailBoxesUsers`, адресные книги — `EmailAddressBooks`. Часть связей внешними ключами БД не закреплена (создатель письма, получатели, адресные книги) — они логические.
 `erDiagram
    Emails["Emails · Письма"] {
        int EmailID PK "идентификатор письма"
        int MailBoxID FK "почтовый ящик"
        int CreatorID "создатель (NULL — извне)"
        int TaskId FK "связанная задача"
        varchar Subject "тема"
        varchar HtmlBody "тело (HTML)"
        varchar From "отправитель"
        varchar To "получатель"
        datetime DateSent "отправлено"
        bit IsDraft "черновик"
        bit IsUnread "не прочитано"
        bit IsDeleted "удалено"
    }
    EmailMailBoxes["EmailMailBoxes · Почтовые ящики"] {
        int MailBoxID PK "идентификатор ящика"
        varchar MailBoxName "название ящика"
    }
    EmailMailBoxesUsers["EmailMailBoxesUsers · Ящики пользователей"] {
        int MailBoxID FK "ящик"
        int UserID FK "пользователь"
        bit IsOwner "владелец"
    }
    EmailMailBoxesFolders["EmailMailBoxesFolders · Папки ящиков"] {
        int FolderID PK "идентификатор папки"
        int MailBoxID FK "ящик"
        int ParentFolderId FK "родительская папка"
        varchar FolderName "название папки"
        int FolderType "тип папки"
    }
    EmailMailBoxesFoldersEmails["EmailMailBoxesFoldersEmails · Письмо в папке"] {
        int FolderId FK "папка"
        int EmailId FK "письмо"
    }
    EmailRecipients["EmailRecipients · Получатели"] {
        int UserId FK "получатель"
        int EmailId FK "письмо"
        bit IsUnread "не прочитано"
        datetime ReadDate "дата прочтения"
    }
    EmailTaskLinks["EmailTaskLinks · Связь письма с задачей"] {
        int TaskId FK "задача"
        int EmailId FK "письмо"
    }
    EmailAddressBooks["EmailAddressBooks · Адресные книги"] {
        int EntryId PK "идентификатор записи"
        int MailBoxId FK "ящик"
        varchar Name "имя"
        varchar EmailAddress "email-адрес"
    }
    EmailAddressBooksEmails["EmailAddressBooksEmails · Адресат письма"] {
        int EntryId FK "запись адресной книги"
        int EmailId FK "письмо"
        int RecipientType "тип (0-От,1-Кому,2-Копия,3-скрытая)"
    }
    Tasks["Tasks · Задачи"] {
        int TaskID PK "идентификатор задачи"
    }
    Users["Users · Пользователи"] {
        int UserID PK "идентификатор пользователя"
    }

    EmailMailBoxes ||--o{ Emails : "MailBoxID"
    Tasks ||--o{ Emails : "TaskId"
    Users ||..o{ Emails : "CreatorID"
    EmailMailBoxes ||--o{ EmailMailBoxesUsers : "MailBoxID"
    Users ||--o{ EmailMailBoxesUsers : "UserID"
    EmailMailBoxes ||..o{ EmailMailBoxesFolders : "MailBoxID"
    EmailMailBoxesFolders ||--o{ EmailMailBoxesFolders : "ParentFolderId"
    EmailMailBoxesFolders ||--o{ EmailMailBoxesFoldersEmails : "FolderId"
    Emails ||--o{ EmailMailBoxesFoldersEmails : "EmailId"
    Emails ||..o{ EmailRecipients : "EmailId"
    Users ||..o{ EmailRecipients : "UserId"
    Tasks ||--o{ EmailTaskLinks : "TaskId"
    Emails ||--o{ EmailTaskLinks : "EmailId"
    EmailMailBoxes ||..o{ EmailAddressBooks : "MailBoxId"
    EmailAddressBooks ||..o{ EmailAddressBooksEmails : "EntryId"
    Emails ||..o{ EmailAddressBooksEmails : "EmailId"` Настройка — в mail/admin.md и mail/business.md.
