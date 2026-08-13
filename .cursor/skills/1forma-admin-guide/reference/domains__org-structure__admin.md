# Организационная структура

Источник: https://help.1forma.ru/domains/org-structure/admin/

Руководство по настройке организационной структуры 1Формы — дерева подразделений и должностей компании. Администрирование: Администрирование → Оргструктура (`/spa/administration/org-structure`). Описаны типы орг. единиц, карточки единиц, массовые операции, справочник должностей, пользовательские настройки и SQL-диагностика.
 Организационная структура — дерево подразделений и должностей компании. Настраивается в администрировании: Администрирование → Оргструктура (`/spa/administration/org-structure`). Здесь создаётся дерево единиц, задаются их типы и должности, к единицам привязываются пользователи и группы.

 Настраивать оргструктуру можно несколькими способами:
 Интерфейс Путь Администрирование (AdminSPA) Администрирование → Оргструктура — `/spa/administration/org-structure` Admin API `/api/admin/org-structure/*`

Таблица БД: `dbo.OrgStructureType`
 Тип задаёт уровень и поведение орг. единицы. Поля типа:
 Поле Назначение Name Название типа (Филиал, Департамент, Отдел, Группа, Должность) Order Числовой уровень — дочерний элемент должен иметь строго больший Order IsPosition Признак «должность» — только один тип с этим флагом; элементы этого типа могут иметь Appointment IsCommercialInfo, HideInUserInfo Флаги видимости Типы настраиваются до создания оргструктуры.

 Новый тип создаётся в форме с полями «Имя», «Уровень» и признаком «Является должностью».

Таблица БД: `dbo.OrgStructureUnit`
 Орг. единица — узел структуры (филиал, отдел, должность и т. д.). Создаётся и редактируется через карточку единицы: при создании указываются имя, родитель («Входит в») и тип.

Настройки:

- Имя, Тип, Входит в (ParentId)

- При выборе значения «Входит в» в списке доступны только организационные единицы, которые могут быть родительскими. Единицы типа с флагом `IsPosition` не показываются, потому что должности нельзя использовать как родительские узлы.

- Порядковый номер типа (`Order`) сам по себе не определяет, может ли единица быть родительской. Для фильтрации используется именно признак `IsPosition`, поэтому в списке могут отображаться единицы с любым уровнем, если они не являются должностями.

- Связанная группа (LinkedGroupId) — привязка к группе для автосинхронизации. У связанной группы поле «Название» в админке групп заблокировано (с 2.268.348): при переименовании орг.единицы имя связанной группы обновляется автоматически.

- Должность (AppointmentId) — только для типа IsPosition

- Является руководящей (IsDirector) — определяет руководителя подчинённых

- Актуальность (IsActual) — неактуальные отображаются серым

- Не показывать в оргструктуре (DoNotShowInOrgStructure)

- Функциональная (IsFuncGroup)

- Комментарий (Comment) — отображается под названием в UI

- Цвет (Color)

 Состав: список пользователей в данной единице (UserOrgStructureUnit).

 Должности: назначение должности из справочника OrgStructureAppointment.
 Над поддеревом оргструктуры доступны массовые операции:
 Операция API Связать поддерево с группами `POST /api/admin/org-structure/orgunit/{unitId}/link-sub-tree-with-group` Синхронизировать группы `POST /api/admin/org-structure/syncgroups` Экспорт в Excel `GET /api/admin/org-structure/export-to-excel?withUsers={bool}` Справочник должностей (`dbo.OrgStructureAppointment`) — простой справочник (ID, Name). Должность назначается конкретной орг. единице через хранимую процедуру `SetAppointmentForOrgUnit`.

Поведение оргструктуры можно дополнительно настроить через настройки приложения:
 Настройка Что делает `OrgStructure_AllowNonUniqueOrgUnitNames` Разрешить/запретить дубликаты имён в одной ветке (по умолчанию: разрешено) `CustomUserDirectorIds` Ручное переопределение руководителя: `userId1:directorId1,userId2:directorId2` `customWorkersDictionarySP` Пользовательская хранимая процедура (SP) вместо стандартного справочника сотрудников `IsOrgUnitAutoCreationGroup` При включении создание орг. единицы автоматически создаёт и привязывает к ней связанную группу SQL-диагностика — запросы для проверки состояния оргструктуры (выполняются в базе данных 1Ф):
 `-- Дерево единиц
SELECT Id, Name, ParentId, OrgStructureTypeId, LinkedGroupId, IsActual, IsDirector
FROM dbo.OrgStructureUnit
ORDER BY ParentId, Name;

-- Типы
SELECT * FROM dbo.OrgStructureType ORDER BY [Order];

-- Должности
SELECT * FROM dbo.OrgStructureAppointment;

-- Пользователи в оргструктуре
SELECT u.UserID, u.OrgStructureUnitId, u.IsPrimary, osu.Name
FROM dbo.UserOrgStructureUnit u
JOIN dbo.OrgStructureUnit osu ON osu.Id = u.OrgStructureUnitId;

-- Единицы без связанной группы (потенциальная проблема)
SELECT Id, Name
FROM dbo.OrgStructureUnit
WHERE LinkedGroupId IS NULL AND IsActual = 1;

-- Пользователи без основной единицы
SELECT UserID
FROM dbo.UserOrgStructureUnit
GROUP BY UserID
HAVING SUM(CASE WHEN IsPrimary = 1 THEN 1 ELSE 0 END) = 0;
`

Определение руководителей, заместителей и ролей опирается на несколько таблиц вокруг `Users`. Руководители задаются либо ролями (`RoleUser` → `Role`, при `Settings.DirectorsDetectionMode = 0`), либо явно в `UserDirectors` с привязкой к орг.единице (`OrgStructureUnit`, при `DirectorsDetectionMode = 1`). Заместители хранятся в `UserAssistant`. Членство в группах — `UserGroups` → `Groups`; орг.единицы образуют иерархию через само-ссылку `OrgStructureUnit.ParentId`.
 `graph LR
    U["Users — пользователи"]
    UG["UserGroups — членство в группах"]
    G["Groups — группы"]
    RU["RoleUser — роли в группах"]
    R["Role — справочник ролей"]
    RP["RolePowers — назначения ролей"]
    UA["UserAssistant — заместители"]
    UD["UserDirectors — руководители"]
    OSU["OrgStructureUnit — орг.единицы"]

    UG -- "UserID" --> U
    UG -- "GroupID" --> G
    RU -- "UserID" --> U
    RU -- "GroupID" --> G
    RU -- "RoleID" --> R
    RP -- "RoleID / RolePowerID" --> R
    UA -- "UserId" --> U
    UA -- "AssistantUserID" --> U
    UD -- "UserId" --> U
    UD -- "DirectorId" --> U
    UD -- "OrgUnitId" --> OSU
    OSU -- "ParentId" --> OSU
    OSU -- "LinkedGroupId" --> G` Полная версия с колонками и схемы связей других подсистем БД — в system/db-er-diagrams.md.
