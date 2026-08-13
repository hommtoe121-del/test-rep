# PT Sandbox — интеграция с антивирусной песочницей

Источник: https://help.1forma.ru/domains/integrations/pt-sandbox/

Документ описывает интеграцию 1Формы с PT Sandbox для проверки файлов перед сохранением в хранилище. Внутри: архитектура, внешние вызовы API PT Sandbox, настройка через AdminSPA, фильтрация, кеш вердиктов, таблицы БД, исключения, enums и расширенные опции сканирования. Версия API PT Sandbox: v5.7 (`/api/v1`).
 Обзор интеграции PT Sandbox
 PT Sandbox (Positive Technologies Sandbox) — внешняя антивирусная песочница, которая проверяет файлы перед их сохранением в хранилище 1Формы. Интеграция выполняет перехват загрузки файлов на уровне `FileProviders.UploadFile` и блокирует вредоносные файлы до записи в БД/хранилище.
 Версия API PT Sandbox: v5.7 (`/api/v1`).

Проверка выполняется в одной точке — перед записью файла в хранилище. Все загрузки файлов через стандартный механизм 1Формы автоматически проходят через PT Sandbox, если сервис настроен и включён.

Проверка перед записью файла в хранилище идёт по шагам:
 `Загрузка файла
  └─ Проверка перед записью в хранилище
       ├─ Получить активные настройки (IsEnabled = true)
       ├─ Если не настроен → пропустить
       ├─ Сканирование:
       │    ├─ Фильтр по расширению (ExtensionFilterMode)
       │    ├─ Фильтр по домену (DomainFilterMode)
       │    ├─ Проверка размера (MaxFileSizeBytes / OversizeFileAction)
       │    ├─ SHA-256 хеш → поиск в кеше PTSandboxFileScans (TTL 24ч)
       │    ├─ Если кеш Threat → блокировка
       │    ├─ Если кеш Clean → вернуть без вызова API
       │    ├─ [Опционально] запись в PTSandboxFileScans (IsLoggingEnabled)
       │    ├─ Загрузка файла → POST /api/v1/storage/uploadScanFile
       │    ├─ Запуск проверки → POST /api/v1/analysis/createScanTask
       │    └─ Результат: Clean / Threat / Unknown / Unwanted
       └─ По результату:
            ├─ Threat → блокировка (ошибка HTTP 400 пользователю)
            ├─ Unwanted + Block → блокировка (как Threat)
            ├─ Unwanted + Warn → предупреждение пользователю, загрузка разрешена
            ├─ Oversize + Block → блокировка
            ├─ Unavailable + Block → блокировка
            └─ Clean / Skip / Unwanted+Skip → продолжить загрузку
`

Авторизация: заголовок `X-API-Key: {accessToken}`.
 Этап 1 — загрузка файла
 `POST {ApiUrl}/api/v1/storage/uploadScanFile
Content-Type: multipart/form-data
X-API-Key: {token}

form-data: file = <binary>

Ответ:
{
  "data": {
    "file_uri": "sfm-files:///...",
    "ttl": 3600
  },
  "errors": []
}
` Этап 2 — запуск проверки (синхронный режим)
 `POST {ApiUrl}/api/v1/analysis/createScanTask
Content-Type: application/json
X-API-Key: {token}

{
  "file_uri": "sfm-files:///...",
  "file_name": "document.docx",
  "short_result": false,
  "async_result": false,
  "options": {
    "mark_suspicious_files_options": { ... },  // из MarkSuspiciousOptionsJson; отсутствует, если поле null
    "sandbox": { ... }                          // из SandboxOptionsJson; отсутствует, если поле null
  }
}

Ответ:
{
  "data": {
    "scan_id": "uuid",
    "result": {
      "verdict": "DANGEROUS" | "CLEAN",
      "threat": "TrojanName",
      "duration": 12.5
    },
    "artifacts": [
      {
        "engine_results": [
          {
            "detections": [
              { "detect": "Trojan.Win32.Example" }
            ]
          }
        ]
      }
    ]
  },
  "errors": []
}
` Этап 3 — проверка статуса (асинхронный режим)
 `POST {ApiUrl}/api/v1/analysis/checkTask
Content-Type: application/json
X-API-Key: {token}

{ "scan_id": "uuid" }
` Текущая реализация использует синхронный режим (`async_result=false`) — ждёт вердикт в рамках одного HTTP-запроса до `ScanTimeoutSeconds`.

