# Иерархия настроек ВКС

Источник: https://help.1forma.ru/domains/conferences/settings-hierarchy/

Описание трёхуровневой системы настроек видеоконференций в 1Форме: глобальный уровень JitsiServiceSettings, уровень категории Subcategories и уровень комнаты ConferenceRooms. Документ разбирает взаимодействие уровней (как настройки наследуются и переопределяются), путь попадания настроек в JWT и room-info, и даёт практическую диагностику проблем с видимостью настроек и авто-запуском записи. Для администраторов и специалистов поддержки.

Настройки ВКС применяются по трёхуровневой иерархии (от глобального уровня к комнате):
 `┌─────────────────────────────────────────────────────────┐
│  Уровень 1: JitsiServiceSettings (глобальный)           │
│  Одна запись на площадку. Настраивается в                │
│  Администрирование → Сервисы → ВКС                      │
│                                                         │
│  Определяет:                                            │
│  • Домен, секрет, префикс комнат                        │
│  • Модератор (SmartScript/SmartFilter)                  │
│  • Security bypass                                       │
│  • Робот-пользователь (ConferenceUserId)                │
│  • Очередь + поток (для AI-саммаризации)                │
│  • CanTranscribe (глобальное вкл/выкл транскрибации)    │
│  • Группа пользователей с персональными комнатами       │
├─────────────────────────────────────────────────────────┤
│  Уровень 2: Subcategories (уровень Категории)           │
│  По записи на каждую Категорию. Настраивается в          │
│  Администрирование → Категории → [Категория] → ВКС      │
│                                                         │
│  Определяет:                                            │
│  • Кто может управлять записью/расшифровкой/лобби/паролем│
│  • Дефолтные значения для новых комнат в этой категории │
│  • Авто-запуск записи и расшифровки при старте          │
├─────────────────────────────────────────────────────────┤
│  Уровень 3: ConferenceRooms (уровень комнаты)           │
│  По записи на каждую комнату. Настраивается              │
│  пользователем через интерфейс (шестерёнка в шапке)      │
│                                                         │
│  Определяет:                                            │
│  • Конкретные значения для этой комнаты                  │
│  • HasLobby, Password, Recording, Transcription         │
└─────────────────────────────────────────────────────────┘
`

Глобальная конфигурация ВКС-сервиса. Одна запись на площадку.
 Колонка Тип Описание `ServiceId` int FK Ссылка на ServicesSettings `Domain` nchar(50) Домен Jitsi (`<видео-сервер>`) `Secret` varchar(100) JWT_APP_SECRET `ConferenceUserId` int? FK Робот для комментариев `QueueId` int? FK Очередь для AI-обработки `MessageFlowId` int? FK Поток для саммаризации `CanTranscribe` bit Глобальное разрешение транскрибации `UserRoomPrefix` — Префикс комнат (в ExtInfo) `IsModeratorSmartFilterId` int? SmartFilter модератора (устаревший) `CheckIsModeratorSmartScriptId` int? SmartScript модератора `CheckSecurityBypassSmartFilterId` int? SmartFilter bypass lobby `CheckSecurityBypassSmartScriptId` int? SmartScript bypass lobby `ExtInfo` (JSON): `externalModuleKey`, `roomCreationKey`, `recordHost`, `recordsApiKey`, `internalModuleKey`.

Добавлены в v2.266. 9 колонок в таблице `Subcategories`.
 Уровни разрешений (ConfPermLevel):
 Id Значение Описание 1 `ByChatRights` По правам чата — заказчик, исполнитель, подписчик 2 `AdminOnly` Только администратор категории 3 `Forbidden` Запрещено всем Колонки:
 Колонка Тип Default Описание `ConfRecordingPermLevelId` tinyint FK 2 (AdminOnly) Кто управляет записью `ConfTranscriptionPermLevelId` tinyint FK 2 (AdminOnly) Кто управляет расшифровкой `ConfLobbyPermLevelId` tinyint FK 2 (AdminOnly) Кто управляет лобби `ConfPasswordPermLevelId` tinyint FK 2 (AdminOnly) Кто управляет паролем `IsConfRecordingEnabledByDefault` bit 1 Разрешить запись для новых комнат `IsConfTranscriptionEnabledByDefault` bit 1 Разрешить расшифровку для новых комнат `IsConfLobbyEnabledByDefault` bit 1 Лобби по умолчанию для новых комнат `IsConfRecordingOnStart` bit 1 Авто-запуск записи при старте `IsConfTranscriptionOnStart` bit 1 Авто-запуск расшифровки при старте Дефолты для чатов: категории P2P и групповых чатов (Settings.ChatSubcatID, GroupChatSubCatID) получают `ByChatRights` (1) вместо `AdminOnly` (2).
 Видимость настроек в интерфейсе зависит от роли пользователя:
 Пользователь Условие видимости опции Администратор категории или площадки Опция видна при любом значении кроме `Forbidden` Заказчик / владелец чата / модератор Опция видна только если `ByChatRights` Меню настроек ВКС скрывается полностью, если все 4 группы = `Forbidden`.

