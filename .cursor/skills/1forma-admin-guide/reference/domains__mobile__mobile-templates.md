# Мобильные шаблоны

Источник: https://help.1forma.ru/domains/mobile/mobile-templates/

Справочник по двум системам шаблонизации в мобильном приложении 1Формы:

- Шаблоны задач — как задача выглядит в карточке, форме создания и списке.

- Шаблоны блоков — как настраиваются плитки и блоки рабочего стола (dashboard), меню и ленты задач.



Шаблоны описывают, как задача отображается в мобильном приложении в трёх контекстах:
 Контекст Описание Cell Краткое представление задачи в списке задач. Ячейка списка. MTF (Main Task Form) Карточка существующей задачи при просмотре/редактировании. NTF (New Task Form) Форма создания новой задачи. Тип шаблона:

- static — создаётся разработчиками, поставляется вместе с приложением.

- dynamic — создаётся и настраивается администраторами через JSON.
  Шаблоны назначаются в настройках категории: вкладка «Вид» → кнопка «Шаблонизация».
 Мобильные шаблоны хранятся в конфигурационном файле платформы.

Параметры карточки шаблона:
 Параметр Описание ID Идентификатор шаблона Имя Отображаемое имя Описание Описание назначения Контекст Cell / MTF / NTF Тип static / dynamic Язык Язык интерфейса МП, которому соответствует шаблон. Если не задан — шаблон используется как базовый для всех языков. Категория ID категории (для статических необязательно) Шаблон JSON-содержимое шаблона

Для каждого шаблона доступна кнопка «Локализация». Локализация — альтернативная версия содержимого JSON для конкретного языка МП.
 Поля формы локализации: - Шаблон — заполняется автоматически - Язык — язык интерфейса МП - Содержимое — JSON-содержимое для данного языка

Трёхуровневая JSON-структура: `sections → blocks → elements`. Раздел `settings` необязателен — может быть как для всего шаблона, так и для отдельных элементов.
 `{
  "settings": {},
  "sections": [
    {
      "name": "...",
      "blocks": [
        {
          "name": "...",
          "settings": {},
          "elements": [
            {
              "name": "...",
              "type": "...",
              "dataKey": "...",
              "viewMode": "...",
              "settings": {}
            }
          ]
        }
      ]
    }
  ]
}
` В шаблоне MTF: `sections` = вертикальные вкладки, `blocks` = горизонтальные блоки, `elements` = элементы. В настоящее время блок в секции может быть только один.
 В шаблоне NTF с несколькими секциями: секции отображаются последовательно (отдельные экраны), имитируя пошаговое создание задачи. На каждом экране — кнопка «Далее», на последнем — «Готово».

Ключи секции:
 Ключ Описание `name` Имя секции `blocks` Массив блоков в секции (рекомендуется один блок) `isDefault` При открытии задачи эта секция активна по умолчанию `isPushOpenDefault` При открытии через push эта секция активна по умолчанию `alias` Уникальный идентификатор для настройки смарт-пуш проваливания `settings` Настройки секции

Ключи блока:
 Ключ Описание `name` Имя блока `elements` Массив элементов `settings` Необязательный. Настройки блока. Пример `settings` для блока:
 `"settings": {
  "titleColor": "#000000",
  "titleSize": 18,
  "titleFontWeight": "medium",
  "textItalic": true,
  "align": "center"
}
`

Ключи элемента шаблона:
 Ключ Описание `name` Имя элемента `type` Тип элемента (см. ниже) `dataKey` Ключ данных `viewMode` Режим отображения (см. ниже) `defaultValue` Значение по умолчанию. Для ДП Lookup: `{"taskId": 123456, "taskText": "Название"}` `placeholder` Пояснение при пустом значении `readOnly` `true`/`false` — только для чтения `alias` `image`, `leftimage`, `rightimage`, `title`, `subtitle`, `value` `hint` Подсказка — справа появляется кнопка «i» `text` Текст вместо значения ДП `settings` Настройки элемента `isHidden` `true`/`false` — скрыть ДП При отрисовке Cell-шаблона для owner, performers, ДП — перед заголовком рисуется аватар высотой `height` (по умолчанию 32).