Настройки управляются через стандартный механизм внешних сервисов (`ServiceType.PTSandbox`).
 Через интерфейс AdminSPA
 Сервис PT Sandbox настраивается в AdminSPA: Администрирование → Сервисы → создать сервис типа «PT Sandbox» (или отредактировать существующий). Обязательные поля формы — `ApiUrl` (адрес API PT Sandbox) и `AccessToken` (токен `X-API-Key`, хранится в БД в зашифрованном виде); остальные поля соответствуют параметрам конфигурации — см. таблицу параметров ниже. Форма покрывает все параметры без исключения: `MarkSuspiciousOptionsJson` и `SandboxOptionsJson` редактируются как textarea с сырым JSON (пустое поле — PT Sandbox применяет собственные дефолты), список доменов вводится как хосты через запятую и резолвится в `DomainIds` на сервере по `Domains.Url`.

 Admin API маршруты
 Метод URL Описание `POST` `/api/services` Создать конфигурацию PT Sandbox `POST` `/api/services/settings` Сохранить настройки `GET` `/api/services/settings/{settingsId}` Получить настройки по ID `DELETE` `/api/services` Удалить конфигурацию `GET` `/api/admin/services/custom/{serviceType}` Список всех конфигураций PT Sandbox Все маршруты требуют прав администратора (`CheckAdminRights()`).

Полный набор параметров сервиса PT Sandbox:
 Поле Тип По умолчанию Описание `ApiUrl` string — Базовый URL API, например `https://sandbox.corp.local/api/v1` `AccessToken` string — Токен `X-API-Key`. Хранится в зашифрованном виде. Доступен для редактирования в AdminSPA → Сервисы → PT Sandbox (поле «Токен доступа», добавлено в v2.268) `IgnoreSslErrors` bool `false` Игнорировать ошибки SSL (для самоподписанных сертификатов) `IsEnabled` bool `false` Включена ли проверка `ScanTimeoutSeconds` int `300` Таймаут ожидания вердикта (секунды) `MaxFileSizeBytes` long `1 073 741 824` (1 ГБ) Максимальный размер файла для проверки `OversizeFileAction` enum `Block` Действие при превышении размера: `Skip` / `Block` `ExtensionFilterMode` enum `ExcludeList` Режим фильтрации по расширению `FileExtensionsToScan` string — Расширения для проверки (ScanList режим, через запятую без точки) `FileExtensionsToExclude` string — Расширения для исключения (ExcludeList режим) `DomainFilterMode` enum `ExcludeList` Режим фильтрации по домену `DomainIds` int[] — Список ID доменов (интерпретация зависит от DomainFilterMode) `FailPolicy` enum `Block` Действие при недоступности PT Sandbox: `Skip` / `Block` `IsLoggingEnabled` bool `false` Сохранять результаты в `PTSandboxFileScans` `FileActionsLogMode` enum `None` Запись в `FileActionsLog`: `None` / `BlockOnly` / `All` `UnwantedFileAction` enum `Skip` Реакция на вердикт «потенциально опасно» (`Unwanted`, v2.268+): `Skip` (разрешить, обратная совместимость) / `Warn` (разрешить, показать предупреждение пользователю) / `Block` (заблокировать, как угрозу) `MarkSuspiciousOptionsJson` string (JSON) `null` JSON-объект с флагами пометки подозрительных файлов (блок `mark_suspicious_files_options`). При `null` — блок не передаётся, PT применяет дефолты (все флаги = `true`). Подробнее: раздел ниже `SandboxOptionsJson` string (JSON) `null` JSON-объект настроек поведенческого анализа (блок `sandbox`). Ключевое поле — `Enabled` (включить/выключить запуск в ВМ). При `null` — блок не передаётся, PT применяет дефолты (поведенческий анализ включён). Подробнее: раздел ниже

