# Синхронизация с Active Directory (AD Sync) — справочник

Источник: https://help.1forma.ru/domains/auth/ad-sync-technical-reference/

Синхронизация пользователей и групп из Active Directory / OpenLDAP в 1Форму. Односторонняя: AD → 1Форма. Справочник описывает, как устроена синхронизация, какие таблицы и настройки задействованы, какие настройки и endpoint'ы доступны администратору и как диагностировать проблемы.
 Сервис подключения к Active Directory создаётся и настраивается в администрировании: указываются домен, учётная запись для доступа и признак «Домен является корнем леса».

Все компоненты синхронизации настраиваются в администрировании, раздел Подключения:
 Что настроить Где (Администрирование → Подключения) Форма Подключение к каталогу (домен, учётная запись) Сервисы → создать сервис типа Active Directory или OpenLDAP `active-directory-service-settings` Профиль синхронизации (привязка к сервису, фильтр OU) Синхронизации `synchronizations` Флаги и параметры синхронизации (что синхронизировать, лимит, LDAP-фильтры) Синхронизации → настройки профиля `synchronization-settings` Маппинг свойств AD → поля 1Ф Синхронизации → вкладки «Общие настройки» и «Расширенные настройки пользователя» — Вход пользователей по учётным записям AD Провайдеры → провайдер типа Active Directory `authentication-providers` Порядок первичной настройки:

- Создать сервис каталога (Подключения → Сервисы): тип Active Directory, домен, учётная запись для доступа.

- Создать профиль синхронизации (Подключения → Синхронизации) и привязать его к сервису.

- В настройках профиля включить нужные флаги (раздел 4), при необходимости задать фильтр OU, LDAP-фильтры и лимит обновлений.

- Настроить маппинг свойств (раздел 5): сопоставить атрибуты AD полям пользователя 1Ф.

- Запустить синхронизацию — вручную (кнопка в профиле или API из раздела 6) либо дождаться планового запуска задания `ADSyncJob` (ежедневно в 20:00).

Синхронизация может запускаться по расписанию, при логине, через смарт-действие или API; связь объектов AD и 1Ф ведётся по идентификатору SID.

Синхронизация может запускаться из нескольких точек:
 Триггер Что синхронизирует По расписанию (ежедневно 20:00), задание `ADSyncJob` Все активные профили: пользователи + группы + членство + вложенность При логине пользователя Один пользователь (по запросу), может создать нового Смарт-действие синхронизации Один пользователь (по UserID), может привязать SID API (admin) — синхронизация пользователя Один пользователь API (admin) — синхронизация групп Все группы Связь между базой 1Ф и AD выполняется через SID: `Users.SID` = `objectSid` пользователя, `Groups.ADSID` = `objectSid` группы.
 В синхронизацию попадают:

- Пользователи — с непустым `SID` и не уволенные (`IsFired_2 = false`).

- Группы — с непустым `ADSID` и принадлежащие текущему клиенту (`CustomerID`).

Плановая синхронизация выполняется заданием `ADSyncJob` со следующими параметрами:

- Расписание: ежедневно в 20:00.

- Параллельный запуск запрещён (защита через OS Mutex `Global\ADSyncJob`) — ручной и автоматический синк не пересекаются.

- Таймаут транзакции: 300 минут.

- Контекст выполнения: системный робот.

Шаги выполняются в следующем порядке; каждый управляется своим флагом профиля (`SynchronizationProfilesADSettings`):
 # Шаг Флаг (SyncSettings) 1 Создание отсутствующих групп `SyncGroupCreation && SyncExistingUsers` 2 Создание новых пользователей из AD `SyncUserDataBySchedule` 3 Синхронизация данных групп (SID, DisplayName, Domain) `SyncGroupData && SyncExistingUsers` 4 Синхронизация данных пользователей (по маппингу) `SyncUserData && SyncExistingUsers` 5 Синхронизация вложенности групп (`GroupParents`) `SyncGroupNesting && SyncExistingUsers` 6 Синхронизация членства user-group `SyncUserGroupMembership && SyncExistingUsers` 7 Нормализация членства (хранимая процедура) `SyncUserGroupMembership && SyncExistingUsers` 8 Проверка лимита строк `MaximumADSyncRows` 9 Перезагрузка кэша пользователей всегда 10 Смарт-события для созданных пользователей для новых пользователей 11 Смарт-события для обновлённых пользователей для изменённых