Допустимые значения ключа `type`:
 Значение Описание `allExtParams` Все ДП категории. Рекомендуется на всю секцию как комплексный параметр. `mainParam` Основные параметры задачи `extparam` Дополнительный параметр (ДП)

Допустимые значения `dataKey` для основных параметров задачи:
 dataKey Поле задачи `category` Категория `taskNum`, `taskId` Номер задачи `taskText` Текст задачи `owner` Заказчик `createdDate` Дата создания `orderedDate` Срок `state` Статус `priority` Приоритет `performers` Исполнители `buttons` Кнопки переходов `usedTasks` Блок «Используется» `stateAction` Комбо-элемент статус + кнопки переходов (синяя кнопка, по нажатию — меню переходов и смарт-кнопок) `files` Файлы `subscribers` Подписчики `tags` Теги `hierarchy` Иерархия Для ДП (`type: extparam`) в `dataKey` указывается числовой ID ДП.

Допустимые значения ключа `viewMode`:
 Значение Описание `normal` Элемент занимает часть блока `valueOnly` Часть блока, отображается только значение (без названия) `full` Занимает весь блок и всю вкладку (например, лента комментариев) `title` Заголовок ячейки. Только в Cell-шаблонах, для одного поля-заголовка. `compact` Заголовок рядом со значением `none` Не показывать `imagePreview` Картинки в виде карусели `checklist` Чеклист `stars` Звёзды оценки (0–5) `button` Только для Cell: статус в виде кнопки `docScan` Таблица с фотосканированием (набор секций + фото-галерея) `round` Скруглённая картинка (аватар) `icon` Название значка. Только для Android. `table` Табличный вид ДП LookUp (удобен при малом числе значений)

Значения по умолчанию: - `fontSize` — 17px - `textColor` — grey - `titleSize` — 17px - `titleColor` — black - `titleFontWeight` — regular - `textItalic` — false - `bgColor` — пусто - `align` — left

Полный перечень ключей `settings` и их применимость по контекстам:
 Ключ Описание Шаблон `afterTaskCreationBehaviour` Поведение после создания задачи. `closeTask` — сразу закрывает карточку. только NTF `alwaysShowTabMenu` `true/false`. Если true — меню вкладок всегда отображается. только MTF `usedBlockTables` JSON со списком настроек блоков «Используется». только MTF `align` `left` / `right` / `center`. Выравнивание текста. все `allowCommentToAll` `true/false`. Разрешить отправку комментариев без адресатов (всем подписчикам). только MTF `commentTypes` Типы комментариев в ленте. Массив: `[3, 15]`. только MTF `forbidOfflineChange` Блокировка ячейки при отсутствии сети (работает только для `buttons`). все `forbidOpen` `true/false`. Блокирование перехода в карточку задачи из списка. только Cell `hideInfoButton` `true/false`. Скрыть иконку «i» у ДП Lookup. все `hideSubscribers` `true/false`. При написании комментария не показывать подписчиков. только MTF `cameraInput` Заполнять ДП файл с камеры. `"isContinuous": true` — непрерывная съёмка. NTF, MTF `fontSize` Размер шрифта текста. все `textColor` Цвет текста. все `titleSize` Размер шрифта заголовка. все `titleColor` Цвет заголовка. все `titleFontWeight` Жирность текста. все `textItalic` Курсив. все `bgColor` Фон текста. все `columns` Для ДП Таблица. `"compact":[...]` — колонки на MTF, `"full":[...]` — при проваливании в строку. NTF, MTF `links` Для текстовых ДП. Какие типы ссылок выделять: `["http", "phone", "email"]`. NTF, MTF `showClearButton` `true/false`. Показать крестик очистки в поле ДП. все `showClearButtonWithConfirm` Как `showClearButton` + модальное подтверждение. все `allowCodeScanInput` `true/false`. Справа у поля — иконка QR-кода для сканирования. все `activeOnShowSection` `true/false`. Автофокус в текстовом ДП. При `allowCodeScanInput=true` — автооткрытие сканера в NTF. все `maxLineCount` Максимальное количество строк таблицы. все `width` Ширина в относительных единицах. все `verticalAlign` Вертикальное выравнивание: `top`, `center`, `bottom`. все `showTaskFilter` Показывать/скрывать фильтр задач. все `allowSorting` `true` по умолчанию. Разрешить/запретить сортировку. все `createButton` Настройка текста кнопки создания задачи. `{"title": "Заказать пропуск"}`. только NTF `header` Заголовок экрана создания задачи (title, subtitle, bgImage с gradient и url). только NTF `subtitle` Текст над NTF задачи. только NTF `bgImage` Фоновая картинка над NTF (gradient, url, alpha, settings.ratio). только NTF `regExp` Регулярное выражение для проверки ввода в поле. NTF, MTF `capsInput` Все вводимые символы — заглавные. NTF, MTF `showLookupCreateButton` Показывает кнопку создания задачи в категории ДП Lookup (если есть права). NTF, MTF `onChangeCopyTo` Кликабельная таблица из ДП Lookup с копированием значения в другой ДП. только NTF `prefix` Строка-префикс перед вводимым текстом в главном или доп.параметре. все `afterLastSignBehaviour` Поведение после последней подписи. `"none"` — ничего. MTF `showLocation` Показать объект на карте (для ДП геолокации). MTF `tableId` ID таблицы БИ для секции. MTF `closeAfterSelect` Для текстовых ДП с `suggestType`: после выбора из подсказок сразу подставляет значение и закрывает модальное окно. NTF, MTF `showAddButtonInNavigationBar` Отображать иконку «+» с addButton в верхнем тулбаре. только Cell

