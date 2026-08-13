# Авторизация и вход

Источник: https://help.1forma.ru/domains/auth/admin/

Документ описывает администрирование домена auth: провайдеры и сервисы аутентификации, политики входа, синхронизацию каталогов, настройку типов входа и регистрации, а также справочник ключей appsettings.json. Для администраторов и инженеров поддержки.

Администрирование `auth` построено вокруг связки "сервис аутентификации -> провайдер аутентификации -> политики входа". Основной UI-контур расположен в разделе `system -> подключения`.
 В домене используются все три уровня:

- формы `dbadmin` (провайдеры и сервисы);

- `EntityEditor` для провайдеров;

- служебные Admin API для проверки LDAP и получения конфигураций.
  Настройки домена доступны через формы автоадминки, EntityEditor и служебные API.
 Формы автоадминки для провайдеров, сервисов и профилей синхронизации:
 Alias формы Название Таблица БД Что настраивается `authentication-providers` Провайдеры аутентификации `dbo.AuthenticationProviders` Тип провайдера, активность, 2FA, видимость `authentication-provider-groups` Доступ к провайдеру по группам `dbo.AuthenticationProviderGroups` Ограничения входа по группам `authentication-provider-domains` Домены провайдера аутентификации `dbo.AuthenticationProviderDomains` Ограничение отображения провайдера по доменам (FQDN) `active-directory-service-settings` Сервис Active Directory `dbo.LDAPServicesCredentials` Подключение к AD `openldap-service-settings` Сервис OpenLDAP `dbo.OpenLDAPServicesCredentials` Подключение к OpenLDAP `oauth-service-settings` Сервис OAuth `dbo.OAuthServicesSettings` OIDC/OAuth параметры `saml-service-settings` Сервис SAML `dbo.SAMLServicesSettings` IdP metadata, сертификаты, claims `radius-service-settings` Сервис Radius `dbo.RadiusServicesCredentials` Radius-сервер `multifactor-service-settings` Сервис Multifactor `dbo.MultifactorCredentials` Внешний второй фактор `synchronizations` Синхронизации (AD, LDAP) `dbo.SynchronizationProfiles` Профили sync с каталогами `synchronization-settings` Настройки синхронизации `dbo.SynchronizationProfilesADSettings` Параметры AD-синхронизации В разделе system → подключения при создании или редактировании сервиса аутентификации форма отображает поля, специфичные для выбранного типа сервиса. Обязательные поля помечены звёздочкой (*). Кнопка «Сохранить» становится активной только после заполнения всех обязательных полей текущего типа.
 Состав обязательных полей зависит от типа сервиса:
 Тип сервиса Обязательные поля Active Directory Домен, Логин для доступа к AD, Пароль для доступа к AD Radius IP Radius сервера, Порт Radius сервера, Shared secret PayControl System ID, API URL, Server Signer, User ID, Reg Accepted Ext Sys Name, Allow Update Ext Sys Name TranslateService Login, Password, Key, URL, Region OpenLDAP Server Address DSS CryptoPro Sign Server, STS App Name, Host, Client ID Для типов OAuth, SAML и некоторых других, не имеющих обязательных полей в конфигурации, кнопка «Сохранить» активна сразу после выбора типа сервиса.

Расширенное редактирование провайдеров по JSON-схеме:
 Схема JSON Таблица Назначение `authenticationProviders` `dbo.AuthenticationProviders` Расширенное редактирование провайдеров Служебные маршруты, используемые админ-интерфейсом:
 Контроллер Маршрут Методы Назначение `AuthenticationProvidersController` `/api/admin/authentication-providers` GET Список провайдеров для админ-интерфейса `LdapController` `/api/admin/ldap` GET, POST Проверка провайдеров, поиск пользователей и групп в LDAP Маршруты входа, по которым проверяют, что настройки работают:
 Контроллер Маршрут Что проверяет `AuthController` `/api/auth/token-v2`, `/api/auth/token/refresh`, `/api/auth/info` Базовый password/refresh контур `SamlAuthController` `/api/auth/saml/*` SAML SSO-поток `OAuthController` `/api/auth/oauth` OAuth/OIDC вход и выход



Где настраивается: `authentication-providers`, `authentication-provider-groups`, `authentication-provider-domains`, `auth_providers.md`
 Таблицы БД:

- `dbo.AuthenticationProviders`

- `dbo.AuthenticationProviderGroups`

- `dbo.AuthenticationProviderDomains`
  Что контролируется:

- тип провайдера (`ActiveDirectory`, `OpenLDAP`, `OAuth`, `SAML`, `Radius`);

- флаги активности/скрытия/"по умолчанию";

- второй фактор;

- доступность провайдера для конкретных групп;

- привязка провайдера к доменам (FQDN) — на странице авторизации провайдер показывается только при входе с указанных доменов. Если список доменов пуст — провайдер доступен со всех доменов. Список настроенных доменов передаётся на фронт через `app-settings.json → Providers`, фильтрация выполняется на стороне клиентского приложения по сравнению текущего `hostname` страницы с заданным списком.

Для OIDC-провайдера список `scope` в запросе к Identity Provider теперь может задаваться явно и/или вычисляться по конфигурации провайдера.
 Что важно при настройке:

- `openid` должен присутствовать всегда — это базовый scope протокола OpenID Connect;

- если в `SettingsJson` задан ключ `Scopes`, платформа использует его как явный список дополнительных scope;

- если `Scopes` не задан, платформа вычисляет дополнительные scope по списку `Claims`;

- для claim `phone_number` и `phone_number_verified` требуется scope `phone`;

- для claim `email` и `email_verified` требуется scope `email`;

- для claim группы профиля (`name`, `family_name`, `given_name`, `middle_name`, `nickname`, `preferred_username`, `profile`, `picture`, `website`, `gender`, `birthdate`, `zoneinfo`, `locale`, `updated_at`) требуется scope `profile`;

- для claim `address` требуется scope `address`.
  Это изменение нужно для сценариев, где пользователь сопоставляется не по стандартному `sub`, а по другому claim, например по номеру телефона. Ранее запрос мог уходить только с `scope=openid`, из-за чего IdP не возвращал нужный claim и вход завершался ошибкой.
 Пример настройки входа по телефону:
 `{
  "Claims": ["phone_number"],
  "Scopes": ["phone"],
  "ClaimsMapperConfig": {
    "MapperType": "ByAttribute",
    "IdentityClaim": "phone_number",
    "UserAttribute": "MobilePhone"
  }
}
` Практическое правило: если используемый для идентификации claim зависит от стандартного OIDC scope, лучше явно указать `Scopes` в `SettingsJson`, чтобы поведение не зависело от неявного вывода.
 Где настраивается: формы `*-service-settings` в `system -> подключения`.
 Таблицы БД:

- `dbo.LDAPServicesCredentials`

- `dbo.OpenLDAPServicesCredentials`

- `dbo.OAuthServicesSettings`

- `dbo.SAMLServicesSettings`

- `dbo.RadiusServicesCredentials`

- `dbo.MultifactorCredentials`
  Что контролируется: точки API (endpoint), учётные данные, сертификаты, сопоставление claims и технические параметры согласования.

Политики входа и защитные лимиты настраиваются в общих системных настройках (`system-settings`) и пользовательских ключах (`sys_user`) и определяют правила безопасности аутентификации.
 Что контролируется:

- лимиты неудачных попыток входа и captcha;

- ограничения на логин по email (ключ `ForbidEmailAsLogin`): если включён — система запрещает вход по полю Email; разрешён только вход по Логину/Нику. Используется, когда email-адреса не уникальны или не предназначены для аутентификации;

- правила регистрации и password-политики.

Три независимых счётчика ограничивают перебор паролей:
 Настройка Эффект при превышении Максимальное число попыток логина до блокировки Учётная запись блокируется. Разблокировка — только администратором Максимальное число попыток логина пользователя до капчи Запрашивается ввод captcha при следующих попытках Максимальное число попыток логина с IP до капчи IP-адрес блокируется; для разблокировки выводится captcha ⚠️ Все три счётчика работают независимо. Если значение пустое — соответствующая защита выключена.
 Работа за балансировщиком нагрузки. Если 1Форма стоит за reverse proxy / load balancer, реальные IP клиентов нужно прокидывать через заголовки `X-Real-IP` и `X-Forwarded-Proto`. В `appsettings.json` за это отвечает секция `ForwardHeaders` — без её настройки счётчик попыток «с IP» будет видеть IP балансировщика как единственный источник и блокировать всех пользователей сразу.
 Адрес страницы логина для редиректа после API-ошибок (с v2.257) задаётся ключом `AuthTokenLoginUrl` в `appsettings.json`. Используется в сценарии `~/api/...?auth=true` (см. `user-ui/admin.md`, §9.2).

Область применения: настройки учитываются при создании пользователей из режима администрирования и при сбросе пароля. Для самостоятельной регистрации действуют требования из пользовательского ключа `RegistrationFields` (см. ниже §«Самостоятельная регистрация»).
 Параметр Поведение Срок действия пароля (дни) Только forms-авторизация. Если задан — в профиле отображается счётчик дней до смены; при просрочке пользователь редиректится на форму смены при следующем входе Минимальная / Максимальная длина пароля Если не задано — не проверяется Содержит спецсимвол (`! @ # $ %` и т. п.) Флаг Содержит заглавную букву Флаг Пароль может совпадать с предыдущим Если выключено — нельзя устанавливать ранее использованный пароль. Применяется только при смене пароля существующего пользователя — у нового пользователя истории паролей нет, поэтому при создании (вручную в админке или через AD-синк с автогенерацией пароля) проверка не выполняется Пароль может совпадать с логином Если включено — допускается пароль, идентичный логину Длина истории паролей Сколько предыдущих паролей запоминается и блокируется к повторному использованию Проверять пароль на наличие даты рождения Запрет использовать дату рождения пользователя Минимальная длина последовательности символов для проверки Распознаются как небезопасные: подряд идущие цифры (`123`, `7890`), буквы по алфавиту (`abc`, `mno`), соседи на клавиатуре (рус.: `йцукен…`, `фывапрол…`, `ячсмить…`; англ.: `qwerty…`, `asdfgh…`, `zxcvbn…`) Минимальная длина повторяющегося паттерна для проверки Запрет паттернов вида `qweqwe`, `123123`. Срабатывает только при непосредственном повторе (не при разрозненном употреблении)