Подробнее о том, какие таблицы затрагивает каждый шаг:
 Шаг Что делает Создание отсутствующих групп Читает маски из `ADGroupSyncMasks`, находит группы в AD, создаёт в `Groups` с заполненным `ADSID` Синхронизация данных групп Обновляет `ADSID`, `DisplayName`, `Domain` у существующих групп Вложенность групп Синхронизирует `GroupParents` (parent-child связи групп) Членство user-group Синхронизирует `UserGroupsActual` (прямое членство пользователь-группа) Данные пользователей Обновляет поля пользователей по маппингу `ADPropertyMapping` Создание пользователей Создаёт новых пользователей (фильтр по OU и `UserAccountControl`) Синхронизация одного пользователя Nick, SID, mapped-свойства, ext-свойства, оргструктура

Синхронизация опирается на две группы таблиц: служебные таблицы профилей и настроек и AD-поля в основных таблицах пользователей и групп.

Профили синхронизации и их параметры хранятся в связанных таблицах:
 `ServicesSettings (ServiceType=ActiveDirectory|OpenLDAP)
  ├── LDAPServicesCredentials (FK ServiceId)
  ├── OpenLDAPServicesCredentials (FK ServiceId)
  └── SynchronizationProfiles (FK ServiceId, UNIQUE)
        ├── SynchronizationProfilesADSettings (FK SynchronizationProfileId, 1:1)
        │     ├── SynchronizationProfilesADOrgUnits (FK SynchronizationProfileADSettingsId)
        │     └── ADGroupSyncMasks (FK SynchronizationProfileADSettingsId)
        └── ADPropertyMapping (FK SynchronizationProfileId)
` Таблица Назначение `ServicesSettings` Реестр внешних сервисов. `ServiceType` = ActiveDirectory, OpenLDAP, SAML, OAuth, Radius `LDAPServicesCredentials` Домен, логин, пароль (зашифрован), `IsDomainHasForestState` `OpenLDAPServicesCredentials` Аналогично для OpenLDAP `SynchronizationProfiles` Профиль: `IsActive`, `SyncType`, `ServiceId`, `CustomerId` `SynchronizationProfilesADSettings` Все флаги синхронизации (per-profile) `SynchronizationProfilesADOrgUnits` Фильтр по OU `ADPropertyMapping` Маппинг: `OfProperty` (поле 1Ф) → `AdProperty` (атрибут AD), per-profile `ADGroupSyncMasks` Шаблоны имён (wildcards) для автосоздания групп

AD-идентификаторы хранятся в основных таблицах пользователей и групп:
 Таблица Колонки Назначение `Users` `SID` (VARCHAR 200, index `IX_Users_SID`), `DomainController` Идентификатор пользователя в AD `Groups` `ADSID` (VARCHAR 8000), `EnableADSync` (computed: `ADSID IS NOT NULL AND LEN > 0`), `Domain` Идентификатор группы в AD `UserADNickHistory` `UserID`, history data История смены AD-логинов В ходе синхронизации записи создаются и обновляются в следующих таблицах:
 Операция Таблица Создание групп `Groups` Обновление групп `Groups` Вложенность `GroupParents` Членство `UserGroupsActual` Обновление пользователей `Users` Создание пользователей `Users` Ext-свойства `UserInfoExtValues`