В шаблоне MTF доступно меню со списком действий. Настраивается через `settings.options`. Без секции `options` — доступны все действия.
 `"settings": {
  "options": {
    "signatures": true,
    "resources": true,
    "createSubtask": true,
    "createLinkedTask": true,
    "createLinkedEvent": true,
    "pin": true,
    "favorites": true,
    "mute": true,
    "videoConference": true,
    "share": true,
    "shareToChat": true
  }
}
`

Полный перечень ключей меню действий:
 Ключ Описание `avatar` Изменение аватара задачи `files` Вложения `resources` Ресурсы `signatures` Подписи `createSubtask` Создать подзадачу `createLinkedTask` Создать связанную задачу `createLinkedEvent` Создать новую задачу `videoConference` Видеоконференция (комната ВКС задачи) `shareToChat` Переслать задачу в чат внутри МП `share` Поделиться вне МП `mute` Переключить «бесшумный» режим `changeTaskText` Изменение текста задачи `pin` Закрепить/открепить в списке чатов `favorites` Добавить/удалить из избранного

В карточке задачи: avatar, resources, signatures, createSubtask, createLinkedTask, createLinkedEvent, videoConference, shareToChat, share, mute, pin, favorites.
 В чате: files (если есть), videoConference, mute, «Покинуть чат» (если не владелец).
 В групповом чате: avatar, files (если есть), changeTaskText, videoConference, mute, «Покинуть группу», pin.
 В канале: files (если есть), changeTaskText (только владелец), mute, «Отписаться» (не владелец).
 В задаче с `viewMode = chat`: avatar, files (если есть), changeTaskText, videoConference, mute, «Отписаться», pin.

Шаблоны блоков описывают структуру и поведение плиток рабочего стола, пунктов меню, лент задач и других элементов контейнеров мобильного приложения.
 Обработка шаблонов реализована в коде приложения: изменения должны строго соответствовать ему. Редактирование шаблонов требует аккуратности — лучше выполнять при участии поддержки 1Ф.

