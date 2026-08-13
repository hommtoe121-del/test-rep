# КриптоПро УЦ 2.0 — Полная техническая документация

Источник: https://help.1forma.ru/domains/signatures/cryptopro-ca/

Полная техническая документация по интеграции КриптоПро УЦ 2.0 с 1Формой: требования к серверу, архитектура, дополнительные параметры на форме задачи, справочник функций Lua-библиотеки и C# API, процесс выпуска сертификата, отзыв, настройка кнопок МТФ и подпись PKCS#7. Для разработчиков и администраторов.



1.1. Программное обеспечение
 Компонент Версия Назначение .NET 9.0 x64 Среда выполнения серверной части 1Формы КриптоПро CSP 5.0+ Криптопровайдер ГОСТ (хранилище ключей, CAPI) КриптоПро УЦ 2.0 REST API удостоверяющего центра Windows Server 2016+ Хостинг IIS, хранилище сертификатов 1.2. Сертификаты в хранилище Windows
 В хранилище `LocalMachine\My` (или `CurrentUser\My`) должен быть установлен сертификат администратора УЦ с приватным ключом. Этот сертификат используется для: - mTLS — клиентская аутентификация при HTTP-запросах к REST API УЦ - PKCS#7 подпись — подписание запросов на выпуск/отзыв сертификатов (ГОСТ 34.10-2012)
 Thumbprint этого сертификата записывается в поле `Certificate` таблицы `CryptoProCertificationCenters`.
 1.4. Библиотека CryptoProCA в SmartScripts
 Lua-библиотека `CryptoProCA` поставляется миграцией `v2.264` и хранится в таблице `SmartScripts` с признаком `IsLibrary = true`.
 1.5. Плагины ЭП на клиенте
 Для операций, выполняемых в браузере (генерация CSR, установка сертификата): - КриптоПро ЭЦП Browser plug-in — работает через cadesplugin - Рутокен плагин — альтернативный вариант (PKCS#11)
 Выбор плагина настраивается через настройки ЭП задачи (`EdsPluginType`).

Создана миграцией `v2.162`. Содержит настройки подключения к УЦ.
 Поле Тип Описание `Id` int IDENTITY PK, используется как `raId` во всех вызовах `Name` nvarchar(max) Название УЦ (для отображения) `Address` nvarchar(max) Базовый URL REST API УЦ, например `https://ca.example.com` `UsersFolder` nvarchar(max) Папка для регистрации пользователей в УЦ `Certificate` nvarchar(max) Thumbprint сертификата администратора (из хранилища Windows) `Guid` uniqueidentifier Уникальный идентификатор записи
 Настройка: Администрирование → CryptoPro УЦ (через dbadmin-дерево).

Общая схема взаимодействия браузера, сервера 1Формы и REST API УЦ:
 `┌─────────────────────────────────────────────────────────────┐
│                        БРАУЗЕР                              │
│                                                             │
│  Кнопка МТФ → JS: tcCryptoLogic.newRequest()               │
│                   tcCryptoLogic.installCertificate()        │
│                   tcCryptoLogic.signFiles()                 │
│                                                             │
│  ┌─────────────────┐    ┌─────────────────┐                │
│  │ CryptoPro ЭЦП   │    │ Рутокен плагин  │                │
│  │ Browser plug-in  │    │                 │                │
│  └─────────────────┘    └─────────────────┘                │
└────────────────────────────┬────────────────────────────────┘
                             │ HTTP API
┌────────────────────────────┼────────────────────────────────┐
│                      СЕРВЕР 1Форма                          │
│                                                             │
│  Lua скрипт (SmartScript)                                   │
│    │                                                        │
│    ├── include("...CryptoProCA...")  ← Lua-библиотека       │
│    │     └── CryptoProCA.issue_certificate(...)              │
│    │                                                        │
│    ├── CRYPTO:send_request()    ← C#-функция CRYPTO         │
│    │     └── HTTP-запрос к REST API УЦ                      │
│    │           ├── mTLS (клиентский сертификат)              │
│    │           └── PKCS#7 подпись через Win32 CAPI           │
│    │                                                        │
│    ├── CRYPTO:parse_cert()      ← парсинг X509              │
│    └── CRYPTO:get_thumbprint()  ← SHA-1 отпечаток           │
│                                                             │
│                        ↓ HTTP/TLS 1.2                       │
│              КриптоПро УЦ 2.0 REST API                      │
│              /api/ra/users                                   │
│              /api/ra/certRequests                             │
│              /api/ra/certificates                             │
│              /api/ra/revRequests                              │
└─────────────────────────────────────────────────────────────┘
`

Уровни реализации интеграции:
 Слой Компонент Ответственность Lua библиотека `CryptoProCA` Бизнес-логика: опрос статуса, обработка ошибок, чтение/запись ДП Lua (низкоуровневые) глобальный объект `CRYPTO:*` HTTP-запросы, парсинг сертификатов, получение thumbprint Транспорт — HTTP к REST API УЦ: mTLS, PKCS#7 подпись, логирование JS (МТФ) `window.tcCryptoLogic` Клиентские операции: генерация CSR, установка сертификата, подпись файлов

Для полного цикла работы с сертификатами нужны следующие ДП:

Дополнительные параметры, необходимые для полного цикла работы с сертификатами:
 ДП Тип Заполняется Описание nameAttributes Строка (JSON) Вручную или скриптом JSON с атрибутами субъекта сертификата. Формат: `{"2.5.4.3":"Иванов Иван Иванович","1.2.643.3.131.1.1":"000000000000","2.5.4.6":"RU"}` userId УЦ Строка `CryptoProCA.register_or_update_user()` UUID пользователя в УЦ. Заполняется автоматически при первой регистрации, далее используется для обновления PKCS#10 запрос Строка (Base64) `tcCryptoLogic.newRequest()` (браузер) CSR в формате Base64. Генерируется криптоплагином на клиенте certRequestId Строка `CryptoProCA.issue_certificate()` UUID запроса на сертификат в УЦ. Заполняется автоматически Сертификат Строка (Base64) `CryptoProCA.issue_certificate()` rawCertificate в Base64. Заполняется автоматически при выпуске Thumbprint Строка `CryptoProCA.get_thumbprint()` SHA-1 отпечаток сертификата. Заполняется автоматически Имя ЦС Строка Вручную (опционально) authorityName — имя центра сертификации, если в УЦ их несколько

Важно: REST API КриптоПро УЦ 2.0 требует обязательное наличие полей Country (`2.5.4.6`) и Locality (`2.5.4.7`) в `nameAttributes` при регистрации пользователя (`POST /api/ra/users`). Без них запрос вернёт ошибку.
 OID Атрибут Обязательность Пример значения `2.5.4.3` CN (Common Name) Обязательный `Иванов Иван Иванович` `2.5.4.6` C (Country) Обязательный (REST API v2.0) `RU` `2.5.4.7` L (Locality) Обязательный (REST API v2.0) `Москва` `2.5.4.4` SN (Surname) Рекомендуемый `Иванов` `2.5.4.42` GN (Given Name) Рекомендуемый `Иван Иванович` `2.5.4.8` ST (State) Опционально `77 Москва` `2.5.4.10` O (Organization) Опционально `ООО "Компания"` `2.5.4.11` OU (Org Unit) Опционально `Отдел разработки` `1.2.643.3.131.1.1` ИНН Опционально `000000000000` `1.2.643.100.1` ОГРН Опционально `0000000000000` `1.2.643.100.3` СНИЛС Опционально `00000000000` `2.5.4.12` Title Опционально `Генеральный директор` `1.2.840.113549.1.9.1` E (Email) Опционально `user@example.com` Минимальный JSON для nameAttributes: `{
  "2.5.4.3": "Иванов Иван Иванович",
  "2.5.4.6": "RU",
  "2.5.4.7": "Москва"
}
`
 Полный JSON (рекомендуемый): `{
  "2.5.4.3": "Иванов Иван Иванович",
  "2.5.4.4": "Иванов",
  "2.5.4.42": "Иван Иванович",
  "2.5.4.6": "RU",
  "2.5.4.7": "Москва",
  "2.5.4.8": "77 Москва",
  "2.5.4.10": "ООО \"Компания\"",
  "1.2.643.3.131.1.1": "000000000000",
  "1.2.643.100.1": "0000000000000",
  "1.2.643.100.3": "00000000000",
  "1.2.840.113549.1.9.1": "user@example.com"
}
`

Поля Country и Locality также должны присутствовать в CSR-атрибутах, иначе сертификат не будет содержать эту информацию.
 `[
  {"rdn": "commonName",   "cpn": "CN",                  "value": "Иванов Иван Иванович"},
  {"rdn": "country",      "cpn": "C",                   "value": "RU"},
  {"rdn": "localityName", "cpn": "L",                   "value": "Москва"},
  {"rdn": "surname",      "cpn": "SN",                  "value": "Иванов"},
  {"rdn": "givenName",    "cpn": "G",                   "value": "Иван Иванович"},
  {"rdn": "organization", "cpn": "O",                   "value": "ООО Компания"},
  {"rdn": "inn",          "cpn": "1.2.643.3.131.1.1",   "value": "000000000000"},
  {"rdn": "snils",        "cpn": "1.2.643.100.3",       "value": "00000000000"}
]
` - `rdn` — имя атрибута для Рутокена - `cpn` — имя атрибута для КриптоПро ЭЦП Browser plug-in

Подключение: `include("[cryptopro] CryptoPro CA 2.0 — REST API library")` или `include(ID)` где ID — числовой идентификатор скрипта в SmartScripts.

Получает SHA-1 Thumbprint сертификата, записывает в ДП и оставляет silent-комментарий.
 Заменяет C#-смарт «Get Thumbprint»
 `CryptoProCA.get_thumbprint(taskId, epCertificateId, epThumbprintId)
` # Параметр Тип Обязательный Описание 1 `taskId` number да ID задачи. Обычно `CONTEXT.Id` 2 `epCertificateId` number да ID ДП (тип «Строка»), содержащего сертификат в формате Base64 (rawCertificate) 3 `epThumbprintId` number да ID ДП (тип «Строка»), куда будет записан SHA-1 Thumbprint Возвращает: `string` — Thumbprint (SHA-1, hex, uppercase)
 Побочные эффекты:

- Записывает Thumbprint в ДП `epThumbprintId` от имени системного робота

- Добавляет silent-комментарий в задачу: `Thumbprint: ABC123...`
  Пример: `include("[cryptopro] CryptoPro CA 2.0 — REST API library")

local tp = CryptoProCA.get_thumbprint(CONTEXT.Id, 503, 505)
-- tp = "A1B2C3D4E5F6..."
`

Регистрирует нового или обновляет существующего пользователя в УЦ.
 Заменяет C#-смарт «CryptoPro CA — new or update user»
 `CryptoProCA.register_or_update_user(raId, taskId, epNameAttrsId, epToSaveUserId)
` # Параметр Тип Обязательный Описание 1 `raId` number да ID записи в таблице `CryptoProCertificationCenters` (настройки подключения к УЦ) 2 `taskId` number да ID задачи. Обычно `CONTEXT.Id` 3 `epNameAttrsId` number да ID ДП (тип «Строка») с JSON `nameAttributes`. Обязательные поля JSON: `2.5.4.3` (CN), `2.5.4.6` (Country), `2.5.4.7` (Locality) — см. раздел 3.2 4 `epToSaveUserId` number да ID ДП (тип «Строка») для чтения/записи UUID пользователя в УЦ. Если пуст — создание, если заполнен — обновление Возвращает: `string` — UUID пользователя в УЦ
 REST API вызовы:

- Создание: `POST /api/ra/users` с телом `{ nameAttributes: {...}, folder: "..." }`

- Обновление: `POST /api/ra/users/{userId}` с телом `{ nameAttributes: {...} }`
  Пример: `include("[cryptopro] CryptoPro CA 2.0 — REST API library")

-- ДП 200 содержит: {"2.5.4.3":"Иванов Иван","2.5.4.6":"RU","2.5.4.7":"Москва"}
-- ДП 201 пуст (новый пользователь) или содержит UUID (обновление)
local userId = CryptoProCA.register_or_update_user(1, CONTEXT.Id, 200, 201)
`

Выпуск сертификата: submit + accept через двойную подпись PKCS#7, опрос, получение rawCertificate.
 Заменяет C#-смарт «CryptoPro CA — Issue certificate»
 `CryptoProCA.issue_certificate(raId, taskId, epUserId, epRequestId,
    epToSaveRequestId, epToSaveCertificateId, epAuthorityId, maxPollAttempts)
` # Параметр Тип Обязательный Описание 1 `raId` number да ID записи в `CryptoProCertificationCenters` 2 `taskId` number да ID задачи. Обычно `CONTEXT.Id` 3 `epUserId` number да ID ДП (тип «Строка») с UUID пользователя в УЦ (результат `register_or_update_user`) 4 `epRequestId` number да ID ДП (тип «Строка») с PKCS#10 запросом в Base64 (результат `tcCryptoLogic.newRequest`) 5 `epToSaveRequestId` number да ID ДП (тип «Строка») — сюда будет записан `certRequestId` (UUID запроса в УЦ) 6 `epToSaveCertificateId` number да ID ДП (тип «Строка») — сюда будет записан `rawCertificate` (Base64) 7 `epAuthorityId` number да ID ДП (тип «Строка») с именем центра сертификации (`authorityName`). Может быть пустым — если в УЦ один ЦС 8 `maxPollAttempts` number нет Количество попыток опрос (по умолчанию `10`). Каждая попытка = пауза 2 сек + GET-запрос Возвращает: `LuaTable` — полный объект сертификата из УЦ (содержит `rawCertificate`, `id`, `status` и др.)
 REST API вызовы (последовательно):

- `POST /api/ra/certRequests` — submit + accept, двойная PKCS#7 подпись

- `GET /api/ra/certificates?certRequestId=...` — опрос (до `maxPollAttempts` раз)

- `GET /api/ra/certificates/{id}` — полная форма с `rawCertificate`
  Пример: `include("[cryptopro] CryptoPro CA 2.0 — REST API library")

CryptoProCA.issue_certificate(
    1,            -- raId
    CONTEXT.Id,   -- taskId
    500,          -- ДП: userId в УЦ
    501,          -- ДП: PKCS#10 запрос
    502,          -- ДП: certRequestId (запись)
    503,          -- ДП: rawCertificate (запись)
    504           -- ДП: имя ЦС (может быть пустым)
)
`

Отзыв сертификата через REST API с двойной подписью.
 Заменяет C#-смарт «CryptoPro CA — revoke certificate»
 `CryptoProCA.revoke_certificate(raId, taskId, epCertificateId, revocationReason, fromTimestamp)
` # Параметр Тип Обязательный Описание 1 `raId` number да ID записи в `CryptoProCertificationCenters` 2 `taskId` number да ID задачи. Обычно `CONTEXT.Id` 3 `epCertificateId` number да ID ДП (тип «Строка») с сертификатом в Base64 4 `revocationReason` number нет Код причины отзыва (0–5, по умолчанию `0`). См. таблицу ниже 5 `fromTimestamp` number|nil нет Unix timestamp даты начала отзыва. Если > текущего времени — отложенный отзыв с `RD=yyyy-MM-ddTHH:mm:ss` Возвращает: `string` — серийный номер отозванного сертификата
 REST API вызов:

- `POST /api/ra/revRequests` — двойная PKCS#7 подпись, кодировка UTF-16LE без BOM
  Коды причин отзыва:
 Код Константа Описание 0 Unspecified Не указана 1 KeyCompromise Компрометация ключа 2 CACompromise Компрометация УЦ 3 AffiliationChanged Изменение принадлежности 4 Superseded Замена (выпущен новый) 5 CessationOfOperation Прекращение деятельности Пример: `include("[cryptopro] CryptoPro CA 2.0 — REST API library")

-- Немедленный отзыв по причине "Замена"
local sn = CryptoProCA.revoke_certificate(1, CONTEXT.Id, 503, 4)

-- Отложенный отзыв (с 01.02.2027)
CryptoProCA.revoke_certificate(1, CONTEXT.Id, 503, 0, 1801526400)
`

Приостановка сертификата. резерв через SOAP — REST API КриптоПро УЦ 2.0 не поддерживает `RR=6`.
 Заменяет C#-смарт «CryptoPro CA — pause certificate»
 `CryptoProCA.pause_certificate(taskId, raId, epCertificateId, fromTimestamp, toTimestamp, revocationReason)
` # Параметр Тип Обязательный Описание 1 `taskId` number да ID задачи. Обычно `CONTEXT.Id` 2 `raId` number да ID записи в `CryptoProCertificationCenters` 3 `epCertificateId` number да ID ДП (тип «Строка») с сертификатом в Base64 4 `fromTimestamp` number да Unix timestamp даты начала приостановки 5 `toTimestamp` number да Unix timestamp даты окончания приостановки 6 `revocationReason` number да Код причины (обычно `6` — CertificateHold) Реализация: на текущей версии вызывает старое смарт-действие приостановки. Когда REST API УЦ добавит поддержку — будет переписана на REST без изменения сигнатуры.
 Пример: `include("[cryptopro] CryptoPro CA 2.0 — REST API library")

-- Приостановить на месяц
CryptoProCA.pause_certificate(CONTEXT.Id, 1, 503, 1780000000, 1782000000, 6)
`

Возобновление приостановленного сертификата. резерв через SOAP — REST API не поддерживает `RR=8`.
 Заменяет C#-смарт «CryptoPro CA — resume certificate»
 `CryptoProCA.resume_certificate(taskId, raId, epCertificateId)
` # Параметр Тип Обязательный Описание 1 `taskId` number да ID задачи. Обычно `CONTEXT.Id` 2 `raId` number да ID записи в `CryptoProCertificationCenters` 3 `epCertificateId` number да ID ДП (тип «Строка») с сертификатом в Base64 Реализация: на текущей версии вызывает старое смарт-действие возобновления.
 Пример: `include("[cryptopro] CryptoPro CA 2.0 — REST API library")

CryptoProCA.resume_certificate(CONTEXT.Id, 1, 503)
`

Низкоуровневые C#-функции CRYPTO:* доступны в Lua без подключения библиотеки — это глобальный объект `CRYPTO`.

Отправка HTTP-запроса к REST API УЦ с mTLS и опциональной PKCS#7 подписью.
 `local resp = CRYPTO:send_request(raId, method, path, query, body, options, taskId)
` # Параметр Тип Обязательный Описание 1 `raId` number да ID записи в `CryptoProCertificationCenters` 2 `method` string да HTTP-метод: `"GET"` или `"POST"` 3 `path` string да Путь API (должен начинаться с `/api/ra/`), например `/api/ra/users` 4 `query` string|nil нет Query string без `?`, например `"certRequestId=abc-123"` 5 `body` LuaTable|nil нет Тело POST-запроса (конвертируется в JSON). `nil` для GET 6 `options` LuaTable|nil нет Опции подписи и таймаута (см. ниже) 7 `taskId` number|nil нет ID задачи (для логирования в AutomationScriptsLog) Поля `options`:
 Поле Тип По умолчанию Описание `sign` boolean `false` Подписать поле `rawRequest` в body PKCS#7 `signMode` string `"submitAndAccept"` `"submitAndAccept"` — двойная подпись, `"submit"`/`"approve"` — одинарная `encoding` string `nil` `"utf16le"` — кодировать строку UTF-16LE перед подписью. `nil` — Base64→бинарные данные `timeoutMs` number `30000` Таймаут HTTP-запроса в миллисекундах Возвращает: `LuaTable` с полями:
 Поле Тип Описание `StatusCode` number HTTP-код ответа (200, 201, 400, 500...) `Body` string|nil Тело ответа (raw string) `Error` string|nil Сообщение об ошибке (`nil` при успехе). Формат: `"ExceptionType: message"` При ошибке HTTP-транспорта (timeout, connection refused) ошибка не выбрасывается исключением, а возвращается в поле `Error`.

5.2. CRYPTO:parse_cert
 Парсинг Base64-сертификата с извлечением основных полей.
 `local info = CRYPTO:parse_cert(certBase64)
` Возвращает: `LuaTable` с полями:
 Поле Тип Описание `serialNumber` string Серийный номер `thumbprint` string SHA-1 отпечаток (hex, uppercase) `subject` string DN субъекта `issuer` string DN издателя `notBefore` string Дата начала действия (`yyyy-MM-ddTHH:mm:ss`) `notAfter` string Дата окончания действия (`yyyy-MM-ddTHH:mm:ss`) 5.3. CRYPTO:get_thumbprint
 `local tp = CRYPTO:get_thumbprint(certBase64)
` Возвращает: `string` — Thumbprint (SHA-1, hex, uppercase)

Кто выполняет: серверный Lua-скрипт (смарт-действие на шаге маршрута)
 `include("[cryptopro] CryptoPro CA 2.0 — REST API library")
CryptoProCA.register_or_update_user(raId, CONTEXT.Id, epNameAttrsId, epToSaveUserId)
` Что происходит:

- Читает JSON из ДП `epNameAttrsId` (обязательно: CN, Country, Locality)

- Если ДП `epToSaveUserId` пуст — `POST /api/ra/users` (создание, подставляет `UsersFolder` из настроек УЦ)

- Если ДП `epToSaveUserId` заполнен — `POST /api/ra/users/{userId}` (обновление атрибутов)

- Сохраняет UUID пользователя в ДП

Шаг 2. Генерация CSR (запроса на сертификат)
 Кто выполняет: браузер пользователя (кнопка на МТФ)
 `window.tcCryptoLogic.newRequest(CONTEXT.taskId, EP_ATTRS_ID, EP_CSR_SAVE_ID, {})
` Что происходит (на клиенте):

- Определяет тип плагина (CryptoPro / RuToken) через настройки ЭП задачи

- Считывает атрибуты из ДП (JSON-массив, формат см. раздел 3.3)

- Генерирует ключевую пару на клиенте

- Формирует PKCS#10 (CSR) в Base64

- Записывает CSR в ДП
  Шаг 4. Установка сертификата на клиенте
 Кто выполняет: браузер пользователя (кнопка на МТФ)
 `window.tcCryptoLogic.installCertificate(CONTEXT.taskId, EP_CERTIFICATE_ID)
` Что происходит (на клиенте):

- Считывает Base64-сертификат из ДП

- Устанавливает через криптоплагин (`CX509Enrollment.InstallResponse`)

- Сертификат привязывается к ключевой паре
  Шаг 5. Получение Thumbprint (опционально)
 Кто выполняет: серверный Lua-скрипт
 `include("[cryptopro] CryptoPro CA 2.0 — REST API library")
CryptoProCA.get_thumbprint(CONTEXT.Id, epCertificateId, epThumbprintId)
`

Кто выполняет: серверный Lua-скрипт
 `include("[cryptopro] CryptoPro CA 2.0 — REST API library")
CryptoProCA.issue_certificate(raId, CONTEXT.Id, epUserId, epRequestId,
    epToSaveRequestId, epToSaveCertificateId, epAuthorityId)
` Что происходит:

- Считывает userId и PKCS#10 из ДП

- Удаляет PEM-заголовки из PKCS#10

- `POST /api/ra/certRequests` с двойной PKCS#7 подписью

- Сохраняет `certRequestId` в ДП

- Опрос `GET /api/ra/certificates?certRequestId=...` (до 10 попыток × 2 сек)

- `GET /api/ra/certificates/{id}` — полная форма с `rawCertificate`

- Сохраняет `rawCertificate` в ДП

Отзыв (Revoke)
 `include("[cryptopro] CryptoPro CA 2.0 — REST API library")
CryptoProCA.revoke_certificate(raId, CONTEXT.Id, epCertificateId, revocationReason)
-- или с отложенной датой:
CryptoProCA.revoke_certificate(raId, CONTEXT.Id, epCertificateId, 0, fromTimestamp)
` Приостановка (Pause) — резерв через SOAP
 `include("[cryptopro] CryptoPro CA 2.0 — REST API library")
CryptoProCA.pause_certificate(CONTEXT.Id, raId, epCertificateId, fromTs, toTs, 6)
` Возобновление (Resume) — резерв через SOAP
 `include("[cryptopro] CryptoPro CA 2.0 — REST API library")
CryptoProCA.resume_certificate(CONTEXT.Id, raId, epCertificateId)
`

Кнопки МТФ вызывают функции через глобальный объект `window.tcCryptoLogic`.
 8.1. Кнопка «Сформировать запрос на сертификат»
 `window.tcCryptoLogic.newRequest(CONTEXT.taskId, EP_ATTRS_ID, EP_CSR_SAVE_ID, {})
`
- `EP_ATTRS_ID` — ID ДП с JSON-массивом атрибутов (формат раздел 3.3)

- `EP_CSR_SAVE_ID` — ID ДП, куда записывается PKCS#10 (Base64)

- Четвёртый параметр опционально принимает `{ extParamsId: { ContainerName: EP_CONTAINER_ID } }`
  8.2. Кнопка «Установить сертификат»
 `window.tcCryptoLogic.installCertificate(CONTEXT.taskId, EP_CERTIFICATE_ID)
`
- `EP_CERTIFICATE_ID` — ID ДП с Base64-сертификатом (rawCertificate)

- Требует: плагин КриптоПро ЭЦП Browser plug-in
  8.3. Кнопка «Сменить PIN»
 `window.tcCryptoLogic.changePin(EP_THUMBPRINT_ID, { taskId: CONTEXT.taskId })
`

9.1. Структура маршрута
 `[Шаг 1: Заполнение данных]
  ↓ Смарт при переходе: CryptoProCA.register_or_update_user(...)
[Шаг 2: Генерация запроса]
  Кнопка МТФ: tcCryptoLogic.newRequest(...)
  ↓
[Шаг 3: Выпуск сертификата]
  Смарт при переходе: CryptoProCA.issue_certificate(...)
  ↓
[Шаг 4: Установка и использование]
  Кнопка МТФ: tcCryptoLogic.installCertificate(...)
  Смарт при переходе: CryptoProCA.get_thumbprint(...)
` 9.3. Скрипт отзыва
 `include("[cryptopro] CryptoPro CA 2.0 — REST API library")

CryptoProCA.revoke_certificate(1, CONTEXT.Id, 503, 4) -- Superseded
`

Полный пример скрипта выпуска сертификата:
 `include("[cryptopro] CryptoPro CA 2.0 — REST API library")

local raId = 1                    -- ID УЦ из CryptoProCertificationCenters
local taskId = CONTEXT.Id

local epUserId = 500              -- ДП: userId в УЦ
local epRequestId = 501           -- ДП: PKCS#10 запрос (Base64)
local epToSaveRequestId = 502     -- ДП: certRequestId (для записи)
local epToSaveCertificateId = 503 -- ДП: rawCertificate (для записи)
local epAuthorityId = 504         -- ДП: имя ЦС (может быть пустым)

CryptoProCA.issue_certificate(
    raId, taskId,
    epUserId, epRequestId,
    epToSaveRequestId, epToSaveCertificateId,
    epAuthorityId
)
`

10.1. Алгоритм
 Подпись формируется штатными средствами .NET; для ГОСТ-ключей используется механизм Windows CryptoAPI (CAPI), так как стандартный путь .NET с ГОСТ-ключами не работает.
 10.2. Алгоритмы и OID ГОСТ
 Соответствие сигнатурных и хеш-OID применяемым алгоритмам ГОСТ:
 Сигнатурный OID Хеш OID Алгоритм 1.2.643.7.1.1.3.2 1.2.643.7.1.1.2.2 ГОСТ 34.10-2012 (256) → Стрибог-256 1.2.643.7.1.1.3.3 1.2.643.7.1.1.2.3 ГОСТ 34.10-2012 (512) → Стрибог-512 1.2.643.2.2.3 1.2.643.2.2.9 ГОСТ 34.10-2001 → ГОСТ 34.11-94 10.3. Режимы подписи (signMode)
 Режим Описание Применение `submitAndAccept` Двойная подпись: Sign(Sign(content)) Выпуск сертификата, отзыв `submit` / `approve` Одинарная подпись Если нужен только submit

Все запросы к REST API логируются в таблицу `AutomationScriptsLog`:
 Поле Значение `ObjectKey` `CRYPTO` `ObjectName` `POST /api/ra/certRequests`, `GET /api/ra/certificates` и т.д. `ObjectParams` Тело запроса (усечено до 500 символов) `AdditionalInfo` `HTTP 201: ...` или `ERROR: сообщение` (ответ усечён до 4000 символов) `TaskId` ID задачи (если передан) `Duration` Время запроса в мс Фильтр для просмотра: `ObjectKey = 'CRYPTO'`

Какая Lua-функция заменяет какое старое смарт-действие:
 # Старое C#-действие Новая Lua-функция Протокол 1 Get Thumbprint `CryptoProCA.get_thumbprint()` Локальный X509 2 CryptoPro CA — new or update user `CryptoProCA.register_or_update_user()` REST API 3 CryptoPro CA — Issue certificate `CryptoProCA.issue_certificate()` REST API 4 CryptoPro CA — revoke certificate `CryptoProCA.revoke_certificate()` REST API 5 CryptoPro CA — pause certificate `CryptoProCA.pause_certificate()` резерв через SOAP 6 CryptoPro CA — resume certificate `CryptoProCA.resume_certificate()` резерв через SOAP Ключевые отличия от старых смартов:

- Протокол: SOAP/COM → REST API + Win32 CAPI для подписи ГОСТ

- Формат данных пользователя: XML → JSON

- Запись в ДП выполняется от имени системного робота

- Обработка ошибок: ошибки возвращаются в поле `resp.Error`, а не выбрасываются исключением

Ограничения среды исполнения и обходные решения для скриптов:
 Проблема Обходное решение Ошибка HTTP-транспорта не выбрасывается исключением `CRYPTO:send_request()` возвращает её в поле `resp.Error` `UTILS:json_decode()` падает на простых JSON-строках (`"uuid"`) UUID из Body извлекается через `resp.Body:gsub('"', '')` `SQL:scalar("...", nil)` падает с ошибкой Всегда передавать `{}` вторым параметром `os.clock()` возвращает CPU-время процесса IIS (тысячи секунд) Для пауз используется `os.time()` (wall-clock) Стандартный путь .NET не подписывает ГОСТ-ключами Используется механизм Windows CryptoAPI (CAPI) REST API не поддерживает RR=6 (CertificateHold) резерв через SOAP через старый C#-смарт REST API не поддерживает RR=8 (RemoveFromCrl) резерв через SOAP через старый C#-смарт
