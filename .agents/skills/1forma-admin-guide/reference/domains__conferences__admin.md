# Видеоконференции — Администрирование

Источник: https://help.1forma.ru/domains/conferences/admin/

Администрирование видеоконференций в 1Форме: механизмы настройки (автоадминистрирование, API администрирования), ключевые параметры сервиса ВКС (Jitsi), телефонии и комнат, типичные ошибки и их диагностика. Для администраторов платформы и специалистов поддержки. Иерархия приоритетов настроек комнаты (глобальные → категория → индивидуальные) описана в иерархии настроек ВКС.

Администрирование конференций использует две группы механизмов: автоадминистрирование (3 формы) и API администрирования (мониторинг комнат).
 Формы автоадминистрирования:
 Alias формы Название Таблица БД Папка `vks-service-settings` Сервис ВКС dbo.JitsiServiceSettings Подключения `telephony-settings` Настройки IP Телефонии dbo.TelephonySettings Служебное `telephony-actions-packs` Настройки событий IP Телефонии dbo.TelephonyActionsPacks Служебное Маршруты API администрирования:
 Маршрут Методы Назначение `/api/admin/conference/rooms` GET Мониторинг статуса конференц-комнат

Сервис ВКС настраивается через автоадминку в форме `vks-service-settings` и хранится в таблице `dbo.JitsiServiceSettings`. 19 полей определяют подключение к Jitsi и параметры интеграции:
 Группа полей Что контролирует Домен, секрет, URL Подключение к серверу Jitsi QueueId, MessageFlowId Очереди для пост-обработки (транскрипты, видео) Настройки записи Автоматическая запись и транскрибация Параметры комнат Lobby, пароли, ограничения по умолчанию Форма имеет 2 зависимости между полями (условная видимость параметров). Значения используются при генерации токенов, создании комнат и обработке webhook-событий.

Два смарт-фильтра контролируют доступ к комнатам ВКС:
 Поле Тип Контракт Описание `CheckIsModeratorSmartScriptId` int? FK → SmartScripts TSQL/Lua → `bool` `true` — пользователь является модератором комнаты. Заменяет устаревший `IsModeratorSmartFilterId` (SmartFilter). `CheckSecurityBypassSmartScriptId` int? FK → SmartScripts TSQL/Lua → `bool` `true` — авторизованный пользователь пропускается в комнату без ожидания в лобби. Отличается от модераторства: bypass не даёт прав управления, только мгновенный вход. ⚠️ Пустое значение `CheckSecurityBypassSmartScriptId` интерпретируется как `true` — система не запрашивает у модератора подтверждение для авторизованных пользователей. Не распространяется на гостей (неавторизованных).
 В форме настройки сервиса ВКС (`/spa/administration/services`) оба поля используют контрол выбора смарт-скриптов (`smart-script`): при выборе скрипта отображается его название, а клик по полю открывает редактор смарт-скриптов по адресу `/spa/administration/smart-script?id={SmartScriptId}`.
 Пять дополнительных параметров в поле `ExtInfo` (JSON):
 Ключ Назначение `externalModuleKey` Ключ для доступа к API модуля ВКС `roomCreationKey` Ключ аутентификации Outlook-плагина при генерации временной комнаты `recordHost` Адрес хоста ссылок на записи ВКС `recordsApiKey` Ключ аутентификации для приёма событий записи ВКС `internalModuleKey` Внутренний ключ модуля ⚠️ Критичное предупреждение: при смене `roomCreationKey` все пользователи должны заново скачать плагин Outlook — старый плагин не сможет создавать временные комнаты.

По умолчанию персональные комнаты (`user{id}`) создаются с: запись разрешена и авто-старт, транскрибация разрешена и авто-старт, лобби отключено.
 Комнаты, создаваемые из плагина Outlook (при планировании встречи), также наследуют дефолтные настройки записи и транскрибации из `ConferenceDefaultUserRoomSettings`. Настройка лобби (`HasLobby`) этим путём из дефолтов не применяется.

 Начиная с v2.267 дефолты можно изменить через Admin API (требуются права God):

- `GET /api/admin/conference/default-user-room-settings` — получить текущие значения

- `POST /api/admin/conference/default-user-room-settings` — обновить значения
  Настройки хранятся в `SettingsCustom` под ключом `ConferenceDefaultUserRoomSettings` (JSON, 5 полей):
 `{
  "IsRecordingAllowed": true,
  "IsRecordingOnByDefault": true,
  "IsTranscribationAllowed": true,
  "IsTranscribationOnByDefault": true,
  "HasLobby": false
}
` Приоритет применения:

- Индивидуальные настройки комнаты (`ConferenceRooms`)

- Настройки категории (`Subcategories` — только для task-комнат)

- Дефолтные из `SettingsCustom.ConferenceDefaultUserRoomSettings`

- Системные значения по умолчанию (запись/транскрибация разрешены и авто-старт, лобби выключено)
  При невалидном JSON применяются системные значения по умолчанию, ошибка пишется в лог. Ключ создаётся автоматически при первом обращении.
 Шаблон автоматически генерируемой ссылки на ВКС-комнату — `CustomSetting ConferenceRoomUrlTemplate`. Default: `https://{origin}/{room}`, где:
 Плейсхолдер Значение `{origin}` `Services.Conference.Domain` (адрес ВКС-сервиса) `{room}` Сгенерированный ID комнаты Пример переопределения (при использовании reverse-proxy с префиксом): `https://{host}/conference/?room={room}
`
 Шаблон используется при формировании ссылок-приглашений в задачах и письмах.

Настройки IP-телефонии задаются через автоадминку в форме `telephony-settings` и хранятся в таблице `dbo.TelephonySettings`. 12 полей: URL сервера, учётные данные, параметры интеграции с телефонией. Определяет доступность звонка по клику (click-to-call) и обработку телефонных событий.
 События телефонии настраиваются через автоадминку в форме `telephony-actions-packs` и хранятся в таблице `dbo.TelephonyActionsPacks`. 5 полей: привязка событий телефонии к действиям в системе. Определяет автоматические действия при входящих/исходящих звонках.
 Настройки конкретной комнаты задаются через API `/api/conference/room/{roomId}/settings` и хранятся в таблице `dbo.ConferenceRooms`. Параметры: lobby, password, recording, transcribation. Задаются пользователем, но влияют на поведение через ту же конфигурационную таблицу.

Типичные проблемы с настройкой конференций и способы их диагностики:
 Симптом Причина Где проверить SQL-диагностика Пользователь не может войти в комнату Неверный домен/секрет Jitsi Форма `vks-service-settings` `select * from dbo.JitsiServiceSettings` История участников пустая Webhook-события не приходят или не обрабатываются `dbo.ConferenceRoomsHistory`, `dbo.ConferenceRoomsUsers` `select * from dbo.ConferenceRoomsHistory where RoomId = {roomId}; select * from dbo.ConferenceRoomsUsers where RoomId = {roomId}` Транскрипт/видео не появились в задаче Неверные QueueId/MessageFlowId или ошибка в пост-обработке Форма `vks-service-settings` → поля очередей `select QueueId, MessageFlowId from dbo.JitsiServiceSettings` Настройки комнаты не применяются Клиент использует устаревшие данные (кэш) `dbo.ConferenceRooms` `select * from dbo.ConferenceRooms where Id = {roomId}` Телефония недоступна Неверные настройки подключения Форма `telephony-settings` `select * from dbo.TelephonySettings`