Управляют расположением, логикой отрисовки и отображением плитки на дашборде.
 Ключ Описание `badgeItemColor` Цвет элемента значка (счётчика) `fallBackTitle` Заголовок плитки. Если пусто — может формироваться автоматически из названия категории. `footerTitle` Текст подвала (нижней части) `groups` ID групп через запятую. Если пусто — плитка для всех групп. При непустом массиве МП определяет пересечение групп пользователя и массива `groups`. Если пересечение пустое — элемент не отображается. Отображение не меняется при замещении — только группы реального пользователя. `headerBackgroundColor` Цвет фона заголовка (`#000000`). Если не задан — прозрачный. `heightMultiplier` Высота плитки. 1 единица = 1/2 ширины экрана в режиме «портрет». Для `dashboardStack` = сумма высот блоков внутри папки. `icon` Название преднастроенного значка (см. полный список ниже). Переопределяется в плитке, иначе берётся из template уровня выше. `iconColor` Цвет иконки. Если не задан — берётся от `titleColor`. `interactions` JSON взаимодействия: `{"tap": {onTapDto}, "longTap": {onTapDto}}`. `textColor` Цвет текста (`#000000`) `titleColor` Цвет заголовка (`#000000`) `verticalGradientColor` Вертикальный градиент (`#00000066` или `#0006`) `widthRatio` Ширина: 1=1/3 экрана, 1.5=1/2, 2=2/3, 3=полная ширина, 4=полная ширина дашборда (с листанием). По умолчанию 1.

Шаблон экрана рабочего стола.
 Ключ Описание `bgImage` Изображение фона `bgImageURL` URL фона с анонимным доступом `cellCounterFontSize` Размер шрифта счётчика (по умолчанию 15) `cellIconSize` Размер иконки (по умолчанию 16) `cellTextFontSize` Размер текста внутри ячейки `cellTitleFontSize` Размер заголовка ячейки (по умолчанию 15) `containerBackgroundColor` Цвет фона контейнера `rightBarButtons` JSON onTap действий `showStyle` Тип анимации: `expand`, `push`, `swipe`, `background` `showTileBorderShadow` Тень для виджета на рабочем столе `tileCornerRadius` Радиус углов плиток (по умолчанию 6 pt) `tileSpacing` Отступ от края экрана

Шаблон одной плитки рабочего стола. Плитки на половину экрана — для навигации. Плитки на всю ширину — не более 2 задач.
 Ключ Описание `actionStyle` `push` (переход), `popup` (всплывающее окно) `autoHeight` Высота по контенту `bgImageUploadId` UploadId фонового изображения `bgImageURL` URL фона `cellCounterFontSize` Размер шрифта счётчика `cellIconSize` Размер иконки (по умолчанию 16) `cellStyle` `default` / `custom`. При `custom` — ячейка отображается по шаблону Cell (краткое представление в шаблонизаторе). `cellTitleFontSize` Размер шрифта заголовка (по умолчанию 15) `cellTextFontSize` Размер шрифта текста `click` JSON описания нажатий на разные части плитки `context` Контекст из настроенного динамического шаблона `disabled` `1` — нажатие ничего не делает, `0` — обычное `emptyContentTitle` Заголовок при пустой плитке `forbidCreateTask` `0` — создание разрешено, `1` — запрещено `hideCounter` `1` — скрыть счётчик. Счётчик — тикер в кружке/овале, цвет фона = `textColor` с альфой 0.3 `hideOnZeroCount` `0` — только при ненулевом счётчике, `1` — всегда `onTapAlert` Текст алерта при нажатии. Связан с `needConfirm`. `leftEdgeColor` Цвет левой границы ячеек дашборда `needConfirm` `1` — вместо алерта показывать конферм с кнопками «Отменить»/«Подтвердить» `slideShow` Автоматическая прокрутка (карусель) `subcatId` ID категории для создания задачи `taskFilter` Фильтр (JSON для task/feeds) `tileCornerRadius` Радиус углов `tileSpacing` Отступ от края Примечание: `hideCounter`, `hideOnZeroCount` и `badgeItemColor` управляют отображением уже полученного счётчика, но не включают его расчёт. Для плиток на основе TaskSource счётчик платформой в общем случае не рассчитывается.