Фильтр по расширению
 `ExtensionFilterMode` Логика `ExcludeList` (по умолчанию) Проверяются все файлы, кроме расширений из `FileExtensionsToExclude`. Пустой список = проверять все. `ScanList` Проверяются только расширения из `FileExtensionsToScan`. Пустой список = не проверять ничего. Фильтр по домену
 `DomainFilterMode` Логика `ExcludeList` (по умолчанию) Проверяются все домены, кроме из списка `DomainIds`. Пустой список = проверять для всех. `IncludeList` Проверяются только домены из `DomainIds`. Пустой список = не проверять вообще. Домен определяется по `Request.Host.Host` → поиск в таблице `Domains.Url`.

SHA-256 хеш файла используется для поиска предыдущих результатов проверки в таблице `PTSandboxFileScans`.

- TTL кеша: 24 часа

- Clean: возвращается из кеша без вызова API (если моложе 24 ч)

- Threat: всегда возвращается из кеша без повторной проверки (навсегда)

- Unknown: не кешируется, каждый раз проверяется заново
  Таблицы БД

Настройки конкретной конфигурации PT Sandbox. Связана с `ServiceSettings` (один к одному).
 Колонка Тип Описание `ServiceId` (PK/FK) int FK → `ServiceSettings.Id` `ApiUrl` nvarchar Базовый URL API `AccessToken` nvarchar Зашифрованный токен `IgnoreSslErrorsF` bit Игнорировать SSL `IsEnabled` bit Включена ли интеграция `ScanTimeoutSeconds` int Таймаут (сек) `MaxFileSizeBytes` bigint Максимальный размер (байты) `OversizeFileAction` int 0=Skip, 1=Block `ExtensionFilterMode` int 0=ExcludeList, 1=ScanList `FileExtensionsToScan` nvarchar Null `FileExtensionsToExclude` nvarchar Null `DomainFilterMode` int 0=ExcludeList, 1=IncludeList `FailPolicy` int 0=Skip, 1=Block `IsLoggingEnabled` bit Включить лог проверок `FileActionsLogMode` int 0=None, 1=BlockOnly, 2=All `UnwantedFileAction` tinyint 0=Skip, 1=Warn, 2=Block (добавлено в v2.268) `MarkSuspiciousOptionsJson` nvarchar(max) / text JSON-опции пометки подозрительных файлов (v2.267+) `SandboxOptionsJson` nvarchar(max) / text JSON-опции поведенческого анализа в ВМ (v2.267+) `PTSandboxDomainSettings`
 Список доменов для фильтрации. FK → `PTSandboxServiceSettings.ServiceId` и `Domains.Id`.
 Колонка Описание `ServiceId` FK → PTSandboxServiceSettings `DomainId` FK → Domains

Журнал результатов проверок (заполняется при `IsLoggingEnabled = true`).
 Колонка Тип Описание `Id` int PK `FileHash` binary(32) SHA-256 хеш содержимого `FileName` nvarchar Имя файла `FileSize` bigint Размер в байтах `UploadUserId` int Пользователь, загрузивший файл `DomainId` int? Домен загрузки `SandboxTaskGuid` uniqueidentifier? ID задания в PT Sandbox `Verdict` int 0=Clean, 1=Threat, 2=Unknown, 3=Skipped, 4=Unwanted `VerdictDetails` nvarchar JSON-ответ от PT Sandbox (угрозы, метаданные) `ScanRequestedAt` datetime Время начала проверки `ScanCompletedAt` datetime? Время завершения `ScanDurationMs` int? Длительность (мс) `ServiceId` int FK → PTSandboxServiceSettings.ServiceId `FileActionsLog`
 При `FileActionsLogMode != None` — записи с типами: - `PTSandboxSkipped` — файл пропущен фильтром - `PTSandboxClean` — проверен, угроз нет (только при `All`) - `PTSandboxBlocked` — заблокирован (угроза / oversize / unavailable)

