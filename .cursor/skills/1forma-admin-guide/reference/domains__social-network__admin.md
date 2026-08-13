# Социальная сеть — Администрирование

Источник: https://help.1forma.ru/domains/social-network/admin/

Руководство по администрированию корпоративной социальной сети 1Формы: сообщества, публикации, лента, статьи. Собственных Admin API контроллеров у домена нет — настройки распределены по смежным доменам через автоадминку. Документ для администраторов площадки: ключевые SettingsCustom, типы категорий, видимость ленты, служебные категории и типичные ошибки.

Домен social-network охватывает настройку корпоративной социальной сети: сообщества, публикации, лента, статьи. Собственных Admin API контроллеров у домена нет — настройки распределены по смежным доменам. Администрирование использует:

- Настройки через смежные домены — `custom-settings` (конфигурация соцсети), `subcategories` (типы категорий), `workplaces` (видимость блока), `users-new-default-settings` (дефолты)
  Автоадминка (dbadmin) — через смежные домены
 Alias формы Название Таблица БД Полей Секций Deps Папка `custom-settings` Кастомные настройки dbo.SettingsCustom 6 1 0 Системные настройки `workplaces` Рабочие места группы dbo.UserGroupSettings 45 5 0 Пользовательский интерфейс `users-new-default-settings` Настройки по умолчанию dbo.UsersNewDefaultSettings 38 5 0 (корень)



Где настраивается: автоадминка -> форма `custom-settings` → запись `Key = socialNetworksSettings` Таблица БД: `dbo.SettingsCustom`
 JSON-значение определяет: - `RootCategoryId` — корневой раздел - `GroupsForPublicationsId` — категория для открытых публикаций - `GroupsForClosedPublicationsId` — категория для закрытых публикаций - `PersonalPublicationsSubcatId` — категория для личных публикаций - `ArticlesSubcatId` — категория для статей (должна иметь `IsWikiSubcat = 1`) - `HeaderStyle` — стиль шапки группы соцсети (`0` или `1`) - `ExtParamCoverId` — ДП для обложки
 ДП обложки привязывается ко всем категориям типа `Chats`, кроме категории личных чатов (`Settings.ChatSubcatId`) — с 2.268.x. На существующих инсталляциях single-run миграция `UnbindEpCoverFromPersonalChatsCategory` отвязывает уже привязанный ДП от категории личных чатов, не трогая значения обложек в групповых чатах, каналах и соцсетях.
 Эффект в runtime: `SocialNetworkService` читает эти настройки для маршрутизации контента по служебным категориям.

Тип категории для соцсети (Subcategories)
 Где настраивается: автоадминка -> FormsGenerator `subcategories` Таблица БД: `dbo.Subcategories`
 Поле Что контролирует `SubcatTypeID` = «Группы соц. сети» Категория работает как сообщество/публикации `SubcatTypeID` = «Пространство» + `IsWikiSubcat = 1` Категория доступна как пространство для статей `OneFMainVisibilityMode` Попадание записей в ленты и списки `AllowGroupSubcription` Подписка участников и видимость контента Видимость блока «Лента корп. сети» (UserGroupSettings)
 Где настраивается: автоадминка -> форма `workplaces` Таблица БД: `dbo.UserGroupSettings`
 Поле Что контролирует `CorporateNetworkFeedVisibleType` Показывает/скрывает блок в левом меню Дефолт для новых пользователей (UsersNewDefaultSettings)
 Где настраивается: автоадминка -> форма `users-new-default-settings` Таблица БД: `dbo.UsersNewDefaultSettings`
 Поле Что контролирует `IsCorporateNetworkFeedVisible` Дефолтная видимость ленты соцсети для новых пользователей Портальный блок соцсети
 Где настраивается: Admin API порталов -> блок «Лента социальной сети» Таблицы БД: `dbo.PortalGrid*`
 Отображение ленты соцсети на портальной странице.