Создаётся при первом изменении настроек конкретной комнаты пользователем.

 Колонка Тип Default Описание `RoomKey` varchar(32) — `user{id}` / `task{id}` / random `HasLobby` bit 0 Лобби включено `Password` nvarchar(100) null Пароль (зашифрован) `IsRecordingAllowed` bit 1 Запись разрешена `IsRecordingOnByDefault` bit 1 Запись при старте `IsTranscribationAllowed` bit 1 Расшифровка разрешена `IsTranscribationOnByDefault` bit 1 Расшифровка при старте Кто может менять:
 Тип комнаты Кто может Персональная (`user{id}`) Только владелец Задачная (`task{id}`) Только заказчик задачи

Записи и расшифровка:
 `Пользователь нажимает «Запись»
  │
  ├─ Глобально разрешено? (CanTranscribe для расшифровки)
  │    └─ Нет → запрещено
  │
  ├─ Категория разрешает? (ConfRecordingPermLevelId != Forbidden)
  │    └─ Нет → запрещено
  │    └─ AdminOnly → только администратор
  │    └─ ByChatRights → любой участник
  │
  └─ Комната разрешает? (IsRecordingAllowed)
       └─ Нет → запрещено
       └─ Да → запись
` Лобби:
 `Новая комната
  │
  ├─ Категория: IsConfLobbyEnabledByDefault = 1?
  │    └─ Да → лобби включено по умолчанию
  │    └─ Нет → лобби выключено
  │
  └─ Пользователь меняет настройку (если ConfLobbyPermLevelId != Forbidden)
       └─ ConferenceRooms.HasLobby = true/false
` Авто-запуск при старте:
 `Конференция начинается
  │
  ├─ Категория: IsConfRecordingOnStart = 1 И ConfRecordingPermLevelId != Forbidden?
  │    └─ Да → клиент конференции запускает запись автоматически
  │
  └─ Комната: IsRecordingOnByDefault?
       └─ Значение передаётся в room-info, клиент решает
`

Итоговые значения настроек передаются клиенту конференции двумя путями:
 `GET /api/conference/room-token → JWT:
  • moderator: true/false (из SmartScript)
  • context.user.security_bypass: true/false
  • context.room.lobby: HasLobby
  • context.room.password: Password (зашифрован)

GET /api/conference/room-info → JSON:
  • IsPrivate, HasLobby, Password
  • IsRecordingAllowed, IsRecordingOnByDefault
  • IsTranscribationAllowed, IsTranscribationOnByDefault
` Клиент конференции использует room-info для решения об авто-запуске записи/расшифровки.

Пользователь не видит настройки ВКС:

- Проверить `ConfRecordingPermLevelId`, `ConfTranscriptionPermLevelId`, `ConfLobbyPermLevelId`, `ConfPasswordPermLevelId` в `Subcategories` для категории задачи

- Если все = 3 (Forbidden) → меню скрыто

- Проверить роль пользователя: для опций уровня AdminOnly требуется администратор категории
  Запись не запускается автоматически:

- `Subcategories.IsConfRecordingOnStart` = 1?

- `Subcategories.ConfRecordingPermLevelId` != 3 (Forbidden)?

- `ConferenceRooms.IsRecordingAllowed` = 1?

- `ConferenceRooms.IsRecordingOnByDefault` = 1?

- Сервис записи (Jibri) доступен?
  SQL для проверки:
 `-- Настройки категории
SELECT SubcatID, ConfRecordingPermLevelId, ConfTranscriptionPermLevelId,
       ConfLobbyPermLevelId, ConfPasswordPermLevelId,
       IsConfRecordingEnabledByDefault, IsConfTranscriptionEnabledByDefault,
       IsConfLobbyEnabledByDefault, IsConfRecordingOnStart, IsConfTranscriptionOnStart
FROM Subcategories WHERE SubcatID = @subcatId;

-- Настройки комнаты
SELECT * FROM ConferenceRooms WHERE RoomKey = @roomKey;

-- Глобальные
SELECT CanTranscribe FROM JitsiServiceSettings;
` Связанные документы:

- Видеоконференции — администрирование — формы настроек, API и диагностика