Когда проверка прерывает загрузку — что её вызывает и что видит пользователь:
 Условие Когда Что видит пользователь Обнаружена угроза в файле найдена угроза «В файле {name} обнаружена угроза: {threats}» Потенциально опасный файл заблокирован вердикт `Unwanted` (UNWANTED/SUSPICIOUS) и `UnwantedFileAction=Block` (v2.268+) «В файле {name} обнаружена угроза: {threats}» (обрабатывается как угроза) Превышен размер файл > MaxFileSizeBytes и OversizeFileAction=Block «Файл {name} превышает…» Сервис недоступен PT Sandbox недоступен и FailPolicy=Block «Сервис недоступен: {reason}» При `FailPolicy=Skip` сервис недоступен — загрузка продолжается, в лог пишется `PTSandboxBlocked`.

При обнаружении угрозы клиенту передаются структурированные данные (имя файла, перечень угроз). Интерфейс распознаёт сценарий «угроза в файле» и показывает блок обратной связи — пометку ложного срабатывания и отправку фидбэка.
 Поведение при обнаружении угрозы (UX)
 При попытке прикрепить заражённый файл пользователь получает уведомление с именем файла и типом угрозы — формат: «В файле {name} обнаружена угроза: {threats}». Файл подсвечивается красным в интерфейсе и не сохраняется. Браузер продолжает работать штатно — зависания не происходит.
 Поведение при вердикте «потенциально опасно» (UX, v2.268+)
 При `UnwantedFileAction=Warn` файл с вердиктом `Unwanted` (UNWANTED/SUSPICIOUS) загружается успешно — блокировки нет. Предупреждение появляется в момент отправки комментария: если среди вложений есть хотя бы один такой файл, перед отправкой показывается диалог подтверждения («Внимание», кнопки «Отправить»/«Отмена») с текстом «Файл '{name}' потенциально опасен: {threats}». Комментарий отправляется только после подтверждения; при отмене можно удалить вложение или передумать. При `UnwantedFileAction=Skip` (по умолчанию) предупреждение не показывается вовсе — поведение как до v2.268.

`PTSandboxVerdict` | Значение | Описание | |---------|----------| | `Clean = 0` | Угроз нет | | `Threat = 1` | Обнаружена угроза | | `Unknown = 2` | Проверка не завершена (таймаут, ошибка) | | `Skipped = 3` | Пропущен фильтром | | `Unwanted = 4` | Потенциально опасен (вердикт PT Sandbox `UNWANTED` / `SUSPICIOUS`, в т.ч. вложенный `threat_level`). Не безусловная угроза — реакция зависит от `UnwantedFileAction` (v2.268+) |
 `PTSandboxFailPolicy` | Значение | Описание | |---------|----------| | `Skip = 0` | Разрешить загрузку при сбое (приоритет доступности) | | `Block = 1` | Блокировать при сбое (приоритет безопасности) |
 `PTSandboxOversizeFileAction` | Значение | Описание | |---------|----------| | `Skip = 0` | Пропустить проверку, загрузить без сканирования | | `Block = 1` | Заблокировать загрузку |
 `PTSandboxExtensionFilterMode` | Значение | Описание | |---------|----------| | `ExcludeList = 0` | Исключить перечисленные расширения | | `ScanList = 1` | Проверять только перечисленные расширения |
 `PTSandboxDomainFilterMode` | Значение | Описание | |---------|----------| | `ExcludeList = 0` | Исключить перечисленные домены | | `IncludeList = 1` | Проверять только перечисленные домены |
 `PTSandboxFileActionsLogMode` | Значение | Описание | |---------|----------| | `None = 0` | Не писать в FileActionsLog | | `BlockOnly = 1` | Только блокировки | | `All = 2` | Блокировки + успешные проверки |
 `PTSandboxUnwantedFileAction` (v2.268+) | Значение | Описание | |---------|----------| | `Skip = 0` | Разрешить загрузку прозрачно, без предупреждения (по умолчанию, обратная совместимость) | | `Warn = 1` | Разрешить загрузку, показать пользователю предупреждение при отправке комментария; событие пишется в `FileActionsLog` | | `Block = 2` | Заблокировать загрузку — обрабатывается как угроза (`Threat`) |

Два JSON-поля в `PTSandboxServiceSettings`, добавленные в v2.267. При сохранении проходят валидацию — невалидный JSON отклоняется с локализованной ошибкой. При `null` соответствующий блок не передаётся в PT Sandbox и сервер применяет собственные дефолты.