Шаблон блока со ссылкой.
 Ключ Описание `allowPreviewInteraction` `1` — взаимодействие с контентом внутри фрейма (нажатия, переходы внутри previewURL) `autoHeight` Высота по контенту `isButton` `1` — по нажатию запускается асинхронный запрос по URLPath (без открытия), `0` — открывает ссылку `openInBrowser` `1` — открывать ссылку вне МП (в браузере) `phone` Номер телефона для звонка (`+74951234567`) `previewURL` Ссылка для отображения webView внутри плитки `URLPath` Ссылка, открываемая по нажатию. Если не указана — плитка не кликабельна.

Плитка подписей с ограничением на группы. Ключи аналогичны `dashboardItem` плюс:
 Ключ Описание `excludeSubcatIds` Список категорий для исключения из показа `includeSubcatIds` Список категорий для показа `showAcceptAll` Показ кнопки «Подписать все» (при более 1 подписи). По нажатию выносится резолюция «Согласовать» по всем отображаемым подписям с подтверждением. `showCategoryFilter` Показ кнопки отбора подписей по категориям

Стек элементов дашборда (объект палитры «Папка»). Используется для иерархии вложенных элементов.
 Ключ Описание `bgImageURL` Фоновое изображение стека `cellTitleFontSize` Размер шрифта заголовка (по умолчанию 15) `cellTitleColor` Цвет текста под кнопками в стеке `colCount` Число колонок (устаревшее) `iconURL` URL иконки по ссылке (приоритет над `icon`) `presentationStyle` Галерея кнопок: `small` (по умолчанию) / `medium`. Для `style = horizontal`. МП игнорирует внутренние настройки декора и использует фиксированный дизайн. Размеры кнопок: 40, иконки: 20, радиус: 12. `separatorColor` Цвет сепаратора в квадратном стеке `showIconCircle` Показ круга вокруг иконки `showSeparators` Показ разделителей между элементами `style` `vertical` / `horizontal` / `square` `titleFontSize` Размер шрифта заголовка плитки Параметры счётчика у элемента стека:

- Указать ненулевое количество задач у элемента папки.

- Задать `badgeItemColor`.
  Вертикальный стек (`style = vertical`):

- `showSeparators` — разделители между элементами.

- `heightMultiplier` — сумма высот всех блоков внутри папки.

- `headerBackgroundColor` — фон стека (по умолчанию белый).

- Внутри папки может быть любое число блоков любого типа.
  Особенности информационной плитки в вертикальном стеке: - Если приходит пустой массив — рисуется 1 ячейка с `fallBackTitle`. - Если `hideOnZeroCount = 1` и счётчик нулевой — блок скрывается. Если все блоки скрыты — папка скрывается целиком. - Если пришёл непустой массив и `hideOnZeroCount = 1` — размер блока подстраивается под количество задач. - По нажатию на ячейку — переход в карточку задачи.
 Горизонтальный стек (`style = horizontal`):

- Выравнивание по центральной линии, иконка фиксированная, блоки распределены равноудалённо.

- Есть возможность настроить действие при тапе на стек через `onTap`.

Шаблон экрана меню.
 Ключ Описание `containerBackgroundColor` Цвет фона контейнера `rightBarButtons` JSON onTap действий `showStyle` Тип анимации

Шаблон одного пункта меню.
 Ключ Описание `cellStyle` `default` / `custom` `context` Контекст из динамического шаблона

Разделительная линия между пунктами меню. Пустая ячейка высотой 20px, без данных.

