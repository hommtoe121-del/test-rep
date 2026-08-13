# Секреты интеграций — хранение и чтение

Источник: https://help.1forma.ru/domains/integrations/secrets/

Модуль Integration Secrets — централизованное хранение секретов внешних сервисов (логины, пароли, API-ключи, токены) через единый механизм чтения. Для администраторов при настройке интеграций и переходе с устаревших таблиц `*Credentials`. Описывает архитектуру хранилища, Admin API, смарт-скрипты, интерфейс администратора и внешнее хранилище (Passwork).

Модуль Integration Secrets обеспечивает централизованное хранение секретов внешних сервисов (логины, пароли, API-ключи, токены). Секрет идентифицируется по строковому ключу `serviceKey` и содержит произвольный payload — набор пар «поле → значение».
 Все серверные компоненты читают секреты через единый механизм. Прямая работа с таблицами `*Credentials` / `*Settings` в новом коде не используется.
 Каждый секрет содержит:

- `serviceKey` — уникальный строковый ключ-идентификатор секрета.

- `displayName` — отображаемое имя (обязательно, ограничено по длине).

- `payload` — набор пар «поле → значение» (`Login`, `Password`, `ApiKey`, `AccessToken` и др.). Значения хранятся в зашифрованном виде.

- `ownerUserId` — владелец секрета, если он заведён пользователем самостоятельно (self-service, см. §11). `NULL` — системный или административный секрет, заведённый через Admin API.

Настройка `UseIntegrationSecrets` работает как переключатель режима чтения секретов.
 Значение Поведение `true` Сначала ищет в `IntegrationSecrets`; если записи нет — обращается к устаревшим таблицам (`*Credentials` / `*Settings`) `false` (по умолчанию) Всегда обращается только к устаревшим таблицам Настраивается через интерфейс администратора → Настройки системы → Custom settings, ключ `UseIntegrationSecrets`.
 Порядок чтения секрета:
 `Чтение секрета
    ├── сначала новое хранилище IntegrationSecrets (зашифрованный payload)
    └── затем устаревшие таблицы *Credentials / *Settings (если записи в новом нет)
` При чтении из устаревших таблиц `serviceKey` разбирается как `{prefix}-{serviceId}`, и значение берётся из соответствующей таблицы. Чтение из устаревших таблиц в аудит не пишется.

Все 14 сервисов уже работают через единый механизм. При включении `UseIntegrationSecrets = true` они автоматически начинают читать из нового хранилища — изменений не требуется.
 Префикс `serviceKey` Сервис `ldap` Active Directory / LDAP `openldap` OpenLDAP `radius` RADIUS `diadoc` Диадок (ЭДО) `sbis` СБИС (ЭДО) `ews` Exchange (EWS) `oauth` OAuth `cryptopro-dss` КриптоПро DSS `azure-cognitive` Azure Cognitive Services `pt-sandbox` PT Sandbox (антивирусная песочница) `kontur-cloud` Контур.Облако `kontur-kedo` Контур.КЭДО `multifactor` Multifactor `jitsi` Jitsi

В отличие от 14 интеграционных сервисов выше, для почтовых ящиков (`mailbox-{MailBoxID}` — пользовательские из `EmailMailBoxes`; `servicemailbox-{Id}` — сервисные из `ServiceMailBoxes`) перенос записи в `IntegrationSecrets` отложен на следующие итерации. На данный момент:

- Чтение идёт стандартным маршрутом: новое хранилище `IntegrationSecrets` (если запись есть), иначе — устаревшая колонка ящика. Значения с префиксом `VENC:` расшифровываются новым механизмом шифрования, остальные — старым (ради обратной совместимости).

- Запись ведётся в те же колонки `EmailMailBoxes` / `ServiceMailBoxes`, но шифруется уже новым механизмом. Установка `UseIntegrationSecrets = true` для почтовых ящиков пока ничего не меняет — пока никто не запишет payload в `IntegrationSecrets` (через Admin API или скрипт), значение берётся из устаревшей колонки.

- Поля payload (для будущего переноса записи в `IntegrationSecrets`): для пользовательского ящика — `Password`, `CalDavPassword`; для сервисного — `Password` (см. § 6).

- Массовая перешифровка устаревших значений на новый механизм — отдельная утилита (запускать после раскатки кода на все экземпляры). Подробности — в Хранение паролей почтовых ящиков.

