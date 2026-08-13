---
name: 1forma-user-guide
description: >-
  Руководство пользователя платформы «Первая Форма» (1Forma): задачи, категории,
  ДП, подписи/согласование, уведомления, комментарии, чат, ВКС, почта, таблица,
  канбан, календарь, ресурсы, соцсеть, файлы, Диск, отчёты, пространства, Гант,
  поиск и AI-поиск, профиль, права, оргструктура, портал, мобильное приложение,
  Анфиса. Use when the user asks how to work in 1Forma as an employee (not
  AdminSPA setup), how a business object behaves, or how to do a user-facing
  action. Official source: https://help.1forma.ru/_guides-md/user-guide-index/
---

# Руководство пользователя 1Forma

Используй этот скилл, когда вопрос про **работу сотрудника** в «Первой Форме»: поставить/выполнить задачу, согласовать, прокомментировать, найти, вложить файл, открыть отчёт, позвонить, спросить Анфису.

Полные тексты страниц — в `reference/`. Сначала открой `reference/INDEX.md`, затем нужный файл. Не выдумывай UI, статусы, типы ДП и права — сверяй с reference.

Источник индекса: [Руководство пользователя](https://help.1forma.ru/_guides-md/user-guide-index/). Снимок: 2026-08-13.

## Как отвечать

1. Определи домен (задача, ДП, подпись, чат, файл, поиск, Анфиса…).
2. Прочитай соответствующий файл из таблицы ниже.
3. Отвечай по шагам, как для пользователя SPA. Если настройка делается только в админке — скажи это и укажи скилл `1forma-admin-guide`.
4. Различай **пользовательский сценарий** и **админскую настройку**. Этот скилл — про первое.

## Карта reference

| Тема | Файл | URL |
|---|---|---|
| О руководстве | `reference/_guides-md__user-guide-index.md` | [index](https://help.1forma.ru/_guides-md/user-guide-index/) |
| Задачи | `reference/domains__tasks__business.md` | [tasks](https://help.1forma.ru/domains/tasks/business/) |
| Категории | `reference/domains__categories__business.md` | [categories](https://help.1forma.ru/domains/categories/business/) |
| ДП | `reference/domains__ext-params__business.md` | [ext-params](https://help.1forma.ru/domains/ext-params/business/) |
| Подписи | `reference/domains__signatures__business.md` | [signatures](https://help.1forma.ru/domains/signatures/business/) |
| Уведомления | `reference/domains__notifications__business.md` | [notifications](https://help.1forma.ru/domains/notifications/business/) |
| Комментарии | `reference/domains__comments__business.md` | [comments](https://help.1forma.ru/domains/comments/business/) |
| Markdown | `reference/domains__comments__text-formatting.md` | [formatting](https://help.1forma.ru/domains/comments/text-formatting/) |
| Чаты | `reference/domains__chat__business.md` | [chat](https://help.1forma.ru/domains/chat/business/) |
| ВКС | `reference/domains__conferences__business.md` | [conferences](https://help.1forma.ru/domains/conferences/business/) |
| Почта | `reference/domains__mail__business.md` | [mail](https://help.1forma.ru/domains/mail/business/) |
| Таблица | `reference/domains__grids__business.md` | [grids](https://help.1forma.ru/domains/grids/business/) |
| Канбан | `reference/domains__kanban__business.md` | [kanban](https://help.1forma.ru/domains/kanban/business/) |
| Календарь | `reference/domains__calendar__business.md` | [calendar](https://help.1forma.ru/domains/calendar/business/) |
| Ресурсы | `reference/domains__resources__business.md` | [resources](https://help.1forma.ru/domains/resources/business/) |
| Соцсеть | `reference/domains__social-network__business.md` | [social](https://help.1forma.ru/domains/social-network/business/) |
| Файлы | `reference/domains__files__business.md` | [files](https://help.1forma.ru/domains/files/business/) |
| Диск | `reference/domains__disk__business.md` | [disk](https://help.1forma.ru/domains/disk/business/) |
| Отчёты | `reference/domains__reports__business.md` | [reports](https://help.1forma.ru/domains/reports/business/) |
| Пространства | `reference/domains__spaces__business.md` | [spaces](https://help.1forma.ru/domains/spaces/business/) |
| Проекты / Гант | `reference/domains__pm__tutorial.md` | [pm](https://help.1forma.ru/domains/pm/tutorial/) |
| Поиск | `reference/domains__search__business.md` | [search](https://help.1forma.ru/domains/search/business/) |
| AI-поиск | `reference/domains__search__ai-search-user-guide.md` | [ai-search](https://help.1forma.ru/domains/search/ai-search-user-guide/) |
| Пользователи | `reference/domains__users-and-groups__business.md` | [users](https://help.1forma.ru/domains/users-and-groups/business/) |
| Интерфейс | `reference/domains__user-ui__business.md` | [ui](https://help.1forma.ru/domains/user-ui/business/) |
| Вход | `reference/domains__auth__business.md` | [auth](https://help.1forma.ru/domains/auth/business/) |
| Права | `reference/domains__permissions__business.md` | [permissions](https://help.1forma.ru/domains/permissions/business/) |
| Оргструктура | `reference/domains__org-structure__business.md` | [org](https://help.1forma.ru/domains/org-structure/business/) |
| Портал | `reference/domains__portal__business.md` | [portal](https://help.1forma.ru/domains/portal/business/) |
| Мобильное | `reference/domains__mobile__business.md` | [mobile](https://help.1forma.ru/domains/mobile/business/) |
| Анфиса | `reference/domains__ai__user-guide.md` | [ai](https://help.1forma.ru/domains/ai/user-guide/) |

## Ядро модели

**Задача** — центральная сущность. Поручение, обращение, встреча, запись справочника, чат — всё это задачи в категориях. Общие поля: категория/раздел, статусы и переходы, заказчик / ответственный / исполнители / наблюдатели, сроки, маркеры просрочки. Специфика процесса — в ДП, файлах, комментариях, подписях, связях.

**Категория** — шаблон объектов (не «подкатегория»). **Раздел** — папка (CRM, HR…). Категория принадлежит разделу. Типы объектов: задачи, справочники, ресурсы, пространства, чаты, новости, календарь.

**ДП** — настраиваемые поля категории (custom fields). Lookup = ссылка на другую задачу, Multilookup = many-to-many, Through = вычисляемое/наследуемое. Права на просмотр/редактирование поля задаёт администратор. История: «Больше → История изменений ДП».

**Подпись** в пользовательском смысле — это **согласование (approval)**, не криптоподпись. Крипто-ЭП (КриптоПро) — отдельный механизм. Акцептант — человек или любой из группы. Задача переходит дальше только после обязательных подписей. В момент подписи система снимает snapshot данных. Статические подписи — на переходе маршрута; динамические — вручную из карточки, маршрут не двигают.

## Создание задач

- «Создать» в навигации: Личная задача, Встреча, Групповой чат, Выбрать категорию.
- Кнопка создания в панели категории; `+` у категории в Избранном.
- Краткая форма из ленты — личная задача (у категории не должно быть обязательных полей).
- Создание всегда в отдельном окне. Категорию можно сменить кликом по названию (поиск по имени, ID категории, номеру задачи).
- Обязательные поля — со звёздочкой.
- Опция «Каждому исполнителю — копию» размножает задачу.

## Права (что видит пользователь)

Доступ к задаче есть, если выполняется **хотя бы один** уровень:

- право группы на категорию;
- роль в задаче (заказчик, исполнитель, подписчик, руководитель);
- смарт-доступ;
- SQL-представление;
- акцептант активной подписи;
- заместитель того, у кого есть доступ.

**Исключение:** конфиденциальная задача — только подписчики.

Три смысла слова «роль»: роль в задаче; роль БП (`BpRoles`); платформенная RBAC-роль. Уточняй, какая нужна.

Типы пользователей: **сотрудник компании**, **внешний**, **системные** (Робот 1Ф, 1CSync, Support).

## Общение

- **19 типов событий** уведомлений; у каждого флаги «в конверт» и «в ленту». Уровни: глобальные дефолты, персональные, категория. Если персональных нет — все галочки сняты.
- Комментарии: пользовательские и системные; треды, реакции, файлы, `.dash.html` как интерактивный отчёт в ленте.
- Markdown: `**жирный**`, `__курсив__` (не жирный!), `~~зачёркнутый~~`, `((подчёркнутый))`, `#12345` — ссылка на задачу, `@имя` — упоминание, `+++` — спойлер «Показать больше».
- Чаты: Personal (2), Group открытый/закрытый, Channel (broadcast), SocialNetwork. Роли: Owner > Admin > Moderator > Author. Owner нельзя передать.
- ВКС из задачи/чата/профиля. В **1F Certificate Edition (ФСТЭК)** ВКС нет целиком.

## Представления и файлы

Таблица категории, «Мои задачи», тикеры (просроченные, на сегодня…), таблицы подписей, lookup-пикер, канбан, календарь, ресурсы, ДП «Таблица».

Канбан: колонки = статусы основного маршрута (или Lookup). Drag меняет статус/значение Lookup. Персональный канбан исполнителя: лимиты 100/100/100.

Диск: «Мои файлы», «Общие файлы», «Обмен файлами», «Файлы автоматизации», файлы замещаемых/подчинённых. Файлы задач — через ДП «Файл» и комментарии; версии; онлайн-редактирование (Р7); сжатие отключает полнотекстовый поиск по файлу, если текст отдельно не извлечён.

## Поиск, портал, мобильное, Анфиса

Поиск всегда режется правами. Быстрый (шапка) и полный. Режимы: полнотекстовый, подстрока, точное, по ДП, по файлам, семантический AI.

**AI-поиск** (вкладка AI beta): режимы Задачи / Комментарии / Содержимое файлов / Файлы по имени. Понимает категории, роли, ДП, статусы, флаги (`открытые`, `просроченные`).

Портал = дашборд из групп → секций → виджетов. Главный портал — по клику на логотип. Фильтры виджета не расширяют права.

Мобильное (iOS/Android): задачи, чаты, календарь, push. Геолокация — только брендированное приложение.

**Анфиса:** в комментарии добавь `@Анфиса` единственным адресатом (или режим «как вопрос»). Ответ 15–30 с. Slash-команды: `/clear` (`/сброс`), выбор модели (`/claude`, `/kimi`…). GPT временно недоступен с 2026-05-12. 38 инструментов; работает внутри 1Формы.

## Сборки ФСТЭК

В Certificate Edition недоступны: ВКС, отчёты FastReport/Word, смарт-скрипты/смарт-действия (это уже админка, но пользователю тоже видно как «нет кнопки»).

## Чего не делать

- Не путать подпись-согласование с ЭЦП.
- Не обещать админские изменения из этого скилла.
- Не утверждать, что внешний пользователь видит всех сотрудников и группы.
- Не описывать ВКС/FastReport как доступные на ФСТЭК-площадке, пока это не подтверждено.