Контейнер для задания содержимого и вида лент задач.
 Ключ Описание `addButton` JSON кнопки создания задачи `catId` ID категории `cellStyle` `default` / `custom` `feedType` Тип ленты: `All`, `Owner`, `Performer`, `Favorites`, `Overdue`, `Discussions`, `Created`, `New`, `Likes`, `LastCommented`, `PinnedToChat`, `PrivateChat`, `GroupChat` (6=MyQuestionsAndAnswers, 7=QuestionToMeAndAnswers) `interactions` JSON взаимодействий `searchType` `like` / `fullText` / `contains` `showCreateButton` Кнопка создания задачи `showSearchButton` Кнопка поиска `showTabMenuButton` Кнопка popup меню из таббара `subcatId` ID категории для создания задачи `tickerAlias` Счётчик: `overDueTasksCount`, `myQuestionsCount`, `unreadCommentsCount`, `questionsCount`, `signaturesCount`, `directorSignaturesCount`, `overdueSigns`, `missedCalls`, `milestones`, `unreadChatCommentsCount`, `badge`, `allTasksUserOwns`, `allTasksUserPerforms`, `95` `viewMode` `default` / `calendar` `topTitle` Альтернативный заголовок в верхнем баре. Если не задан — берётся из таббара. Типы лент комментариев (`lentaCommTypes`): 0=Все, 1=Непрочитанные, 4=Вопросы мне, 3=Вопросы мои, 5=Избранные, 6=Мои вопросы и ответы, 7=Вопросы мне и ответы.

Плитка источника данных с выбором типа отображения списка.
 Ключи аналогичны `dashboardItem` плюс:
 Ключ Описание `dataSourseUrl` Ссылка на публикацию `objectId` ID объекта для открытия `parentTaskId` Задача-шаблон, из которой копируется ДП `showSingleTask` `1` — если одна задача, открывать её напрямую `showTaskOrCreate` `1` — при пустом списке автоматически создать задачу и открыть `showTaskOrNTF` Открыть задачу или карточку создания `taskTemplateName` Шаблон задач: «План развития», «Чеклист», «Событие» `UrlPath` Ссылка по нажатию

Шаблон нижнего меню (таббар).
 Ключ Описание `badgeItemColor` Цвет значка `containerBackgroundColor` Цвет фона контейнера `itemColor` Цвет элемента на таббаре `selectedItemColor` Цвет выделенного элемента

Шаблон кнопки нижнего меню.
 Ключ Описание `addButton` JSON кнопок создания (иконка, action, title, id, isSeparated) `context` Контекст из динамического шаблона `interactions` JSON взаимодействий `objectId` ID объекта для открытия `parentTaskId` Задача-шаблон `isDefault` Вкладка по умолчанию `showTabMenuButton` Показ кнопки popup в таб меню `topTitle` Альтернативный заголовок верхнего бара `onLongTap` JSON модалки при длинном нажатии `onTap` JSON onTap действий Настройка вкладок чатов: через элемент-папку с id `Chats` можно настроить несколько вкладок на экране чатов. Содержимое вкладки — ID TasksFeed, TaskSourceXX или SubcatXX. Сортировка: по дате последнего комментария (чем позже — тем выше). Все блоки должны иметь `cellStyle: chat`.

Шаблон представлений профиля пользователя.
 Ключ Описание `bgImageURL` URL фона с анонимным доступом `cellIconSize` Размер иконки (по умолчанию 16) `cellTitleFontSize` Размер шрифта заголовка `cellTextFontSize` Размер шрифта текста `fallbackTitle` Заголовок плитки `groups` Ограничение по группам (аналогично общему ключу) `headerBackgroundColor` Цвет фона заголовка `heightMultiplier` Высота плитки `icon` Иконка `iconColor` Цвет иконки `showVCardQR` `1` — при нажатии открывает QR-код VCard профиля `tileCornerRadius` Радиус углов `tileSpacing` Отступ от края `titleColor` Цвет заголовка `verticalGradientColor` Вертикальный градиент `widthRatio` Ширина

Плитка с медиа: изображение или видео. Аналог `dashboardItem`, но с приоритетом графического контента над текстом. Используется в каталогах, новостях, рекламных блоках.
 Ключ Описание `bgImageURL` URL изображения или превью видео (анонимный доступ) `bgImageUploadId` UploadId медиа (вместо URL) `videoURL` URL видео (если плитка видеоплеер) `slideShow` Автоматическая прокрутка нескольких медиа `tileCornerRadius`, `tileSpacing`, `widthRatio` Геометрия плитки (аналогично `dashboardItem`)