Кодовых изменений не требуется. Достаточно:

- Включить `UseIntegrationSecrets = true` в custom settings (однократно для всего экземпляра).

- Создать запись в `IntegrationSecrets` с нужным `serviceKey` через Admin API или смарт-скрипт:
  `UTILS.set_secret_payload("diadoc-1", { Login: "user@domain.ru", Password: "secret" });
` После этого платформа начнёт возвращать значения из нового хранилища. Устаревшие записи продолжают работать как резерв для остальных сервисов, у которых записи ещё нет.

Маршрут: `api/admin/secrets` (только права администратора).
 Метод URL Назначение `GET` `api/admin/secrets` Список секретов (без значений payload) `GET` `api/admin/secrets/{serviceKey}` Карточка секрета (только `payloadKeys`) `GET` `api/admin/secrets/{serviceKey}/payload` Расшифрованный payload (фиксируется в аудите) `PUT` `api/admin/secrets/{serviceKey}` Создание / обновление `DELETE` `api/admin/secrets/{serviceKey}` Деактивация (не физическое удаление) `GET` `api/admin/secrets/contracts` Список зарегистрированных контрактов payload `GET` `api/admin/secrets/{serviceKey}/contract` Контракт для конкретного ключа `POST` `api/admin/secrets/{serviceKey}/audit` Аудит по секрету `POST` `api/admin/secrets/audit` Общий аудит с фильтрацией `POST` `api/admin/secrets/{serviceKey}/migrate-to-external` Перенос локального секрета в alias на внешнее хранилище (тело: `ExternalServiceId`, `ExternalPath`). Создаёт элемент во внешнем хранилище Passwork, обнуляет `PayloadEncrypted` в БД. Аудит: `MigratedToExternal` `POST` `api/admin/secrets/{serviceKey}/migrate-to-local` Обратная миграция: payload вычитывается из внешнего хранилища, шифруется и сохраняется в БД 1F. Item во внешнем хранилище не удаляется (защита от потери данных — удаляет админ через UI хранилища). Аудит: `MigratedToLocal` `GET` `api/admin/secrets/external/storages` Список инстансов внешних хранилищ (для UI «куда мигрировать»). Берётся из `ServicesSettings` по `ExternalServiceType in { Passwork }` `POST` `api/admin/secrets/external/{serviceId:int}/passwork/refresh-tokens` Ручное обновление access/refresh-токенов Passwork-инстанса `POST` `api/admin/secrets/external/{serviceId:int}/verify` Проверочный запрос к внешнему хранилищу: статус `Ok` / `Warning` / `Critical`. Результат сохраняется для диагностики `POST` `api/admin/secrets/legacy/migrate-all` Массовая миграция устаревших `*Credentials`-таблиц в единое `IntegrationSecrets`. Возвращает счётчики по сервисам `POST` `api/admin/secrets/legacy/migrate` Миграция одного устаревшего сервиса (тело — `serviceKey` либо `serviceType` + `serviceId`) Разделение метаданных и значений. Списковые и карточечные методы возвращают только `payloadKeys` — имена полей payload без их значений. Расшифрованные значения доступны исключительно через отдельный маршрут (`/payload`). Каждый вызов этого endpoint фиксируется в аудите.
 Деактивация vs удаление. `DELETE` выполняет деактивацию записи, а не физическое удаление. Это сохраняет историю в журнале аудита и позволяет отменить операцию при необходимости.

Для зарезервированных префиксов зарегистрированы контракты — допустимые поля payload. При создании/обновлении секрет валидируется против контракта. Список префиксов совпадает с таблицей в п. 3.
 Если `serviceKey` не относится ни к одному зарезервированному префиксу, контракт отсутствует и валидация по полям не выполняется.
 При `PUT api/admin/secrets/{serviceKey}` платформа проверяет:

- `serviceKey` — должен соответствовать допустимому формату ключа.

- `displayName` — обязателен, ограничен по длине.

- `payload` — должен быть непустым объектом.

