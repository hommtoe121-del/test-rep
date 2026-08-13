# Оргструктура — синхронизация

Источник: https://help.1forma.ru/domains/org-structure/sync/

Оргструктуру можно наполнять и поддерживать несколькими способами: синхронизацией с Active Directory / LDAP, синхронизацией с 1С, импортом из Excel и внешних источников или вручную. Эта страница описывает интеграции; детальная техническая спецификация — Справочник AD Sync.



Данные из Active Directory попадают в оргструктуру 1Ф через профиль синхронизации:
 `AD Server (LDAP)
  └─ OU (Organizational Units)
       └─ Users, Groups
            ↓ задание ADSyncJob / API синхронизации
SynchronizationProfiles
  └─ SynchronizationProfilesADSettings
       ├─ SyncOrganizationUnits (устаревшее поле)
       └─ SynchronizationProfilesADOrgUnits (фильтры OU)
            ↓
OrgStructureUnit ← UserOrgStructureUnit ← Users
` В синхронизации оргструктуры с AD задействованы следующие таблицы:
 Таблица Назначение `SynchronizationProfiles` Профиль синхронизации (Id, SyncType, ServiceId, IsActive) `SynchronizationProfilesADSettings` Флаги: SyncUserData, SyncGroupData, MaximumADSyncRows, LDAP-фильтры (поле `SyncOrganizationUnits` — устаревшее, см. ниже) `SynchronizationProfilesADOrgUnits` Фильтр OU: OrgUnitPath, OrgUnitName, IncludeNested `LDAPServicesCredentials` Учётные данные подключения к домену `ADPropertyMapping` Маппинг атрибутов AD → поля 1Формы

Синхронизация оргструктуры с AD идёт по такой логике:

- Профиль определяет домен и настройки: какие данные синхронизировать (пользователей, группы, оргструктуру).

- Фильтр OU (`SynchronizationProfilesADOrgUnits`) определяет, из каких Organizational Units AD тянуть данные. `IncludeNested` — включать вложенные OU.

- Маппинг свойств — блок «Рабочая информация» в ADPropertyMapping содержит динамические поля `OrgUnitType{Id}`, связывающие атрибуты AD (department, title, division) с типами оргструктуры 1Формы.

- При синхронизации пользователя обновляются его привязки к OrgStructureUnit через UserOrgStructureUnit.
  При включённом `SyncOnlyADEntity = true` — синхронизируются только те элементы оргструктуры, которые существуют в AD. Элементы, созданные вручную в 1Форме, не затрагиваются.
 ⚠️ Поле `SyncOrganizationUnits` в `SynchronizationProfilesADSettings` — устаревшее: исторически хранило список OU (`nvarchar`), сейчас фильтрация OU выполняется таблицей `SynchronizationProfilesADOrgUnits`, а само поле должно быть пустым (`null`). Непустое legacy-значение ломало поиск SID пользователя в AD на PostgreSQL; с 2.268 значение очищается миграцией.

Управление AD-синхронизацией доступно через REST API:
 Route Назначение `GET/POST /api/sync-ad/profiles/{profileId}/settings` Настройки профиля `GET /api/sync-ad/profiles/{profileId}/domain/folders-tree` Дерево OU домена `POST /api/sync-ad/profiles/{profileId}/domain/folder` Содержимое конкретной OU `POST /api/sync-ad/profiles/{profileId}/domain/unload-users` Выгрузка пользователей из AD `POST /api/sync-ad/link-users` Привязка AD-пользователей по SID `GET /api/sync-ad/profiles/{profileId}/sync-properties/blocks` Блоки маппинга свойств `POST /api/sync-ad/sync-properties/update` Обновление маппинга При настройке синхронизации с AD учитывайте ограничения:

- Иерархия AD должна быть ясной; тип AD-объекта сопоставляется 1:1 с типом `OrgStructureType`

- Должность (`IsPosition`) обязательна в обеих системах

- На PostgreSQL отсутствует хранимая процедура `tc_NormalizeGroupUsersMembership` (только для MS SQL)

Базовый путь API: `/api/admin/org-structure/1c-sync/...`
 Синхронизация с 1С настраивается и запускается так:

- Создаётся XML-конфигурация маппинга полей 1С ↔ 1Форма

- Шаблон конфигурации: `GET /api/admin/org-structure/1c-sync/configs/default`

- Сохранение конфигурации: `POST /api/admin/org-structure/1c-sync/configs`

- Запуск синхронизации: `POST /api/admin/org-structure/1c-sync/sync`
  Связь единиц 1Ф с объектами 1С хранится в полях:
 Поле Назначение `GuidFrom1C` GUID объекта в 1С `ParentGuidFrom1C` GUID родителя в 1С При синхронизации сопоставление идёт по GuidFrom1C.

Загрузка пользователей из файла Excel. При загрузке могут создаваться оргструктурные единицы.
 Через механизм импорта данных оргструктуру можно наполнять:

- Ручной импорт или автоматический по расписанию

- Внешний источник → OrgStructureUnit (через ExternalID / ParentExternalID)
  Связь единиц 1Ф с внешней системой хранится в полях:
 Поле Назначение `ExternalID` ID из внешней системы `ParentExternalID` ID родителя из внешней системы `IsImported` (UserOrgStructureUnit) Признак: привязка пользователя создана импортом Оргструктуру можно создать и вручную — через интерфейс администратора или Admin API:

- Создание дерева единиц сверху вниз

- Назначение типов, должностей

- Привязка пользователей

- Связывание с группами
  Последовательность: типы → дерево → должности → пользователи → группы.