Для встроенных учётных записей платформа использует алгоритм хеширования паролей Argon2id (RFC 9106) как актуальный стандарт хранения паролей.
 Текущие параметры хеширования:

- память — `19 MiB` (`m=19456`);

- итерации — `3`;

- parallelism — `1`;

- длина хеша — `32 байта`;

- длина salt — `16 байт`.
  Хеш хранится в поле `PasswordHashPhc` в формате PHC string: `$argon2id$v=19$m=19456,t=3,p=1$<salt-base64>$<hash-base64>`
 Переход со старого формата выполнен без принудительного сброса паролей. При входе пользователя система сначала проверяет актуальный формат Argon2id. Если пароль был сохранён в старом формате Argon2i, выполняется резервная проверка по старому хешу, после чего при успешной аутентификации пароль автоматически перехешируется в формат Argon2id.
 Это поведение относится только к встроенным учётным записям 1Формы. Для пользователей Active Directory, LDAP, SAML и OIDC пароль в БД 1Формы не хранится, поэтому поле `PasswordHashPhc` для них может оставаться пустым.
 Параметр «Срок действия кода для восстановления пароля в днях» задаёт TTL email-кода, высылаемого по запросу «Забыли пароль?».

Реализовано (релиз 2.266)
 Где настраивается:

- `appsettings.json` → секция `Auth` (серверные лимиты);

- спецправо `GENERATEPAT` на группу (управление через `system → подключения → спецправа` или dbadmin-форму `groups_spec`).
  Параметры `appsettings.json` → `Auth`:
 Ключ Тип Дефолт Описание `PatMaxTokensPerUser` int 10 Максимум не-отозванных токенов на одного пользователя `PatMaxExpirationDays` int 365 Максимальный TTL токена в днях; `0` = бессрочные разрешены Таблица БД: `dbo.PersonalAccessTokens`
 Admin API: `/api/admin/pat/*` — CRUD для администраторов (генерация, просмотр, отзыв токенов любого пользователя).
 User API: `/api/user/pat/*` — генерация (только при наличии спецправа `GENERATEPAT`), просмотр и отзыв своих токенов.
 DevOps-задача: при деплое/обновлении — убедиться, что ключи `PatMaxTokensPerUser` и `PatMaxExpirationDays` добавлены в `appsettings.json`.

Серверные ключи секции `Auth` (и связанных) в `appsettings.json`:
 Ключ По умолчанию Назначение `AuthTokenLoginUrl` — URL формы логина для редиректа из API при ошибке 401 (используется в сценарии `~/spa/entry/signin?fromUrl=...`, см. `user-ui/admin.md` §9.2). С v2.257+ `AuthTokenExpiresInMinutes` `1500` (25 ч) TTL access-токена. ⚠️ В SPA по истечении нужна повторная авторизация `AuthRefreshTokenExpiresInMinutes` `43200` (30 дней) TTL refresh-токена `AuthTokenRefreshStrategy` `SlidingExpiration` Стратегия обновления: `SlidingExpiration` — авто-обновление при активности, `RefreshToken` — через `api/auth/token/refresh`, `None` — без обновлений `AuthTokenRefreshPeriodInMinutes` `20% от AuthTokenExpiresInMinutes` Период автообновления при `SlidingExpiration`. Действует только при этой стратегии `AuthTempTokenExpiresInMinutes` — TTL временного access-токена с ограниченными правами (выдаётся при смене пароля) `AuthBasicAllowedPaths` — Регексы путей с разрешённой Basic-аутентификацией (через `;` или `,`). ⚠️ Экранируйте спецсимволы regex `AuthLoginCodeExpiresInMinutes` `15` TTL кода регистрации/аутентификации, в минутах `ConcurrentSessionMinLengthInMinutes` `1` Минимальная длительность активной сессии для конкурентной лицензии (через кэш `ConcurrentSessionsExpirationCache`). Меньше — быстрее перераспределение лицензий, больше — медленнее освобождение при закрытой вкладке `LDAPSearchBase` — База поиска для LDAP `UseSecureLDAP` `true` SSL для LDAP-подключений, включая провайдер OpenLDAP. При включении подключение идёт на порт 636 (LDAPS) `OidcAliases` — Алиасы OpenID-провайдеров (через запятую): `"sm,forma"` `AlignedAuthResponseTimeMs` `700` Минимальное время ответа аутентификации (для защиты от time-attacks). При неуспехе — задержка до этого порога. Успешные ответы (200) — без задержки. `0` или отрицательное — отключение (для медленного AD) ### 6.1. appsettings.json — PAT, Windows-аутентификация, Multifactor, cookies и SQL-диагностика В этом разделе описаны параметры `appsettings.json`, управляющие PAT-токенами, Windows-аутентификацией, двухфакторной аутентификацией через Multifactor, поведением cookie и SQL-диагностикой.
 | `PatMaxTokensPerUser` | `10` | Максимум активных PAT-токенов на пользователя | | `PatMaxExpirationDays` | `365` | Макс TTL PAT в днях. `0` — разрешает бессрочные | | `WinAuthHost` | `IIS` | Тип хостинга Windows-аутентификации. `IIS` — Integrated Windows Auth через IIS-модуль (Windows host). `Kestrel` — ASP.NET Core Negotiate scheme (Linux/Docker host, требует Kerberos keytab/SPN со стороны OS). `None` — Windows-auth выключена | | `Negotiate:RequireKerberos` | `false` | При `true` принимается только Kerberos; NTLM-аутентификация отклоняется (`WinClaimsTransformation` на Windows, `NegotiateClaimsTransformation` на Linux). Нужно когда NTLM запрещён политикой безопасности. Полный путь: `Auth:Negotiate:RequireKerberos`. Доступно с v2.268.302 | | `Multifactor.IsEnabled` | `true` | Двухфакторная через сервис MULTIFACTOR | | `Multifactor.Host` | — | Адрес сервиса Multifactor | | `ActiveDirectoryAuthenticationMode` | `ldap` | Библиотека для AD-аутентификации: `DirectoryServices` (System.DirectoryServices, только AD), `PrincipalContext` (более высокоуровневая, нужна при одноимённых учётных записях в лесе AD), `ldap` (Novell.Directory.Ldap — работает и с AD, и с OpenLDAP) | | `AuthUseInsecureCookies` | bool / `false` | Разрешает не-secure cookie для аутентификации. ⚠️ Включать только при работе по HTTP. Для HTTPS — снижает безопасность | | `SetCookieForUpperLevelDomain` | bool / `false` | Cookie `1FormaAuth` ставится на домен верхнего уровня (нужно при нескольких поддоменах, например `forms.example.ru` + `win.example.ru`). По умолчанию — на полный FQDN |
 Диагностика:
 `-- Все активные PAT пользователя
select
        pat.Id,
        pat.Name,
        pat.TokenPrefix,
        pat.CreatedAt,
        pat.ExpiresAt,
        pat.LastUsedAt,
        pat.IsRevoked
from PersonalAccessTokens pat with (nolock)
where pat.UserId = @UserId
      and pat.IsRevoked = 0
order by pat.CreatedAt desc;
`

