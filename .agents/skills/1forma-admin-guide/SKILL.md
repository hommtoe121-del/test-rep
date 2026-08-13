---
name: 1forma-admin-guide
description: >-
  Руководство администратора «Первой Формы» (AdminSPA / автоадминка): пользователи
  и группы, AD/SSO, права, перевоплощение, оргструктура, категории и маршруты,
  форма задачи MTF/NTF, ДП и типы ДП, смарт-правила/пакеты/скрипты (C# Roslyn,
  Python, Lua, Jint), уведомления, почта, файлы, отчёты FastReport, порталы,
  канбан, календарь Exchange/CalDAV, интеграции 1С/ЭДО, SettingsCustom, ЭП
  КриптоПро. Use when configuring 1Forma, diagnosing AdminSPA, smart packs,
  routes, extra params, or permissions. Official source:
  https://help.1forma.ru/_guides-md/admin-guide-index/
---

# Руководство администратора 1Forma

Используй этот скилл для **настройки и диагностики** «Первой Формы»: AdminSPA, автоадминка (`dbadmin`), категории, ДП, маршруты, смарт, права, интеграции.

Полные тексты — в `reference/` (134 страницы). Начни с `reference/INDEX.md`, затем открой файл домена. Не угадывай alias форм, имена таблиц, EventID и типы ДП.