Скрываемый баннер с пользовательской ссылкой. Появляется на дашборде, пользователь может закрыть его кнопкой «×» — повторно не показывается до сброса.
 Ключ Описание `bgImageURL` URL фонового изображения баннера `URLPath` Ссылка, открываемая по нажатию `openInBrowser` `1` — открывать ссылку вне приложения `dismissible` `1` — баннер можно скрыть (по умолчанию `1`) `title`, `subtitle` Текст баннера `groups` Ограничение по группам Скрытие баннера хранится локально в МП-клиенте.

Иерархическое представление задач категории по нескольким ДП. На каждом уровне иерархии задачи группируются по значению очередного ДП; пользователь сворачивает/разворачивает группы.
 Ключ Описание `subcatId` Категория-источник задач `extParamIds` Массив ID ДП, по которым строится иерархия (порядок = порядок уровней) `taskFilter` Дополнительный фильтр на задачи (JSON для `task/feeds`) `cellStyle` `default` / `custom` (тогда — по шаблону Cell) `cellTitleFontSize`, `cellTextFontSize` Шрифты ячейки Глубина иерархии не ограничена количеством ДП в `extParamIds`; платформа последовательно группирует на каждом уровне.

Список значений ДП пользователя в виде дашборд-плитки. Используется для быстрого доступа к персональным параметрам (например, доступные орг.единицы, любимые проекты).
 Ключ Описание `extParamId` ID пользовательского ДП (тип «Выбор пользователей», «Список значений») `cellStyle` `default` / `custom` `taskFilter` Фильтр для перехода в список задач при тапе на элемент Параметр читается из профиля текущего пользователя (`UserExtParams`), не из задачи.

Не блок-шаблон, а служебный id контейнера для отдельного дашборда «Избранное» — параллельный экран наряду с главным `Dashboard`, `TabBar`, `MainMenu`.
 Настройка:

- На главном дашборде добавить элемент типа «Дашборд» с шаблоном `dashboard`.

- Указать в поле `id` значение `FavouritesMenu`.

- Содержимое контейнера МП загружает отдельным запросом (`POST /api/mobile/containers ['FavouritesMenu']`).
  Цвет иерархии можно менять только для контейнера `FavouritesMenu`.

Полный список значений ключа `icon` (преднастроенные значки):
 DBAdvertising, DBAgreement, DBAirport, DBAlarmClock, DBBarcode, DBBeginner, DBBinoculars, DBBookmark, DBBox, DBBoxImportant, DBBriefcase, DBBusiness, DBCalendar, DBCalls, DBCategory, DBChats, DBCheckAll, DBChecked, DBChip, DBChoice, DBClock, DBCoins, DBCollaboration, DBCollapse, DBCollect, DBComments, DBConference, DBConnect, DBContacts, DBCreate, DBCurrencyExchange, DBDocumentary, DBDownloadFromTheCloud, DBEdit, DBError, DBEvents, DBExample, DBExchange, DBExclamationMark, DBFile, DBGamepad, DBGood, DBHandshake, DBHighImportance, DBHome, DBIcons8, DBIdea, DBInstagram, DBInstagramOld, DBInTransit, DBKey, DBLeaderboard, DBLink, DBLogistics, DBManager, DBManual, DBMaterials, DBMedal, DBMedia, DBMySpace, DBMySpace3, DBMySpaceApp, DBNewProduct, DBNews, DBOk, DBOpenedFolder, DBParticipants, DBPen, DBPlus, DBProjects, DBPuzzle, DBQRCode, DBQuestionIn, DBQuestionMark, DBQuestionOut, DBReddit, DBRate, DBReport, DBRestart, DBRestaurant, DBSearch, DBSellProperty, DBShare, DBSign, DBSmileQuestion, DBStructure, DBSun, DBSupport, DBSwot, DBSynchronize, DBTasks, DBThink, DBTickBox, DBTinder, DBToolbox, DBTraining, DBTreeStructure, DBTwitter, DBUncheckAll, DBVideoCall, DBWallet.

Если для ключа в шаблоне включена опция «Локализуемый», то в контейнере напротив ключа отображается значок локализации. По нажатию открывается окно для ввода значений на всех языках системы. При выборе языка интерфейса МП — значения локализуемых ключей отображаются на этом языке.