Если пользователь, успешно прошедший аутентификацию в Keycloak, не имеет лицензии на модуль «Первая форма», или используется конкурентная лицензия `FirstFormaConcurrent` и лимит сессий исчерпан, вход в 1Форму через OIDC блокируется на этапе создания токена.
 Что происходит:
 Ситуация Результат Нет именной лицензии `FirstForma` Редирект на `/spa/entry/signin?authError=Нет лицензии для модуля Первая форма`. Токен не выдаётся. Исчерпан лимит `FirstFormaConcurrent` Редирект на `/spa/entry/signin?authError=…` с сообщением о превышении лимита сессий. Лицензия в порядке Обычный вход, сессия и токен создаются. Текст сообщения берётся из локали (`Language.doesNotHaveLicenceShort`) и отображаемого имени модуля `Module.FirstForma.GetDisplayName()` — аналогично парольному входу.
 Примечание для администратора. Сообщение появляется на странице входа 1Формы, а не в Keycloak. Если пользователь видит страницу входа 1Формы с ошибкой лицензии после успешного ввода учётных данных в Keycloak, проверьте лицензионные настройки пользователя в 1Форме (именная или конкурентная лицензия «Первая форма»).

Контент в `business.md` описывает концепции и параметры протоколов. Ниже — UI-сценарии создания сервисов и провайдеров.

Настройка любого внешнего способа входа идёт в три шага:

- Создать сервис (`system → подключения → Сервисы`) — выбрать тип, заполнить параметры подключения

- Создать провайдер аутентификации (`authentication-providers`) — выбрать сервис, настроить MFA, группы, видимость

- Настроить AuthConfig (пользовательский ключ) — определить доступные типы входа
  Сервис (`openldap-service-settings`):
 Поле Описание Адрес сервера URL LDAP-сервера DN путь Базовый DN для поиска DN привязки DN учётной записи для bind Пароль привязки Пароль для bind SSL-подключение: ключ `UseSecureLDAP` в `web.config`/`appsettings.json`. Флаг действует на обе точки подключения OpenLDAP — проверку пароля пользователя и служебный bind под учётной записью администратора каталога. При выключенном флаге оба пароля передаются по сети открытым текстом.
 Параметры: business.md#openldap
 Сервис (`radius-service-settings`):
 Поле Описание IP-адрес Адрес RADIUS-сервера Порт Порт (обычно 1812) Shared secret Общий секрет Параметры: business.md#radius
 Сервис (`multifactor-service-settings`):

- В админ-панели Multifactor (`admin.multifactor.ru`): Настройки → Расширенное API → скопировать API Key и API Secret

- В 1Ф: создать сервис типа Multifactor, указать API URL (`https://api.multifactor.ru`), API-ключ и Секретный ключ

- Установить сертификат `<corp-domain>.cer` на сервер

- В провайдере аутентификации: Второй фактор → Сервис → выбрать сервис Multifactor
  appsettings.json: секция `Multifactor` с ключами `IsEnabled` и `Host`. web.config: `MultifactorIsEnabled` и `MultifactorHost`.
 Процесс и параметры: business.md#multifactor-сервис