Управляет блоком `mark_suspicious_files_options`.
 Поле Тип Дефолт Описание `EncryptedNotUnpacked` bool `true` Помечать зашифрованные архивы, которые нельзя распаковать для анализа `MaxDepthExceeded` bool `true` Помечать файлы с превышением глубины распаковки вложенных архивов (защита от zip-bomb) `OfficeEncrypted` bool `true` Помечать зашифрованные документы Office (содержимое не проверяется) `OfficeHasMacros` bool `true` Помечать Office-файлы с макросами VBA `OfficeHasEmbedded` bool `true` Помечать Office со встроенными OLE-объектами `OfficeHasActiveX` bool `true` Помечать Office с ActiveX-элементами `OfficeHasDde` bool `true` Помечать Office с DDE-полями (Dynamic Data Exchange) `OfficeHasRemoteData` bool `true` Помечать Office, подгружающий данные с удалённых ресурсов при открытии `OfficeHasRemoteTemplate` bool `true` Помечать Office со ссылкой на удалённый шаблон (template injection) `OfficeHasAction` bool `true` Помечать Office с автодействиями при открытии/закрытии (`AutoOpen`, `Document_Open` и др.) `PdfEncrypted` bool `true` Помечать зашифрованные PDF `PdfHasEmbedded` bool `true` Помечать PDF со встроенными файлами `PdfHasOpenAction` bool `true` Помечать PDF с действием `OpenAction`, выполняющимся при открытии `PdfHasAction` bool `true` Помечать PDF с любыми триггерными действиями `PdfHasJavascript` bool `true` Помечать PDF, содержащий JavaScript `PdfProtected` bool `true` Помечать PDF с ограничениями (запрет печати/копирования/редактирования) Граничные случаи:

- `null` в БД → блок не передаётся, PT применяет дефолты (все 16 = `true`). Поведение до v2.267 сохраняется.

- Частичный JSON → отсутствующие поля заполняются дефолтами DTO (все `true`).

Управляет блоком `sandbox`.
 Поле Тип Дефолт Описание `Enabled` bool `true` Включает поведенческий анализ (запуск в ВМ). При `false` — только статический анализ (быстрее, но ниже детектирование неизвестных угроз) `ImageId` string — Идентификатор образа ВМ: `win7-sp1-x64`, `win10-1607-x64`, `astra-1.7-x64` и др. Доступные образы зависят от инсталляции PT. Обязательно если `Enabled = true` — проверяется при сохранении настроек: без него сохранение блокируется понятной ошибкой, а не откладывается до HTTP 400 от PT Sandbox при следующей проверке файла `AnalysisDuration` int `120` Длительность поведенческого анализа (секунды, минимум 10). Рекомендации PT: 120 для офисных документов, 300+ для EXE `Bootkitmon` bool `false` Режим анализа загрузочных модулей (буткиты/руткиты). Виртуалка перезагружается с инструментом мониторинга ранней стадии инициализации `AnalysisDurationBootkitmon` int `60` Длительность bootkitmon-анализа (секунды, минимум 10). Применяется только при `Bootkitmon = true` `SaveVideo` bool `false` Сохранять видеозапись экрана ВМ во время анализа (для отладки ложных срабатываний) `MitmEnabled` bool `false` MITM-расшифровка HTTPS-трафика ВМ для анализа C2-коммуникаций malware `CustomCommand` string? `null` Команда запуска файла в ВМ. Маркер `{file}` заменяется путём к файлу. При `null` PT использует дефолтный обработчик по типу `FileTypes` string[]? `null` Явный список групп файлов для поведенческого анализа (`executable-files/`, `adobe-acrobat/`, `microsoft-office/`). При `null` PT определяет тип автоматически Граничные случаи:

- `null` в БД → блок не передаётся, PT применяет дефолты (поведенческий анализ включён).

- Чтобы отключить поведенческий анализ — передать объект с `Enabled: false`, а не `null` всего поля.

- Минимально валидный объект: `Enabled` + `ImageId` (если `Enabled = true`).