- Для зарезервированных префиксов дополнительно проверяется соответствие payload зарегистрированному контракту (допустимые поля).
  Каждое обращение к payload фиксируется с полями: `serviceKey`, тип действия, пользователь, момент, IP, источник (`adminApi` / `smartScript` / `userSelfService`). Чтение из устаревших таблиц в аудит не попадает.
 Значения `IntegrationSecretAuditAction`: `Read`, `Created`, `Updated`, `Deleted`, `MigratedToExternal` (payload перенесён во внешнее хранилище — alias, см. § 9), `MigratedToLocal` (обратная миграция alias → локальное хранение).

Чтение и запись секретов из смарт-скриптов (JavaScript и Lua):
 `// JavaScript — чтение
var apiKey = UTILS.get_secret_value("sbis-1", "ApiKey");
var creds  = UTILS.get_secret_payload("sbis-1");

// JavaScript — запись
UTILS.set_secret_value("sbis-1", "ApiKey", newKey);
UTILS.set_secret_payload("sbis-1", { ApiKey: newKey, Login: "user" });
` `-- Lua — чтение
local apiKey = UTILS:get_secret_value("sbis-1", "ApiKey")
local creds  = UTILS:get_secret_payload("sbis-1")

-- Lua — запись
UTILS:set_secret_value("sbis-1", "ApiKey", newKey)
UTILS:set_secret_payload("sbis-1", { ApiKey = newKey, Login = "user" })
` Если секрет или поле не найдены — возвращается `null`, исключение не бросается. Каждый вызов фиксируется в аудите с источником `smartScript`.

Маршрут: `/administration/secrets` (только права администратора). Страница для создания, просмотра и редактирования записей `IntegrationSecrets` через `api/admin/secrets` (см. § 5).
 Экраны:
 Экран URL Назначение Список секретов `/administration/secrets` Таблица с фильтрацией. Кнопки «Создать», «Общий журнал аудита» Форма секрета side-panel из списка Создание / редактирование Журнал по секрету `/administration/secrets/{serviceKey}/audit` Аудит операций конкретного секрета Общий журнал `/administration/secrets/audit` Аудит по всем секретам с фильтрацией Колонки списка: `ServiceKey` · `displayName` · `Status` (чип: «Активен» / «Деактивирован») · `Payload` (chips, имена ключей) · `Created` · `Updated`.
 Контекстное меню строки: Редактировать · Показать значения · Аудит · Деактивировать (см. § 5 — `DELETE` выполняет деактивацию, а не физическое удаление).
 Форма создания / редактирования:

- `Service Key` — обязательный идентификатор. Регэксп: `[a-z0-9.-]+`, ≥2 символов. При редактировании поле disabled (изменение ключа делает аудит бессмысленным; для смены — пересоздание).

- `Название` (`displayName`) — опциональный, до 512 символов.

- `Payload` — динамический массив пар «ключ → значение», ключ до 128 символов, значение до 8192. Структура payload произвольная: набор полей определяется тем, что вводит админ, а не зафиксированной схемой (контракт по § 6 применяется только к зарезервированным префиксам). Дубликат ключа — валидационная ошибка. Значения вводятся скрытыми, с показом по иконке-глазу. Автозаполнение браузера отключено.

- Можно добавлять/удалять пары через «Добавить поле».
  Просмотр payload (readonly):
 ReadOnly-таблица с колонками «Ключ», «Значение», «Действие». Значения по умолчанию замаскированы как `••••••••`; кнопка-глаз раскрывает конкретное поле — каждый раскрытие фиксируется в аудите как `Read` через `GET api/admin/secrets/{serviceKey}/payload`.
 Журнал аудита:
 Фильтр Значение Период datepicker (С / По) с точностью до минуты Действие чипы: `Created`, `Updated`, `Read`, `Deleted` Пользователь инициатор операции Пагинация стандартная (skip/take) Каждая строка журнала раскрывается до деталей операции (см. поля `AuditEntry` в § 5 и таблицу `IntegrationSecretAudit`).
 Связанная возможность: конфигурация самих сервисов интеграций (URL, порты, флаги) управляется через отдельную форму `/administration/services` — см. Сервисы — единая форма AdminSPA. `ServiceType` в Сервисах соответствует префиксу `serviceKey` в Секретах.

С v2.268 запись в `IntegrationSecrets` может хранить payload не в самой 1F, а во внешнем хранилище (Passwork v7 — единственный реализованный провайдер; механизм рассчитан на добавление других). Локальная строка в этом случае становится alias — указателем на элемент во внешнем хранилище.