В синхронизации задействованы хранимые процедуры (колонка PG — поддержка на PostgreSQL):
 SP Назначение PG `tc_NormalizeGroupUsersMembership` Рекурсивный CTE по `GroupParents` → нормализует `UserGroups` и `GroupUsersMembership` с учётом вложенности Нет `sp_SyncAutoManagedGroup` Синхронизация автоуправляемых групп ? `RefreshUserNames` Обновление кэша имён пользователей Да Процедура нормализует членство с учётом вложенности групп в четыре шага:

- Вставляет отсутствующие записи в `GroupUsersMembership` из `UserGroups`

- CTE `NestingMembershipProjection` — рекурсивный обход `GroupParents`, добавляет membership с учётом вложенности

- Закрывает устаревшие записи membership (`EndDate`)

- Синхронизирует `UserGroups`: удаляет не подтверждённые через вложенность, добавляет новые
  Механизм записи на MS SQL и PostgreSQL различается — см. раздел 8.

Раньше настройки синхронизации хранились в общей таблице `Settings`. Сейчас они перенесены в профили; таблица соответствия:
 Колонка По умолчанию Мигрирована в `MaximumADSyncRows` 10 `SynchronizationProfilesADSettings.MaximumADSyncRows` `ADSyncGroupCreation` true `.SyncGroupCreation` `ADSyncGroupData` true `.SyncGroupData` `ADSyncGroupNesting` true `.SyncGroupNesting` `ADSyncUserData` true `.SyncUserData` `ADSyncUserGroupMembership` true `.SyncUserGroupMembership` `ADSyncOrganizationUnits` NULL `.SynchronizationProfilesADOrgUnits` `ADSyncUserDataBySchedule` false `.SyncUserDataBySchedule` `IncludeDomainInNick` — `.IncludeDomainInNick` `DomainController` NULL `LDAPServicesCredentials.DomainController` `DomainControllerUser` NULL `LDAPServicesCredentials.DomainControllerUser` `DomainControllerPassword` NULL `LDAPServicesCredentials.DomainControllerPassword`

Поведение синхронизации задаётся флагами профиля. Настраиваются в администрировании: Подключения → Синхронизации → настройки профиля (форма `synchronization-settings`).
 Флаг Тип Описание `SyncExistingUsers` bool Главный выключатель. Все шаги (кроме 2) требуют true `SyncUserData` bool Синхронизировать данные пользователей (шаг 4) `SyncGroupData` bool Синхронизировать данные групп (шаг 3) `SyncGroupCreation` bool Создавать отсутствующие группы (шаг 1) `SyncUserGroupMembership` bool Синхронизировать членство user-group (шаги 6, 7) `SyncGroupNesting` bool Синхронизировать вложенность групп (шаг 5) `CreateUsers` bool Разрешить создание пользователей при логине `SyncUserDataBySchedule` bool Создавать новых пользователей при плановой синхронизации (шаг 2) `SyncOnlyADEntity` bool Синхронизировать только AD-сущности в оргструктуре `IncludeDomainInNick` bool Добавлять домен к нику (DOMAIN\user) `MaximumADSyncRows` int? Лимит строк (защита от массового повреждения). По умолчанию в Settings = 10 `UsersADFilter` string LDAP-фильтр пользователей (max 1000 символов) `GroupsADFilter` string LDAP-фильтр групп `ChangePasswordException` string Исключения смены пароля `EwsServiceSettingsId` int? FK на EWS-настройки (Exchange)

Маппинг определяет, какие атрибуты Active Directory переносятся в поля пользователя 1Ф. Настраивается в профиле синхронизации (Подключения → Синхронизации) на вкладках «Общие настройки» и «Расширенные настройки пользователя».

В интерфейсе настройки маппинга поля сгруппированы в блоки:
 Блок Поля 1Ф Рабочая информация Динамические типы оргединиц (`OrgUnitType{Id}`) Личные данные Nick, LastName, FirstName, MiddleName, BirthDate, DisplayName, ImgAvatar, MaidenName, EnglishDisplayName, UserText География Country, City, Room Контакты Phone, Phone2, Phone3, CellPhone, HomePhone, Fax, Email, ExternalEmail, ICQ (Telegram), Skype, LiveJournal (—), Twitter (WhatsApp), SIP, TimeZone Функционал BusinessFunctions Прочее Notes