Источник индекса: [Руководство администратора](https://help.1forma.ru/_guides-md/admin-guide-index/). Снимок: 2026-08-13.

## Правило работы в AdminSPA

По умолчанию в админке **только смотри и фиксируй**. Не нажимай Сохранить / Применить / Удалить / Добавить, не меняй флаги, права, ДП, смарт, маршруты, группы — пока пользователь в этом чате **явно** не попросил внести конкретное изменение.

Запомнить настройку ≠ изменить её. Если для просмотра нужно что-то включить или сохранить — сначала спроси.

## Как отвечать

1. Найди домен в карте ниже и прочитай reference (admin + support-guide / academy-patterns при разборе инцидента).
2. Называй объекты канонически: **Раздел** vs **Категория**, **ДП**, **пакет действий**, **смарт-правило**, **НТФ/МТФ**.
3. Права выдаются **группам**, не пользователям напрямую и не группе Users («Все пользователи»).
4. Для стека/установки — скилл `1forma-tech-stack`. Для «как работает пользователь» — `1forma-user-guide`.

## Критичная терминология

| Как говорят | Что это в 1Форме | Таблица БД |
|---|---|---|
| Раздел (CRM, HR) | папка / namespace | `dbo.Categories` |
| Категория (Клиенты, Договоры) | шаблон задач / entity type | `dbo.Subcategories` |
| Подкатегория | **не использовать** — путает с вложенностью | — |
| ДП | custom field | `dbo.ExtParams` + таблицы значений |
| НТФ | форма **создания** задачи | — |
| МТФ | карточка **открытой** задачи | — |
| Подпись | approval на переходе, не ЭЦП | `dbo.Signatures` |
| ЭЦП / КриптоПро | криптоподпись | см. `cryptopro-ca` |
| Роль | (1) в задаче (2) БП `BpRoles` (3) RBAC `Roles` | уточняй |

Инверсия имён в БД обязательна: `Categories` = разделы, `Subcategories` = категории.

## Карта reference (частое)

Полный список — `reference/INDEX.md`. Ниже якоря для типовых запросов.

### Пользователи, вход, права

| Тема | Файл |
|---|---|
| Пользователи и группы | `domains__users-and-groups__admin.md` |
| Паттерны групп/оргструктуры | `domains__users-and-groups__academy-patterns.md` |
| Видимость пользователей | `domains__users-and-groups__groups-visibility-auto-creation.md` |
| Авторизация | `domains__auth__admin.md` |
| AD Sync | `domains__auth__ad-sync-technical-reference.md` |
| AD/SSO проблемы | `domains__auth__support-guide-ad-sso.md` |
| Нет доступа к форме | `domains__auth__runbook-form-access-auth.md` |
| Права на задачи | `domains__permissions__admin.md` |
| Паттерны прав | `domains__permissions__academy-patterns.md` |
| Перевоплощение | `domains__permissions__impersonation.md` |
| Оргструктура | `domains__org-structure__admin.md`, `domains__org-structure__sync.md` |

### Категории, форма, подписи, ДП

| Тема | Файл |
|---|---|
| Категории (319 полей) | `domains__categories__admin.md` |
| Справочник переходов | `domains__categories__transition-settings-reference.md` |
| Системные категории | `domains__categories__system-categories-reference.md` |
| Диагностика маршрута | `domains__categories__routing-troubleshooting.md` |
| Задачи / маршруты | `domains__tasks__admin.md`, `domains__tasks__support-guide-routes.md` |
| Форма MTF/NTF | `domains__task-forms__admin.md` |
| Блоки формы | `domains__task-forms__blocks-reference.md` |
| Миграция JS-вставок | `domains__task-forms__mtf-ntf-differences.md` |
| Подписи | `domains__signatures__admin.md` |
| КриптоПро УЦ 2.0 | `domains__signatures__cryptopro-ca.md` |
| Настройка ДП | `domains__ext-params__admin.md` |
| Справочник типов ДП | `domains__ext-params__types-reference.md` |
| ДП Файл / Lookup / Users / Таблица | `domains__ext-params__file__settings-reference.md`, `lookup__settings-reference.md`, `select-users__settings-reference.md`, `table__settings-reference.md` |
| Права на ДП | `domains__ext-params__permissions-model.md` |
| Сквозные ДП | `domains__ext-params__through.md` |
| JS→смарт видимость | `domains__ext-params__faq-visibility-migration-js-to-smart.md` |

### Смарт

| Тема | Файл |
|---|---|
| Правила, пакеты, скрипты | `domains__smart-actions__admin.md` |
| Справочник действий | `domains__smart-actions__actions-reference.md` |
| Ловушки пакетов | `domains__smart-actions__action-package-pitfalls.md` |
| Переменные | `domains__smart-actions__variables-reference.md` |
| Смарт-фильтры / синтаксис | `domains__smart-filters__admin.md` |
| Чтение сущностей | `domains__smart-filters__read-data.md` |
| Jint / C# / Lua / Python / NLP | `js-scripting-jint.md`, `js-jint-patterns.md`, `csharp-scripting-roslyn.md`, `lua-pg-guide.md`, `python-scripting.md`, `nlp-api.md` |

### Прочее администрирование

Уведомления, комментарии, чат, ВКС, почта, файлы, Диск, отчёты, порталы, канбан, гриды, календарь (EWS/CalDAV), ресурсы, соцсеть, пространства, мобильное, поиск, локализация, интеграции (1С, Exchange, ЭДО, PT Sandbox, секреты), публикации, SettingsCustom, housekeeping БД, ER, миграция конфигурации, форм-контролы, телефония — все файлы в `reference/INDEX.md`.

## Модель настройки

Три контура админки:

1. **Автоадминка (dbadmin)** — CRUD по alias форм (`subcategories`, `authentication-providers`, `smart-access`…).
2. **EntityEditor** — JSON-схемы (роли, сводные категории).
3. **Admin API** — операции, где формы мало (`/api/admin/subcategories`, `/api/admin/smart/packs`, …).

Категория: Администрирование → Категории. Создание: раздел + имя + тип (Задачи, Справочник, Ресурсы, Пространство, Чаты, Новости, Календарь…). Копирование переносит настройки, ДП, смарты и статические подписи.

Блоки настроек категории: Основные, Настройки, Доступ, ДП, Маршрут, Смарты, Формы, Уведомления. Поиск по 319 полям — строка в шапке формы категории.

## Права

- Права на категорию — **только группам**. Для одного человека заведи индивидуальную группу.
- Не выдавай права группе Users / «Все пользователи» — это тормозит систему.
- Сразу выдавай Administrators, иначе сам не увидишь результат.
- Смарт-доступ: выражение возвращает UserID; видимость проверяется только по действию «Просмотр задачи».
- Конфиденциальные задачи — только подписчики, остальные уровни не действуют.
- Перевоплощение ≠ замещение. Замещение передаёт права; перевоплощение — новая сессия. Админам можно, если нет `DENYIMPERSONATION`. Не-админ не перевоплощается в администратора.

AD Sync **односторонний**: AD → 1Форма. На PG `ADSyncJob` может не работать (известная проблема).

## ДП

Тип ДП **нельзя сменить** после создания. Неверный тип → удалить и создать заново (удаление невозможно, если ДП уже используется в задачах).

Типы и хранение (кратко):

| Тип | ExtParamType | Где значение |
|---|---|---|
| Текст | `Text` | `ExtParamValues.ExtParamValue` |
| Большой текст | `TextArea` / `TextareaWOFormat` | `ExtParamValue` |
| Число / Деньги / Дата | `NumericValue` / `Money` / `Date`/`Datetime` | `DecimalValue` / `MoneyValue` / `DateTimeValue` |
| Флажок | `Checkbox` | `BoolValue` |
| Select / Combobox / Tree | `Select` / `Combobox` / `Tree` | `ExtParamValue` + `DataSourceItemID` |
| Lookup / Multilookup | `LookUpField` / `MultiSlctSubcatTasks` | `ExtParamValueSelectedTasks` |
| Выбор пользователей | `SelectUsers` | `ExtParamSelectUsersValues` (+ groups/org) |
| Таблица | `Table` | `ExtParamTableValues` |
| Файл | `File` | `FileStorageFileToExtParamLinks` |
| Сквозной | `Through` | вычисляется |

Видимость/обязательность ДП лучше смарт-фильтрами, не JS-вставками (есть FAQ миграции).

При правке настроек ДП через API: не выдумывай частичный PATCH. Типовой безопасный паттерн в доке: GET полного объекта → точечно изменить → POST полный объект. Не затирай чужие поля.

## Смарт

Цепочка: **событие (EventID + ParameterValue) → условие (смарт-выражение) → пакет действий**. Действия идут по порядку; результат можно положить в переменную для следующих шагов. Циклический пакет — по объектам выборки.

Где: `/administration/smart-packs`, правило `/administration/smart-rule/:id`. API: `/api/admin/smart/packs`, `packs-on-events`, `scripts/editor`.

`SmartExpressions`: `IsFilter=1` — фильтр; `IsFilter=0` — правило, нужен `EventID`. `SubcatID=NULL` — глобальное.

Движки скриптов: **C# / Roslyn**, **Python** (внешний исполнитель), **Lua** (осторожно с PG), **JS / Jint**. HTTP/SQL/API — из справочника действий.

В **Certificate Edition (ФСТЭК)** смарт-скрипты и смарт-действия **исключены**.

Редактор скрипта: GET `/api/admin/smart/scripts/editor` → меняй **только код** → POST обратно, остальные поля не трогать.

Пакеты: типичные ошибки — в `action-package-pitfalls.md`. Не путай триггер правила (смена ДП / переход) с ручным селектором в карточке.

## Форма задачи и JS/CSS

Вставки: настройки категории, блок JS/CSS. Методы JS API — там же и в `task-forms/admin`. Портальные скрипты — `portal/admin`. Миграция старой карточки → новой: `mtf-ntf-differences.md`. Форм-контролы имеют матрицу совместимости MTF/NTF.

## Интеграции и система

- 1С: XML-теги, маппинг сущностей, runbook подключения — `integrations/*` и `support-guide-1c.md`.
- Секреты — `integrations/secrets.md`, не клади пароли в смарт-текст.
- SettingsCustom — тонкие флаги поведения, `system/settings-custom.md`.
- Очистка логов БД — `db-housekeeping.md` (только безопасные таблицы из доки).
- Deep links SPA — `user-ui/admin.md`.
- Перенос конфигурации между площадками — `domains__migration.md`.

## ФСТЭК / Certificate Edition

Исключены: смарт, FastReport/Word-отчёты, ВКС (1f-meet/Jitsi). Не предлагай их на такой сборке.

## Чего не делать

- Не сохранять настройки «чтобы посмотреть».
- Не выдавать права группе «Все пользователи».
- Не менять тип существующего ДП.
- Не писать частичный JSON настроек, затирая соседние поля.
- Не путать пакет действий со смарт-правилом (событие → пакет).
- Не выдумывать endpoint Admin API — бери из reference.