Колонки, добавленные к таблице `IntegrationSecrets`:
 Колонка Тип Семантика `ExternalServiceId` int NULL (FK → `ServicesSettings(Id)`) Инстанс внешнего хранилища, в котором лежит payload. NULL для локальных записей `ExternalSecretId` nvarchar(256) NULL Идентификатор элемента во внешнем хранилище (для Passwork — `itemId`) `PayloadEncrypted` varbinary(max) NULL Стало nullable. Для alias-записей всегда `NULL` — payload лежит снаружи Инвариант: либо `PayloadEncrypted IS NOT NULL` и `ExternalServiceId IS NULL` (локальный секрет), либо наоборот (alias). Гибридного режима нет.

Шаги переноса payload во внешнее хранилище:

- Админ вызывает `POST api/admin/secrets/{serviceKey}/migrate-to-external` с телом `{ ExternalServiceId, ExternalPath }`.

- Платформа расшифровывает текущий `PayloadEncrypted` и формирует элемент для внешнего хранилища.

- Элемент создаётся в хранилище Passwork — в хранилище по умолчанию или по пути из `ExternalPath`.

- После успешного ответа запись `IntegrationSecrets` обновляется атомарно: `PayloadEncrypted = NULL`, проставляются `ExternalServiceId` и `ExternalSecretId`.

- Аудит: `MigratedToExternal`.
  Запрос отклоняется, если у Passwork-инстанса выключена запись (`SupportsWrite = 0`) либо последняя проверка вернула статус `Critical`.

Поток «обратная миграция» (`MigratedToLocal`):

- `POST api/admin/secrets/{serviceKey}/migrate-to-local` (без тела).

- Платформа читает элемент из внешнего хранилища.

- Payload шифруется штатным механизмом и пишется в `PayloadEncrypted`.

- `ExternalServiceId` и `ExternalSecretId` обнуляются.

- Элемент во внешнем хранилище не удаляется (защита от случайной потери — удаление остаётся ручной операцией в интерфейсе хранилища).

- Аудит: `MigratedToLocal`.
  Чтение и запись alias-секретов в реальном времени зависят от `ExternalServiceId`: - `NULL` — payload расшифровывается из локального `PayloadEncrypted`. - `NOT NULL` — значение читается из внешнего хранилища по `ExternalSecretId`.
 Запись по alias (`UTILS.set_secret_value` из смарт-скриптов и интерфейса администратора) симметрична: значение обновляется во внешнем хранилище, локальный `PayloadEncrypted` остаётся `NULL`.
 Проверка / диагностика: `POST api/admin/secrets/external/{serviceId}/verify` запускает проверочный запрос к внешнему хранилищу и возвращает статус (`Ok` / `Warning` / `Critical`). Используется на странице «Сервисы» и в самодиагностике.

Параллельно с external-storage в v2.268 завершается миграция от множественных таблиц `*Credentials` (по сервису) к единому `IntegrationSecrets`:

- `POST api/admin/secrets/legacy/migrate-all` — пройти по всем устаревшим источникам секретов и перенести payload в `IntegrationSecrets` (с шифрованием). Возвращает счётчики по сервисам.

- `POST api/admin/secrets/legacy/migrate` — то же для одного сервиса (`serviceKey` либо `serviceType` + `serviceId`).
  После миграции устаревшие таблицы остаются как резерв на чтение (на случай если кто-то ещё пишет в них напрямую), но основным источником становится `IntegrationSecrets`.