Стандартный набор атрибутов AD, доступных для сопоставления:
 `assistant, comment, company, department, description, displayName, division,
givenName, homePhone, mail, manager, mobile, sn, telephoneNumber, otherTelephone,
title, userPrincipalName, physicalDeliveryOfficeName, co, l
` Для каждого расширенного свойства пользователя (`UserInfoExt`) с заполненным `AdProperty` значение копируется из AD в `UserInfoExtValues`. Настраивается через вкладку «Расширенные настройки пользователя» в UI синхронизации.

При импорте thumbnailPhoto (или иного binary-атрибута фото по маппингу) сохраняется в `Users.AvatarFileId` через `UserAvatarService.SyncAvatarFromAd`.
 Дедупликация по хешу. Считается SHA-256 исходных байт AD-фото (`HashHelper.ComputeSHA256Hex`), результат хранится в системном расширенном свойстве `ADSyncAvatarHash` (`UserInfoExt`/`UserInfoExtValue`). Если хеш совпадает с сохранённым И у пользователя есть `AvatarFileId` — sync ничего не делает: не перезаливает файл, не бампает `LastPersonalInfoUpdateTime`, не инвалидирует кеши файлов.
 Восстановление после ручного удаления. При отсутствии `AvatarFileId` (пользователь удалил аватар вручную) фото восстанавливается из AD даже при совпадении хеша.
 Multi-instance nuance. Текущий `AvatarFileId` берётся из БД (`UsersEntityService`), не из `UsersCache` — иначе кеш может отставать в кластере.

Новое фото из AD заливается новой версией существующего файла, а не отдельным файлом: `Users.AvatarFileId` не меняется, `LatestVersionId` увеличивается на единицу, предыдущая версия остаётся в истории файла. Ранее выданные ссылки вида `/api/files/download/{fileId}/{versionId}` продолжают отдавать своё изображение. Новый файл создаётся только если у пользователя ещё нет аватара либо `AvatarFileId` указывает на удалённый файл (`File.IsExist`).

Дедупликация врезана в ветку синхронизации, а не в `InsertPersonImageAvatar` — тот же метод используется ручной загрузкой из профиля и мобильного приложения. При ручной загрузке аватара и при его удалении сохранённый `ADSyncAvatarHash` сбрасывается, поэтому следующая синхронизация не попадает в пропуск и применяет фото из AD.
 AD — источник истины. Если у пользователя в AD есть фото, ручная замена аватара в 1Ф будет перезаписана при следующей синхронизации. Чтобы пользователи могли ставить свои аватары, снимите маппинг свойства фото в профиле синхронизации.

Сценарий Результат Фото в AD не менялось, аватар есть Пропуск: файл, версия, `LastPersonalInfoUpdateTime`, хеш не меняются Фото в AD заменено `AvatarFileId` прежний, `LatestVersionId` +1, хеш обновлён У пользователя нет аватара Создаётся новый файл, заполняются `AvatarFileId` и хеш Аватар удалён вручную Хеш сброшен, `AvatarFileId` пуст — синк восстанавливает фото из AD Аватар заменён вручную Хеш сброшен — синк создаёт новую версию и перезатирает ручное фото из AD Фото в AD отсутствует `SyncAvatarFromAd` не вызывается, текущий аватар остаётся

Администрирование синхронизации доступно через REST API. Ниже — основные группы endpoint'ов.

Управление настройками профиля, навигацией по дереву AD и связыванием пользователей:
 HTTP Route Назначение GET `profiles/{id}/settings` Получить настройки профиля POST `profiles/{id}/settings/update` Обновить настройки профиля GET `profiles/{id}/domain/folders-tree` Дерево OU домена GET `profiles/{id}/domain/root-folder` Корневые папки домена POST `profiles/{id}/domain/folder` Загрузить папку дерева POST `profiles/{id}/domain/unload-users` Выгрузить пользователей POST `profiles/{id}/users-for-link` Пользователи для связывания POST `link-users` Связать пользователей с AD GET `ad-properties-names` Список AD-атрибутов GET `profiles/{id}/sync-properties/blocks` Блоки маппинга свойств POST `profiles/{id}/sync-properties/update` Обновить маппинг свойств GET `extra-sync-properties` Расширенные свойства синка POST `extra-sync-properties/update` Обновить расширенное свойство