Сервис (`active-directory-service-settings`):
 Поле Описание Домен Имя домена компании Домен является корнем леса Работа с корнем леса AD Логин/Пароль для доступа к AD Опционально — если пользователь приложения уже имеет нужные права ⚠️ Режимы аутентификации: по умолчанию используется `DirectoryServices`. При наличии одноимённых учётных записей в лесу AD переключить на `PrincipalContext` через настройку `ActiveDirectoryAuthenticationMode` в `web.config`/`appsettings.json`. Без переключения система может авторизовать не того пользователя.
 Провайдер: тип `ActiveDirectory`, выбрать созданный сервис, настроить второй фактор при необходимости.
 Параметры и бизнес-логика: business.md#active-directory

Сервис (`saml-service-settings`):
 Поле Описание IDP Metadata URL URL метаданных IdP, например `https://your-domain.com/FederationMetadata/2007-06/FederationMetadata.xml` Сопоставление пользователей На данный момент — только по SID из Active Directory Settings (JSON):
 Ключ Описание `Issuer` Издатель запроса. ⚠️ Для ADFS должен совпадать с доменом SP-приложения (relying party trust в терминах ADFS) `SignatureAlgorithm` Алгоритм подписи (по умолчанию: `rsa-sha256`) `UserClaimName` Claim для идентификации пользователя. Для ADFS: `primarySid` `SignAuthRequests` Подписывать ли запросы к IDP. ⚠️ Для ADFS обязательно `true` `SignCertificatePath` Путь к `.pfx`-сертификату для подписания SP→IDP `SignCertificatePassword` Пароль к `.pfx`-файлу ClaimsMapperConfig:
 Ключ Описание `IdentityClaim` Атрибут для однозначной идентификации: `Email`, `Nick`, `ExternalAccount`, `SID` `CreateProfiles` `true` → автосоздание профиля из SAML-атрибутов, если УЗ не найдена по IdentityClaim `ProfileAttributesMap` Маппинг полей профиля (см. ниже) ProfileAttributesMap — 29 полей: `Nick`, `LastName`, `FirstName`, `MiddleName`, `FullName`, `Phone`, `Phone2`, `Phone3`, `Email`, `ExternalEmail`, `DisplayName`, `Position`, `EnglishDisplayName`, `ExternalDisplayName`, `CellPhone`, `HomePhone`, `Fax`, `Skype`, `ICQ`, `LiveJournal`, `Twitter`, `IsEmployee`, `BirthDate` (datetime), `WorkStartDate` (datetime), `Country`, `City`, `Gender` (bool), `Notes`, `SID`, `GuidFrom1C` (guid), `PhoneAdditional`, `Phone2Additional`, `Phone3Additional`, `HomePhoneAdditional`, `MaidenName`, `CanEditAvatar` (bool), `TelegramUserName`
 Если для SAML включена подпись запросов, но сертификат подписи не задан или не читается (пустой путь, файла нет, неверный пароль), либо некорректны метаданные IdP, вход через SAML не работает — провайдер сообщает, что интеграция не настроена. Проверьте путь к файлу сертификата и пароль, права на чтение файла и адрес метаданных IdP.
 ⚠️ `CreateProfiles: true` = автопровизионинг пользователей без предварительного создания в 1Ф. Без понимания ограничения `Issuer` — SSO с ADFS не заработает.
 appsettings.json: `SamlEntityId` (доменное имя SP), `SamlCertificatesRoot` (путь к сертификату).
 Маршруты: `/api/auth/saml/{providerId}/login`, `/api/auth/saml/{providerId}/assertionconsumer`, `/api/auth/saml/{providerId}/logout`, `/api/auth/saml/{providerId}/singlelogout`.
 Метаданные SP: `/api/auth/saml/{providerId}/metadata`.
 Концепции и SSO-поток: business.md#saml

Сервис (`oauth-service-settings`):
 Поле Описание Alias Алиас провайдера `[a-z]` Реализация OAuth OpenIDConnect ClientId ID приложения KeyCloak (Clients → компания → General → Client ID) ClientSecret Секретный ключ (Clients → компания → Credentials → Client secret) Сопоставление пользователей Один из 4 типов (см. ниже) Settings (JSON):
 Ключ Описание `AuthorityUrl` URL центра OAuth `ResponseType` Тип ответа AtomID `ResponseMode` Режим ответа `CallbackPath` URL возврата после аутентификации `Claims` Список claims `ClaimsMapperConfig` Настройки маппинга (см. ниже) 5 типов ClaimsMapper:
 Класс Описание `ClaimsMapperByDictionary` По справочнику (категории) `UserClaimsMapperByAttribute` По произвольному атрибуту `UserClaimsMapperByExternalAccounts` По УЗ внешнего сервиса (UserExternalAccounts) `UserClaimsMapperByNickname` / по SID По Nickname или SID `UserClaimsMapperByUpn` По UPN из Windows-claim. Используется по умолчанию в `NegotiateSettings.UserMapper` для Kerberos/Negotiate-входа; `ClaimsMapperConfig.UserProfileAttribute` задаёт поле пользователя для маппинга (по умолчанию `Nick`) ClaimsMapperConfig:
 Ключ Описание `IdentityClaim` `Email`, `Nick`, `ExternalAccount`, `SID` `DictionarySubcatId` ID категории справочника (для ClaimsMapperByDictionary) `DictionaryIdentityExtParamId` ID ДП для идентификации `DictionaryUserIdExtParamId` ID ДП для UserID После настройки:

- Добавить провайдер аутентификации с созданным сервисом

- Прописать алиас в ключе `OidcAliases` в `appsettings.json` (несколько провайдеров — через запятую)
  Логирование: вход через OIDC логируется в `LoginsLog` аналогично Forms-аутентификации (сброс счётчика попыток, добавление IP в белый список). Выход не логируется.
 Концепции и способы сопоставления: business.md#oauth-20--openid-connect

Пользовательский ключ `AuthConfig` управляет доступными способами входа на форме авторизации.
 По умолчанию: `{"AuthTypes": []}`
 Параметры каждого элемента AuthTypes:
 Параметр Описание `Type` `login-pass`, `phone-code`, `email-code`, `external-provider` `IsDefault` Тип входа по умолчанию. При нескольких `true` — появляется кнопка переключения `AllowRegister` Показать кнопку регистрации `AutoRegister` Автоматическая регистрация `Visibility` `all`, `web`, `mobile` — ограничение по устройству `PrivacyLink` Ссылка на соглашение при входе `RegisterPrivacyLink` Массив ссылок на соглашение при регистрации (`linkUrl` + `linkTitle`) `RegistrationType` `all`, `phone`, `email` `HideProviders` Скрыть выбор провайдера ⚠️ Параметры в AuthTypes обязательно с заглавной буквы.
 Как работает `Visibility`. Значение ограничивает способ входа по типу устройства: `mobile` — только мобильные (Android/iOS), `web` — только компьютеры, `all` (или не указано) — все устройства. Устройство определяется по фактической платформе браузера (Android/iOS), а не по режиму отображения страницы: способ с `Visibility: "mobile"` виден на реальном мобильном устройстве независимо от того, включён ли режим mSPA.
 Вход по email (`email-code`). Пользователь вводит адрес почты, указанный в его профиле, и нажимает «Далее»; на этот адрес приходит 4-значный код, ввод которого завершает вход. Для работы способа в профиле пользователя должен быть добавлен почтовый ящик, а в общих настройках приложения — выбран почтовый ящик для системных писем и настроен SMTP-сервер.

 Вход по номеру телефона (`phone-code`). Пользователь вводит номер телефона и нажимает «Далее»; на него приходит 4-значный код, ввод которого завершает вход. Требуется настроенный SMS-провайдер.

 Поведение `external-provider`: отображается экран только с кнопками внешних провайдеров (без формы логина/пароля и капчи). Внизу — ссылка «Войти по логину и паролю» с кнопкой «Назад». ⚠️ Если в AuthTypes нет `login-pass` — ссылка не отображается.
 Приоритет: AuthConfig > `sys_general_settings` — если регистрация отключена в ключе, глобальная настройка «Разрешена регистрация» не поможет.
 Время жизни кода: `AuthLoginCodeExpiresInMinutes` в `appsettings.json` → `Auth` (по умолчанию 15 минут). Код регистрации/аутентификации — 6-значный и одноразовый: повторно использовать один код нельзя даже при параллельных запросах.
 Балансировщик нагрузки: без настройки `ForwardHeaders` все запросы воспринимаются как один IP → массовая блокировка.
 `"ForwardHeaders": {
  "Headers": "For,Proto,Host,Prefix",
  "Networks": "*",
  "Proxies": "*"
}
` Полная JSON-схема и типы входа: business.md#способы-входа-на-форме-авторизации

Ключ `RegistrationFields` определяет набор полей на форме самостоятельной регистрации.
 9 кодов полей:
 Код Тип Описание `Email` email Адрес почты `CellPhone` phone Мобильный телефон `Nick` string Псевдоним `FirstName` string Имя `LastName` string Фамилия `Gender` select Пол (1 = Мужской, 0 = Женский) `City` string Город `Password` password Пароль `Note` string Примечание (поддерживает HTML; доп. параметры `title` и `Color`) Поле `Note` выводит на форму регистрации произвольный текстовый блок с поддержкой HTML-разметки; его заголовок и цвет текста задаются параметрами `title` и `Color`.

 Структура элемента: `{"Key": "Email", "isRequired": true/false, "IsHidden": true/false}
`
 PasswordSettings (вложенная секция для поля Password) — 11 параметров:
 Параметр Описание `MinNumberOfChar` Минимальное количество символов `MaxNumberOfChar` Максимальное количество символов `UpperLowercaseRequired` Заглавные и строчные обязательны `AcceptableLanguage` Допустимый язык (например `ru-RU`) `MinNumberOfDigits` Минимум цифр `NoSpaces` Без пробелов `MinNumberOfSpecialChar` Минимум спец. символов `DisallowLoginOrBirthdayPattern` Запрет логина/даты рождения в пароле `DisallowedSequenceLength` Запрет последовательностей (йцукен, qwerty, 123) `DisallowedRepeatingPatternLength` Запрет повторяющихся паттернов (qweqwe, 123123) `NumberOfPreviousPasswordsToCheck` Проверка истории паролей (рекомендуется ≥ 10) `MinCharPasswordDifference` Мин. отличие от предыдущего пароля (рекомендуется ≥ 5) ⚠️ PasswordSettings действует только на форму самостоятельной регистрации. Для создания пользователей из режима администрирования и сброса паролей — `sys_general_settings`.
 7 правил регистрации:

- Одно из `CellPhone`, `Email`, `Nick` — обязательно

- Поле, использованное для верификации (телефон/email), скрывается на форме

- При заполнении `Nick` — пароль обязателен

- Пустой `Nick` автозаполняется из Phone или Email

- Пустой `DisplayName` автозаполняется из `FirstName LastName` или `Nick`

- При заданном `CellPhone` без `Email` (если Email обязателен) — `phone@domain`

- `Gender` по умолчанию = 1 (Мужской)
  Предзаполнение через URL: `~/spa/entry/signup?{RegistrationCode}={значение}`
 Параметр `source`: `spa_base` (веб) или `mobile` (МП) — используется в смарт-событии «После создания пользователя».
 Автозаполнение и бизнес-правила: business.md#самостоятельная-регистрация

Где настраивается: `synchronizations`, `synchronization-settings`.
 Таблицы БД:

- `dbo.SynchronizationProfiles`

- `dbo.SynchronizationProfilesADSettings`
  Что контролируется: импорт пользователей/групп и консистентность каталога для auth-проверок.
 Профиль синхронизации (`synchronizations`) связывается с сервисом через поле Сервис — выбирается один из настроенных AD/LDAP-сервисов.
 ⚠️ Для каждого сервиса может быть только одна активная настройка синхронизации.
 Ключевые параметры профиля:
 Параметр Описание Синхронизация с AD Синхронизация по расписанию `ADSyncJob` Данные пользователей Синхронизация полей профиля Названия групп Синхронизация имён групп Создание новых групп Авто-создание групп из AD Членство пользователей Синхронизация состава групп Вложенность групп Синхронизация иерархии групп Создавать пользователей из AD Авто-создание новых пользователей Логины в формате Domain\Nick Для работы в нескольких доменах с одинаковыми никами Только орг. единиц из AD Сохраняет ручные орг.единицы при синке Маска для создания групп Фильтр групп по маске Организационные единицы Дерево OU для синхронизации AD фильтр пользователей LDAP-фильтр пользователей (до 1000 символов) AD фильтр групп LDAP-фильтр групп Макс. допустимое кол-во обновлений Лимит — при превышении задание не выполняется Маппинг полей (встроенные): `company` → Компания, `department` → Отдел, `description` → Должность, `givenName` → Имя, `displayName` → Псевдоним (рус.), `telephoneNumber` → Рабочий, `mail` → E-mail, `thumbnailPhoto` → Аватар.
 ⚠️ Названия полей регистрозависимые — `displayname` и `DisplayName` распознаются как разные параметры.
 Детали синхронизации и AD-ограничения: ../users-and-groups/business.md

Дополнительные пользовательскые настройки приложения, влияющие на вход:
 Ключ Тип Назначение `ForbidTCLogin` `0` / `1` Если `1` — вход в приложение TaskCenter (платформа .NET Framework, legacy-клиент) проверяется на права. Если `0` — войти может любой пользователь. Используется для блокировки legacy-доступа `PasswordRecoveryText` string (Markdown) Пользовательскый текст на странице восстановления пароля (`/spa/entry/password-recovery`). Markdown-форматирование поддерживается `LogRefreshTokenRequests` bool Если `true` — обновления токенов мобильного клиента логируются в `LoginsLog` и отображаются в журнале пользователя/логах МП наряду с обычными входами. Используется для аудита

Частые проблемы входа, их причины и способ проверки:
 Симптом Причина Где проверить SQL-диагностика На форме логина нет нужного провайдера Провайдер неактивен/скрыт/не назначен по умолчанию `authentication-providers` `select ProviderID, Name, ProviderType, IsActive, IsHidden, IsDefault from dbo.AuthenticationProviders order by ProviderID;` Пользователь видит провайдер, но вход запрещён Нет привязки группы в `AuthenticationProviderGroups` Доступ к провайдеру по группам `select * from dbo.AuthenticationProviderGroups where ProviderId = @ProviderId;` LDAP/AD поиск пользователей в админке не работает Неверные учётные данные или адрес сервиса `active-directory-service-settings`, `openldap-service-settings` `select * from dbo.LDAPServicesCredentials; select * from dbo.OpenLDAPServicesCredentials;` Согласование SAML/OAuth завершается ошибкой Ошибка metadata/issuer/cert/callback-path `saml-service-settings`, `oauth-service-settings` `select * from dbo.SAMLServicesSettings; select * from dbo.OAuthServicesSettings;` Вход блокируется после серии ошибок Слишком жёсткие лимиты попыток / captcha `system-settings` `select FailedAttemptsCount, LastUpdate from dbo.IpList where Ip = @Ip; select LogFailsCount from dbo.Users where Nick = @Login;` PAT-токен не работает (401) Токен отозван, просрочен или повреждён `/api/*/pat/list` или БД `select TokenPrefix, IsRevoked, ExpiresAt, LastUsedAt from PersonalAccessTokens where UserId = @UserId;` Пользователь не может генерировать PAT Нет спецправа `GENERATEPAT` спецправа группы Проверить наличие права `GENERATEPAT` на группу пользователя Лимит токенов исчерпан `PatMaxTokensPerUser` достигнут `appsettings.json` `select count(*) from PersonalAccessTokens where UserId = @UserId and IsRevoked = 0;`

