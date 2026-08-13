# Онлайн-просмотр файлов — Решение проблем

Источник: https://help.1forma.ru/domains/files/online-preview-troubleshooting/

Руководство по диагностике проблем с онлайн-просмотром и редактированием файлов в 1Форме. Охватывает ONLYOFFICE, Р7-Офис и встроенный просмотрщик.
 Если площадка собрана как 1F Certificate Edition (ФСТЭК): онлайн-просмотр и редактирование документов (ONLYOFFICE / Р7-Офис / Office WebApps) в этой сборке отсутствуют по дизайну — подсистема исключена из сборки целиком. Отсутствие действия «Открыть в редакторе» и маршрутов `api/r7/*` — штатное поведение, а не ошибка настройки. Дальнейшая диагностика в этом руководстве — для полной сборки.

Проблема.
 При клике на файл MS Office происходит скачивание вместо онлайн-просмотра.

Действие при клике управляется настройкой `FileOnClickActionMode` на двух уровнях:
 Уровень Где настраивается Приоритет Пользователь Личные настройки пользователя → «Действие с файлами MS Office при клике» Высший (если не «По умолчанию») Общие настройки Кастомные настройки приложения (`sys_user`) Применяется, когда у пользователя стоит «По умолчанию» Значения `FileOnClickActionMode` (колонка `Users.FileOnClickActionMode`, тип `int`):
 Значение Код Описание 0 `Download` Скачать файл 1 `ShowInOfficeOnlineEditor` Открыть для просмотра в веб-интерфейсе 2 `EditInWebDav` Открыть для редактирования в десктопном приложении (WebDav) 3 `Default` По умолчанию (берётся из общих настроек) Типичный сценарий.
 У пользователя стоит `Default` (3) → система берёт значение из общих настроек → там стоит `Download` (0) → файл скачивается.
 Диагностика.
 `-- Проверить настройку конкретного пользователя
SELECT UserID, Login, FileOnClickActionMode
FROM Users
WHERE UserID = {userId};

-- Массовая проверка: сколько пользователей на каком режиме
SELECT FileOnClickActionMode, COUNT(*) AS Cnt
FROM Users
GROUP BY FileOnClickActionMode;
` Решение.

- Изменить настройку конкретного пользователя через интерфейс: Настройки пользователя → «Действие с файлами MS Office при клике» → «Открыть для просмотра в веб-интерфейсе».

- Изменить общую настройку для всех пользователей с режимом «По умолчанию»: Администрирование → Кастомные настройки приложения.



Симптом: «онлайн просмотр периодически не работает — падает ошибка резолва DNS». Ошибка появляется нестабильно, может проходить и возвращаться.
 Причина: сервер приложения 1Ф обращается к ONLYOFFICE по внутреннему hostname. Если ONLYOFFICE запущен в Docker, контейнер 1Ф (или хост) может не резолвить hostname ONLYOFFICE-сервера через стандартный DNS.
 Диагностика:

-  Проверить, какой `serverAddress` указан в настройке `OfficeOnlineEditor`: `SELECT [Key], [Value]
FROM SettingsCustom
WHERE [Key] = 'OfficeOnlineEditor';
`


-  С сервера приложения 1Ф проверить доступность ONLYOFFICE: `# Проверка DNS resolution
nslookup <hostname-из-serverAddress>

# Проверка доступности
curl -I <serverAddress>
`


-  Если ONLYOFFICE в Docker — проверить сетевую конфигурацию контейнера: `docker inspect <container_name> | grep -A 20 "Networks"
`

  Решение:
 Добавить `extra_hosts` в `docker-compose.yml` контейнера, который обращается к ONLYOFFICE (или наоборот):
 `services:
  app:
    extra_hosts:
      - "onlyoffice.local:192.168.1.100"
` Либо указать IP-адрес вместо hostname в `serverAddress`:
 `{"editor": "OnlyOffice", "settings": {"serverAddress": "https://192.168.1.100/office"}}
` Альтернативы:

- Использовать Docker bridge network для обоих контейнеров (если оба в Docker)

- Настроить DNS-сервер внутренней сети для резолва hostname

Симптом: документы не открываются в онлайн-редакторе, ошибка в консоли браузера при загрузке iframe.
 Диагностика:

- Проверить `serverAddress` (см. SQL выше).

- С сервера 1Ф — `curl -I <serverAddress>`.

-  Проверить, запущен ли контейнер/сервис ONLYOFFICE: `docker ps | grep -i office
systemctl status onlyoffice
`


-  Проверить логи ONLYOFFICE: `docker logs <onlyoffice-container> --tail 100
`

  Решение: перезапустить контейнер/сервис, проверить сертификаты, проверить firewall.

Симптом: редактор не открывается или открывается в неожиданном режиме.
 Что проверить:
 Параметр Что проверить `editor` Версия <= 2.264: `"r7"` для OnlyOffice. Версия >= 2.265: `"onlyOffice"` для OnlyOffice, `"r7"` только для Р7-Офис. Значение `"webApps"` ранее использовалось для Office Web Apps Server — в актуальной сборке не работает (OWA удалён после миграции на .NET Core). `serverAddress` URL доступен с сервера 1Ф `allowedIPs` IP сервера 1Ф входит в список (или список пуст) Secret в `appsettings.json` Секция `R7` → `Secret` задан и соответствует серверу `-- Проверка текущей конфигурации
SELECT [Key], [Value]
FROM SettingsCustom
WHERE [Key] = 'OfficeOnlineEditor';
` Корректный формат ключа: `{"editor": "onlyOffice", "settings": {"serverAddress": "https://domain/office", "allowedIPs": []}}
`