Поиск пользователей и групп непосредственно в каталоге LDAP:
 HTTP Route Назначение GET `providers` Активные профили AD-синхронизации POST `search-users` Поиск пользователя в LDAP POST `search-groups` Поиск групп в LDAP Точечная синхронизация и управление настройками сервиса:
 HTTP Route Назначение POST `/api/admin/users/{userId}/sync-with-ad` Синхронизировать одного пользователя GET `/api/admin/groups/domains` Список доменов групп POST `/api/admin/groups` (sync all) Синхронизировать все группы CRUD `/api/admin/services-settings/{id}` Управление настройками сервиса (GET/POST/PUT/DELETE) Операции синхронизации также доступны через MCP-серверы:
 Сервер Тулов Доступ `mcp-admin-api-ldap` 3 admin `mcp-user-api-v2-sync-ad` 13 user `mcp-admin-api-services-settings` 6 admin Смарт-действие синхронизации: `action_sync_user_with_a_d` — параметры: `param_0` (UserID), `param_1` (SynchronizationProfileADDto, optional).

Дополнительные параметры синхронизации задаются в конфигурации приложения:
 Источник Ключ Описание `Configuration` `UsePostgreSQLDatabase` Переключатель MSSQL/PG `Configuration` `ExcludeAdSubdomains` Исключённые поддомены при навигации по дереву AD `Configuration` `SystemRobotId` ID робота для контекста выполнения ADSyncJob `web.config` `ActiveDirectoryAuthenticationMode` Режим аутентификации: `DirectoryServices` (по умолчанию) или `PrincipalContext` (для одноимённых учёток в лесе) SettingsCustom `LDAP_AdGlobalCatalogHosts` Оптимизация: список GC-хостов для ускорения загрузки дерева AD



На PostgreSQL часть операций синхронизации работает иначе, чем на MS SQL:
 Область MSSQL PG Нормализация членства (`tc_NormalizeGroupUsersMembership`) Выполняется (шаг 7) Не выполняется Создание новых групп из AD Работает Ограничено Финальное сохранение изменений Да Нет Последствия отсутствия `tc_NormalizeGroupUsersMembership` на PG:

- Вложенность групп не нормализуется после синхронизации

- `UserGroups` не получает записи, основанные на рекурсивном обходе `GroupParents`

- `GroupUsersMembership` не ведёт историю

- Создание новых групп из AD на PG не работает

Схема показывает, каким таблицам и колонкам базы 1Ф соответствуют объекты и атрибуты Active Directory:
 `Active Directory (LDAP)              1Forma DB
=======================              =========

Users                                Users
  objectSid         ──────────────►    SID
  sAMAccountName    ──────────────►    Nick
  givenName         ──────────────►    FirstName
  sn                ──────────────►    LastName
  mail              ──────────────►    Email
  (mapped attrs)    ──────────────►    (mapped fields)
  (ext attrs)       ──────────────►    UserInfoExtValues

Groups                               Groups
  objectSid         ──────────────►    ADSID
  displayName       ──────────────►    Descr
  (domain)          ──────────────►    Domain

Membership
  group.member      ──────────────►    UserGroupsActual (прямое)
                    ──[SP]────────►    UserGroups (развёрнутое с учётом вложенности)
                    ──[SP]────────►    GroupUsersMembership (история)

Nesting
  group.memberOf    ──────────────►    GroupParents (parent-child)
`