Раздел и служебные категории социальной сети не требуется создавать вручную. Их ID задаются в `socialNetworksSettings`, а категории должны находиться в системном разделе социальной сети.
 Параметр Служебная категория Тип категории Права группы «Все пользователи» по умолчанию `GroupsForPublicationsId` Группы для публикаций «Группы соц.сетей» `Исполнять`, `Просмотр всех задач`, `Создавать задачи`, `Менять заказчика` `GroupsForClosedPublicationsId` Группы для закрытых публикаций «Группы соц.сетей» `Создавать задачи` `PersonalPublicationsSubcatId` Личные публикации «Группы соц.сетей» `Исполнять`, `Просмотр всех задач`, `Создавать задачи`, `Менять заказчика` `ArticlesSubcatId` Статьи «Пространство» Права настраиваются как для пространства; категория используется при создании статьи из публикации ⚠️ При смене типа сообщества «Открытое»/«Закрытое» задача сообщества автоматически переносится между категориями `GroupsForPublicationsId` и `GroupsForClosedPublicationsId`. Права для группы «Все пользователи» переназначаются в зависимости от целевой категории.
 Для личных публикаций после первого поста пользователя в категории `PersonalPublicationsSubcatId` создаётся агрегатор-задача `Публикации. {Фамилия Имя Отчество}` от лица пользователя. Последующие посты пользователя хранятся в этой задаче как сообщения.

Ниже приведён полный JSON-объект настройки социальной сети с описанием каждого поля.
 `{
  "RootCategoryId": 4441,
  "PersonalPublicationsSubcatId": 62720,
  "HeaderStyle": 1,
  "GroupsForPublicationsId": 62730,
  "GroupsForClosedPublicationsId": 62900,
  "ArticlesSubcatId": 62830,
  "ExtParams": {
    "ExtParamCoverId": 100020
  }
}
` Параметр Назначение `RootCategoryId` ID системного раздела социальной сети `PersonalPublicationsSubcatId` ID категории личных публикаций `HeaderStyle` Необязательный стиль шапки группы: `0` — аватар группы и автор после контента, `1` — имя и аватар автора в шапке `GroupsForPublicationsId` ID категории открытых сообществ `GroupsForClosedPublicationsId` ID категории закрытых сообществ `ArticlesSubcatId` ID категории статей с типом «Пространство» `ExtParams.ExtParamCoverId` ID файлового ДП для обложки Подробные пользовательские сценарии публикаций, сообществ и `HeaderStyle` описаны в business.md; в admin.md фиксируются только параметры настройки и их эффект на маршрутизацию контента.

В таблице перечислены наиболее частые симптомы неправильной настройки социальной сети, их причины и способы диагностики.
 Симптом Причина Где проверить SQL-диагностика Публикации создаются «не туда» Неверные ID категорий в `socialNetworksSettings` Форма `custom-settings` `select [Value] from dbo.SettingsCustom where [Key] = 'socialNetworksSettings'` Не создаётся статья из публикации `ArticlesSubcatId` указывает на категорию без `IsWikiSubcat` `dbo.Subcategories` `select Id, SubcatTypeID, IsWikiSubcat from dbo.Subcategories where Id = {articlesSubcatId}` Не виден блок соцсети в меню `CorporateNetworkFeedVisibleType` выключен Форма `workplaces` `select GroupId, CorporateNetworkFeedVisibleType from dbo.UserGroupSettings` Лента соцсети пустая в портале Некорректный блок или нет доступа к сообществам Блок портала Проверить конфигурацию блока и доступ к категориям Тип сообщества не распознаётся Категория не имеет правильный `SubcatTypeID` `dbo.Subcategories` `select Id, SubcatTypeID from dbo.Subcategories where Id = {subcatId}`
- `business.md` — бизнес-логика (сообщества, публикации, статьи)

- `../spaces/admin.md` — пространства (пересечение по категориям с `IsWikiSubcat`)

- `../system/admin.md` — системные настройки (`custom-settings`)

- `../categories/admin.md` — настройка категорий (типы, видимость)

- `../../platform/backend/admin-architecture.md` — общая архитектура администрирования

- `../../reference/database/dbadmin-forms-map.md` — карта всех форм автоадминки