Способы аутентификации при обращении к API:
 Метод Заголовок Когда использовать PAT `1F-Pat: {token}` Production-окружение JWT `1FormaAuth: {jwt}` Тестовые стенды — PAT не работает Cookie `1FormaAuth={token}` Браузерные сессии `Authorization: Bearer {token}` → всегда 401. Не использовать.

Различия аутентификации между боевым окружением и тестовыми стендами:
 Аспект Production-окружение Тестовые стенды PAT Работает Не работает → только JWT SSL Доверенный сертификат Self-signed → `verify=False` / `-k` Авторизация `1F-Pat: {token}` `1FormaAuth: {jwt}` через `POST /api/auth/token-v2` Python 3.9 не использует macOS Keychain → `ssl.create_default_context()` с `check_hostname=False`, `verify_mode=CERT_NONE`.

Время жизни access-токена для Windows-аутентификации (IIS Integrated Windows Authentication) задаётся параметром `Auth.WinAuthTokenExpiresInMinutes` в `appsettings.json`.
 Поведение:

- Если параметр не задан (`null`) — используется значение из `Auth.AuthTokenExpiresInMinutes` (по умолчанию 1500 мин = 25 ч).

- Если задан — используется указанное значение (в минутах).
  Пример:
 `{
"Auth": {
"WinAuthTokenExpiresInMinutes": 1500
}
}
` Когда настраивать. Параметр добавлен после инцидента, в котором TTL Win-токена был жёстко зашит на 5 минут — это вызывало попап ввода пароля Windows в IIS каждые 5 минут. Параметр позволяет явно выровнять TTL Win-токена с обычным token lifetime.

С v2.268.302 поддерживается нативная Kerberos-аутентификация на Linux: ASP.NET Core схема Negotiate (`auth.AddNegotiate()`) включается, когда `Auth:WinAuthHost = "Kestrel"`. Платформа не поднимает IIS; за согласование Kerberos (SPNEGO/GSSAPI) отвечает Microsoft.AspNetCore.Authentication.Negotiate, инфраструктура (`krb5.conf`, keytab с SPN сервиса, `KRB5_KTNAME`) настраивается стандартными средствами Linux/контейнера — в коде платформы этого нет.
 appsettings.json:
 `{
  "Auth": {
    "WinAuthHost": "Kestrel",
    "Negotiate": {
      "RequireKerberos": false,
      "UserMapper": "UserClaimsMapperByUpn",
      "MapperConfig": {
        "UserProfileAttribute": "Nick"
      }
    }
  }
}
` `Auth:Negotiate:RequireKerberos: true` запрещает откат на NTLM — попытки NTLM-аутентификации отклоняются, и в лог пишется предупреждение «NTLM authentication rejected (RequireKerberos=true)». Использовать только когда KDC и SPN-инфраструктура гарантируют согласование Kerberos для всех клиентов; иначе пользователи получат ошибку входа.
 Endpoint статуса — `GET /api/admin/auth/kerberos/status` (требует прав администратора), возвращает `KerberosStatusDto`:
 Поле Значение `winAuthHost` значение из `Auth:WinAuthHost` (`IIS` / `Kestrel` / `None`) `negotiateSchemeRegistered` `true`, если ASP.NET Core зарегистрировал схему `Negotiate` (для Linux — индикатор что `AddNegotiate()` отработал на старте) `requireKerberos` значение из `Auth:Negotiate:RequireKerberos` `claimsTransformationType` активный класс трансформации: `WinClaimsTransformation` (Windows) или `NegotiateClaimsTransformation` (Linux) `platform` ОС сервера, например `Ubuntu 24.04.4 LTS` или `Microsoft Windows` Если `winAuthHost = "Kestrel"`, но `negotiateSchemeRegistered = false` — приложение стартовало без Windows-аутентификации, нужно проверить, как настраивается host.
 Тест маппинга UPN → пользователь — `GET /api/admin/auth/kerberos/test-mapping?upn={upn}&attribute={attribute}` (по умолчанию `attribute=Nick`). Без реального согласования Kerberos проверяет, найдёт ли настроенный маппер пользователя по переданному UPN: вызывает `UserClaimsMapperByUpn` с заданным `attribute` как полем профиля и возвращает найденного пользователя (либо признак «не найден»). Используется для верификации `ClaimsMapperConfig.UserProfileAttribute` до подключения Kerberos.
 Логирование протокола. При каждом входе через Negotiate в колонке `LoginsLog.AuthProtocol` сохраняется фактически использованный протокол (`Kerberos` / `NTLM` / `Negotiate` для остальных случаев) — основная диагностика «почему SSO не Kerberos» делается через эту колонку.