Запросы ниже помогают быстро оценить состояние синхронизации; выполняются в базе данных 1Ф:
 `-- Активные профили синхронизации
SELECT sp.Id, ss.Description, ss.ServiceType,
       ads.SyncUserData, ads.SyncGroupData, ads.SyncGroupCreation,
       ads.SyncUserGroupMembership, ads.SyncGroupNesting,
       ads.MaximumADSyncRows
FROM SynchronizationProfiles sp
JOIN ServicesSettings ss ON sp.ServiceId = ss.Id
JOIN SynchronizationProfilesADSettings ads ON ads.SynchronizationProfileId = sp.Id
WHERE sp.IsActive = 1;

-- Пользователи с SID (привязаны к AD)
SELECT COUNT(*) AS TotalLinked FROM Users WHERE SID IS NOT NULL AND LEN(LTRIM(RTRIM(SID))) > 0;

-- Группы с AD sync
SELECT GroupID, Descr, ADSID, Domain, EnableADSync
FROM Groups WHERE EnableADSync = 1;

-- Маски для создания групп
SELECT m.Wildcard, ads.SynchronizationProfileId
FROM ADGroupSyncMasks m
JOIN SynchronizationProfilesADSettings ads ON m.SynchronizationProfileADSettingsId = ads.Id;

-- Маппинг свойств
SELECT OfProperty, AdProperty, Entity
FROM ADPropertyMapping WHERE SynchronizationProfileId = @profileId;

-- Последний запуск ADSyncJob (Quartz)
SELECT TOP 1 * FROM QRTZ_FIRED_TRIGGERS WHERE JOB_NAME LIKE '%ADSync%' ORDER BY FIRED_TIME DESC;
`

Наиболее частые проблемы синхронизации и способы их диагностики:
 Симптом Причина Диагностика Группы не создаются из AD PG: SP отсутствует; MSSQL: маска не задана или `SyncGroupCreation = false` Проверить флаги + маски + `UsePostgreSQLDatabase` Пользователи не синхронизируются `SyncExistingUsers = false` (главный выключатель) `SELECT SyncExistingUsers FROM SynchronizationProfilesADSettings` Тихий сбой (нет ошибок и нет изменений) `MaximumADSyncRows` = 10 (по умолчанию), строк больше лимита → откат Проверить значение лимита, увеличить Дубликаты SID → ошибка В `Users` две записи с одинаковым SID `SELECT SID, COUNT(*) FROM Users GROUP BY SID HAVING COUNT(*) > 1` Ошибка обращения к глобальному каталогу AD Некорректная настройка `LDAP_AdGlobalCatalogHosts` Проверить список GC-хостов; если не помогает — обратиться в поддержку 1Ф Фильтр OU не работает Известный баг: при включённом фильтре OU пользователи не создаются Обратиться в поддержку 1Ф

При планировании и настройке синхронизации учитывайте следующие ограничения:

- PG: нет `tc_NormalizeGroupUsersMembership` — вложенность групп не нормализуется автоматически. Создание новых групп из AD на PG не работает.

- OS Mutex `Global\ADSyncJob` — защита от параллельного ручного и автоматического запуска.

- Транзакция 300 минут — при большом количестве пользователей может не хватить.

- MaximumADSyncRows по умолчанию = 10 — очень мало для реальных площадок, часто вызывает тихий откат без ошибок.

- SID-дубликаты — при дубликатах SID в БД синхронизация завершается ошибкой.

- Динамические группы AD не поддерживаются.

- Символы `#` и `&` в логинах/паролях — запрещены, вызывают ошибку при синхронизации.

- `ActiveDirectoryAuthenticationMode` — при одноимённых учётках в лесе AD нужен `PrincipalContext`, иначе ошибки аутентификации.

-  Инвалидация токенов при смене пароля — при смене пароля в AD фоновый процесс проверки провайдеров аутентификации (каждые 15 минут) автоматически инвалидирует ранее выданные токены.


-  `ObjectDisposedException` при чтении отдельных AD-атрибутов на Windows — устранено на уровне платформы; см. support-guide-ad-sso.md, п. 1.5.