Устаревшие таблицы `*Credentials` / `*Settings` шифровали значения старым механизмом: Rijndael-CBC с ключом из `Rfc2898DeriveBytes` и статической солью. Для миграции в `IntegrationSecrets` каждое такое хранилище получило собственный резолвер в `Valhalla.Secrets/Provider/Legacy/` — класс с суффиксом `LegacySecretResolver`, который читает старую запись, расшифровывает её унаследованным алгоритмом и передаёт открытый payload в `EncryptionService` для перешифровки в новый формат AES-GCM v2 (префикс `VENC:`) перед записью в `IntegrationSecrets.PayloadEncrypted`.
 Резолвер Префикс `serviceKey` Источник данных `ADLegacySecretResolver` `ldap` Active Directory `LDAPLegacySecretResolver` `openldap` OpenLDAP `OAuthLegacySecretResolver` `oauth` OAuth-провайдеры `RadiusLegacySecretResolver` `radius` RADIUS `JitsiLegacySecretResolver` `jitsi` Jitsi `S3LegacySecretResolver` `s3` S3-совместимые хранилища `PassworkLegacySecretResolver` `passwork` Passwork (внешнее хранилище секретов) `DiadocLegacySecretResolver` `diadoc` Диадок (ЭДО) `KonturCloudLegacySecretResolver` `kontur-cloud` Контур.Облако `KonturKEDOLegacySecretResolver` `kontur-kedo` Контур.КЭДО `EWSLegacySecretResolver` `ews` Exchange Web Services `MultifactorLegacySecretResolver` `multifactor` Multifactor `PTSandboxLegacySecretResolver` `pt-sandbox` PT Sandbox `SbisLegacySecretResolver` `sbis` СБИС (ЭДО) `CryptoProDSSLegacySecretResolver` `cryptopro-dss` КриптоПро DSS `AzureCognitiveLegacySecretResolver` `azure-cognitive` Azure Cognitive Services Резолверы вызываются из `POST api/admin/secrets/legacy/migrate` и `legacy/migrate-all` (см. § 9.4). Сама старая крипта в исходных таблицах не переписывается — резолвер работает read-only с legacy-источником и пишет уже новый формат только в `IntegrationSecrets`.

Integration Secrets — секреты внешних сервисов (API-ключи, логины, токены). Управляются через `ISecretsProvider` и Admin UI, видны администратору в интерфейсе.
 Мастер-ключ платформы (`Settings.EncKey`) — корневой ключ шифрования самой платформы, которым шифруются в том числе payload-ы секретов интеграций. Настраивается через `appsettings.json` и `masterkey-tool`, не отображается в UI. Описание режимов защиты и процедура смены ключа: `platform/backend/master-key-management.md`.

Помимо административного пути (§5), пользователь может завести собственные секреты в ограниченном неймспейсе — без прав администратора. Сценарий — личный токен интеграции (например, для внешнего API или бота), которым пользуется только сам сотрудник: не нужно просить администратора заводить общий секрет.
 Владение. Колонка `OwnerUserId` (`IntegrationSecrets`, FK → `Users.UserId`, `ON DELETE NO ACTION`) — `NO ACTION`, а не `SET NULL`, т.к. на ту же таблицу уже ссылается `UpdatedByUserId` с `SET NULL`, и SQL Server не допускает несколько cascade-путей от одной таблицы. Индекс `IX_IntegrationSecrets_Owner` — по (`OwnerUserId`, `ServiceKey`). `NULL` в `OwnerUserId` — секрет системный/административный, заведён не через self-service.
 Разрешённый неймспейс и формат ключа. Зарегистрировано два self-service контракта с разными префиксами `serviceKey`:

- `token` — ключ `token-u{uid}_{slug}`, payload валидируется по контракту `TokenSecretPayload`.

- `custom` — ключ `custom-u{uid}_{slug}`, `FreeForm` — контракта полей нет, payload произвольный.
  `{slug}` — `^[a-z0-9._]{1,64}$`. Формирование ключа (`BuildServiceKey`) — fail-closed: попытка обратиться к чужому `uid` в ключе завершается `403 AccessDeniedException`, не `404`.
 User API. Маршрут `api/user/secrets`, весь контроллер — `[Authorize]`.
 Метод URL Назначение `GET` `api/user/secrets` Список собственных секретов пользователя `GET` `api/user/secrets/contracts` Доступные self-service контракты (`token`, `custom`) `GET` `api/user/secrets/{serviceKey}` Карточка собственного секрета `POST` `api/user/secrets` Создание / обновление собственного секрета `DELETE` `api/user/secrets/{serviceKey}` Деактивация собственного секрета Владение проверяется на каждой операции (`EnsureOwned`): секрет с чужим `OwnerUserId` или без владельца (системный) через User API недоступен.
 Аудит. Источник фиксируется как `userSelfService` (см. §6), с IP-адресом обращения.
 Отличие от Admin API. Пользователь видит и меняет только собственные секреты в разрешённом неймспейсе (`token-u{uid}_*`, `custom-u{uid}_*`); доступа к чужим и к системным/административным секретам (вне self-service неймспейса) через User API нет.