Симптом: файлы старых форматов Office не открываются или открываются с ошибками.
 Причина: ONLYOFFICE / Р7-Офис поддерживает преимущественно новые форматы (`.docx`, `.xlsx`, `.pptx`). Старые форматы (`.doc`, `.xls`, `.ppt`) могут открываться с ограничениями.
 Решение:

- Конвертировать файлы в новые форматы

- Скачать и открыть локально
  PDF не открывается в ONLYOFFICE.
 Симптом: PDF не открывается или открывается с ошибками в онлайн-редакторе.
 Что проверить:

- Поддерживает ли установленная версия ONLYOFFICE / Р7-Офис формат PDF

- PDF-файлы открываются встроенным просмотрщиком 1Ф (без ONLYOFFICE). Если встроенный просмотрщик не работает — проблема на стороне браузера

Симптом: `System.ArgumentNullException: fileKey` или `ArgumentException: Invalid key (tus:...)` при открытии office-документа из вложения .eml файла.
 Причина: До версии 2.266.554 система не поддерживала TUS-ключи (`tempfile:tus:...`) при работе с email-вложениями.
 Диагностика:

- Проверить версию системы.

- Проверить логи: `grep -i "Invalid key" logs/uniform-api.log
grep -i "ArgumentException\|ArgumentNullException" logs/tcclasslib.log
`
  Решение:

- Версия < 2.266.554 → обновить систему. Это критичное исправление.

- Версия >= 2.266.554 → проверить настройки TUS в `appsettings.json` и работу job `CleanExpiredFilesFromPreuploadedTusStoreJob`.
  Временное решение (для версий < 2.266.554): скачать файл и открыть локально.

Симптом: документ открывается только в режиме просмотра, хотя у пользователя есть права на редактирование. Проблема проявляется при большом числе одновременных пользователей.
 Причина: ONLYOFFICE Community Edition ограничивает число одновременных сессий редактирования — максимум 20 вкладок. При превышении все последующие подключения переходят в режим только для чтения.
 Диагностика:

- Проверить нагрузку на ONLYOFFICE: сколько пользователей одновременно работают.

- Логи ONLYOFFICE: ограничение фиксируется в логах сервера.
  Решение:

- Для > 20 одновременных редакторов: перейти на Enterprise-версию ONLYOFFICE или Р7-Офис

- Кластеризация CE не поддерживается. Можно запустить несколько независимых реплик, но при совместном редактировании одного документа возможна рассинхронизация

Где искать ошибки онлайн-просмотра:
 Лог Что содержит `logs/uniform-api.log` Ошибки API онлайн-редактора и работы с файлами `logs/tcclasslib.log` Ошибки онлайн-редактора: формат ключей, доступ к файлам Консоль браузера Ошибки загрузки iframe, CORS, сетевые ошибки Docker-логи ONLYOFFICE Внутренние ошибки сервера документов Типичные ошибки в логах:
 Ошибка Причина Решение `ArgumentException: Invalid key (tus:...)` Версия < 2.266.554, TUS-ключи не поддерживаются Обновление системы `ArgumentNullException: fileKey` fileKey не передан (null/empty) Проверить источник файла, логи `NotFoundException: Preuploaded file not found` Файл в PreUploadedFiles отсутствует или истёк Проверить TTL, job очистки `UnauthorizedException: Access denied` Нет прав на файл Проверить права пользователя DNS resolution error Hostname ONLYOFFICE не резолвится См. секцию 2.1

SQL-запросы для проверки конфигурации редактора:
 `-- 1. Настройки редактора
SELECT [Key], [Value]
FROM SettingsCustom
WHERE [Key] = 'OfficeOnlineEditor';

-- 2. Тип редактора
SELECT JSON_VALUE([Value], '$.editor') AS Editor,
       JSON_VALUE([Value], '$.settings.serverAddress') AS ServerAddress
FROM SettingsCustom
WHERE [Key] = 'OfficeOnlineEditor';

-- 3. Настройка действия при клике на файл (конкретный пользователь)
SELECT UserID, Login, FileOnClickActionMode
FROM Users
WHERE Login = '{login}';

-- 4. Preupload-файлы (диагностика проблем с eml)
SELECT ID, FileName, DateUploaded, UserID, DATALENGTH(FileContent) AS SizeBytes
FROM PreUploadedOnPostTaskFiles
WHERE ID = {id};

-- 5. Просроченные preupload-файлы (> 24 часа)
SELECT COUNT(*) AS ExpiredFiles
FROM PreUploadedOnPostTaskFiles
WHERE DateUploaded < DATEADD(HOUR, -24, GETDATE());
`

Команды для проверки доступности сервера редактора:
 `# С сервера 1Ф:
curl -I <serverAddress-из-OfficeOnlineEditor>

# Проверка конфигурации через API 1Ф:
curl -s -H "1F-Pat: <token>" "https://<instance>/api/r7/config"

# Если ONLYOFFICE в Docker:
docker ps | grep -i office
docker logs <container> --tail 50
` Чеклист перед обращением в поддержку 1Ф.
 Что приложить при обращении:

- Версия платформы

- Значение `OfficeOnlineEditor` из CustomSettings (без секретов)

- Результат `curl -I <serverAddress>` с сервера 1Ф

- Логи `uniform-api.log` и `tcclasslib.log` в момент ошибки

- Скриншот ошибки из консоли браузера

- Формат и источник файла (ДП, комментарий, Диск, eml-вложение)
