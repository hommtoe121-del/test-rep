# Формы задач (MTF/NTF)

Источник: https://help.1forma.ru/domains/task-forms/admin/

Документ описывает администрирование форм задач 1Формы: настройку карточки задачи (MTF) и формы создания новой задачи (NTF), компоновку элементов и панель инструментов, JS/CSS-вставки, глобальные JS-ресурсы, справочник JS API (методы ExtParam, кнопки перехода, порталы, файлы, телефония), диалоги, веб-сервисы Valhalla, миграцию на новый MTF/NTF и примеры JS/CSS-вставок. Внутри — настройки категории для MTF, шаблоны поведения, массовое управление панелью через API, жизненный цикл JS-вставок, таблица совместимости методов и типов ДП, интеграция с публикациями пакетов действий.

⚠️ На внешний вид карточки задачи влияют также глобальные флаги `UseNewMTF` и `UseNewMTFStyle` в файле `web.config`.
 Основные настройки карточки задачи (MTF), связанные с отображением, датами и повторением:
 Настройка Описание Отображать основной маршрут Если активна, в карточке отображается лента основного маршрута. Переходы доступны, если в их настройках включена опция «Основной маршрут». Разрешить назначать приоритет В карточке созданной задачи отображается приоритет; по клику можно изменить выбор. Отображать Статус Отображает текущий статус задачи. Автоматически отключена для категорий с типом «Пространство». Отображать Начало работы Доступно поле «Начало работы». Если отключено, скрывается и опция «Разрешено устанавливать дату начала работы при постановке». Разрешить ставить дату начала работы в прошлом Разрешает указать дату начала раньше даты создания задачи. По умолчанию запрещено — при попытке выбрасывается ошибка. Проверка срабатывает в MTF, при создании из чата и через API. Введена в версии 2.267. Разрешить изменять дату начала работы в завершенных задачах Поле «Дата начала» остаётся доступным после перехода в завершающий статус. По умолчанию отключено. Действует только на «Дата начала», не влияет на «Срок». Активность конфигурации повторения Смарт-фильтр, определяющий активность серии повторений. Если фильтр не указан или возвращает `true`, задание по таймеру создаёт новую задачу из серии. Другие действия Название системной кнопки «Другие действия». Становится доступной, если в настройках пользовательской кнопки включен параметр «Отображать в 'других действиях'». Количество колонок доп. параметров (МТФ) Определяет список доступных значений в настройке «Количество занимаемых колонок» для каждого ДП категории. По умолчанию — 3.

Основные настройки карточки задачи (MTF), связанные с действиями, компоновкой и МТФ-панелью:
 Настройка Описание Отображать кнопку Делегировать Отображает пункт «Делегировать» в меню «Участники». Отображать кнопку Перенести срок / Установить срок Отображает пункт в меню «Срок». Выводить блоки над остальными ДП Устаревшая настройка, скрыта из интерфейса. Порядок управляется напрямую в списке ДП категории. Скрыть дату создания Скрывает дату создания в карточке задачи. Компактные переходы Если отключена — кнопки перехода отображаются и в основном блоке, и как переключатель справа. Если активна — остаётся только переключатель справа. Компактные действия При включении правее кнопки статуса добавляется кнопка «Действия» с выпадающим меню смарт-кнопок. При отключении смарт-кнопки отображаются отдельно. Положение кнопок переходов и смарт-кнопок Размещение блока кнопок: «В начале формы» (по умолчанию), «В конце формы» или по смарт-выражению (должно возвращать `start` или `end`). Показывать кнопки «Следующая задача» и «Предыдущая задача» Показывает стрелки в шапке карточки для быстрого перехода между задачами. Отображаются только при открытии задачи из списка. По умолчанию отключено. Имя стандартного блока ДП Произвольное название системного блока «Параметры», в котором отображаются все ДП, не входящие ни в один блок. Секция отображается только для подкатегорий с чатовым типом представления: Личный чат, Пространство, Групповой чат, Канал соц. сети.
 Настройка Описание Показывать МТФ-панель Если активна, при открытии чата отображается правая боковая панель с карточкой задачи (МТФ). Группы доступа Мультиселект групп пользователей. Если список пуст — панель доступна всем. Если указаны группы — видят только их участники.

Панель инструментов карточки задачи состоит из набора элементов (иконка и название). Состав элементов настраивается в категории.

- Чтобы отключить видимость элемента, надо отключить индикатор напротив него в дереве элементов.

- При обновлении платформы новые элементы панели инструментов автоматически добавляются в конфигурацию категории (включёнными по умолчанию), даже если она ранее настраивалась вручную.

- Некоторые элементы не отображаются на панели, даже если включены в дереве — их видимость определяется настройками категории (например, режим выборочного шифрования или проектное управление).
  ⚠️ На видимость кнопок «Делегировать» и «Перенести/Установить срок» влияют не только настройки категории, но и права доступа конкретного пользователя.
 Если в конфигурации панели инструментов не выбрано ни одного значения, панель инструментов полностью скрывается.
 Переход к настройкам из пользовательского интерфейса: администраторам доступен пункт «Настройки» в контекстном меню при ПКМ на панели инструментов карточки задачи.



`POST api/admin/subcategories/toolbar/mass-update` — скрывает или показывает конкретную кнопку (по `LanguageKey`) сразу в нескольких категориях.
 Параметр Тип Описание `LanguageKey` string Ключ элемента панели инструментов `IsVisible` bool `true` — показать, `false` — скрыть `SubcatIds` int[] Список ID категорий (необязательно, если указан `SubcatConfigType`) `SubcatConfigType` int Тип категории (необязательно, если указан `SubcatIds`) `POST api/admin/subcategories/toolbar/reset-to-default` — сбрасывает пользовательскую конфигурацию панели инструментов на стандартную для указанных категорий.
 Параметр Тип Описание `SubcatIds` int[] Список ID категорий

Параметры в карточке задачи отображаются в той последовательности, в которой они находятся в таблице настроек ДП категории. Для управления очерёдностью перетащите строку мышью на нужное место.
 Чтобы изменить количество ДП, отображаемых в одной строке, задайте значение в настройке Количество занимаемых колонок. Большинство ДП по умолчанию размещаются в одной колонке, за исключением:

- Таблица;

- Большой текст с/без форматирования;

- Выбор нескольких задач из категории (Multilookup).
  Количество доступных колонок определяется настройкой категории Количество колонок доп. параметров (МТФ) (по умолчанию — 3).
 Если у дополнительного параметра установлен флаг «Отображать с новой строки», это поле и все следующие за ним в порядке отображения начинаются с новой визуальной строки. Поведение сохраняется при любой ширине экрана — поле может занимать одну колонку или всю ширину, но визуально оно всегда начинается с новой строки. Если поле со флагом скрыто, последующие поля отображаются стандартно.

Если поля объединены в блоки ДП, они располагаются в порядке отображения на странице настроек, но в пределах своих блоков. Поля, не входящие в блоки, отображаются на форме над всеми блоками ДП и группами блоков ДП.
 Порядок расположения блока ДП относительно других блоков и БИ устанавливается в настройках блоков в колонке Порядок на форме.
 Блоки ДП могут быть объединены в группы. Порядок следования блоков в группе определяется настройкой Порядок на форме. Группы блоков могут чередоваться в карточке задачи с БИ и другими блоками ДП.
 Относительно необъединённых в блоки параметров блок «Используется» отображается в нижней части формы задачи под всеми параметрами.
 Если ДП объединены в блоки, порядок БИ и блоков ДП соответствует настройкам Порядок на форме. Если у блока «Используется» порядок = 1, а у блока ДП = 2, в карточке блок ДП располагается ниже БИ, и наоборот. Для групп блоков ДП и БИ логика сохраняется.
 Блоки ДП вне группы, группы блоков ДП и БИ отображаются в задаче в соответствии с номером значения в их настройке Порядок на форме.
 Последовательность отображения параметров, которые не включены в блоки, и блоков ДП (включая группы блоков и БИ) управляется напрямую в списке ДП категории — перетащите строки в нужном порядке. Настройка категории «Выводить блоки над остальными ДП» больше не используется и скрыта из интерфейса.

JavaScript используется для динамических настроек, когда поведение меняется в зависимости от условий. CSS-вставки применяются для статичных настроек, неизменных при любых условиях.

При работе с карточкой задачи текст JS-вставки — это действия, вызываемые после загрузки окна с карточкой задачи, то есть после события `MTFMainLoadFinished`.
 `window.addEventListener('MTFMainLoadFinished', function() {
  // ...
});
` При переходе на новый MTF обязательно должно быть использовано событие `MTFMainDestroyed`, которое позволяет очищать память при изменении контекста задачи (закрытие, переключение на другую задачу, смена `taskId` и т.д.).
 `window.addEventListener('MTFMainDestroyed', function onMTFDestroy(e) {
  // cleanup
}, { once: true });
` При работе с карточкой новой задачи используется событие `NewTaskLoadFinished`:
 `window.addEventListener('NewTaskLoadFinished', function() {
  // ...
});
` При переходе на новую карточку создания задачи (NTF) обязательно должно быть использовано событие `NewTaskDestroyed`:
 `window.addEventListener('NewTaskDestroyed', function onDestroy(e) {
  // cleanup
}, { once: true });
` ⚠️ Без использования событий `MTFMainDestroyed` / `NewTaskDestroyed` JS-вставки при переходе в новый MTF/NTF работать не будут корректно.
 ⚠️ В карточке новой задачи методы `save()` и `update()` недоступны, так как задача ещё не существует в БД.
 В обработчик `MTFMainLoadFinished` передаётся объект `e.detail`:
 `window.addEventListener('MTFMainLoadFinished', function onMTFInit(e) {
  const ctx = e.detail;
  const taskInitId = ctx.taskId;
  const rootEl = ctx.root;
});
`

Позиция блока указывается одним из значений:

- `'MainBlockAfter'` — после блока системных параметров;

- `'MainBlockBefore'` — перед блоком системных параметров, после панели инструментов;

- `'SecondaryBlockBefore'` — перед блоками ДП и БИ;

- `'SecondaryBlockAfter'` — после блоков ДП и БИ, перед комментариями.
  Пример:
 `window.addEventListener('MTFMainLoadFinished', function onMTFInit(e) {
  const ctx = e.detail;

  window.addEventListener('MTFMainDestroyed', function onMTFDestroy(e) {
    const ctx = e.detail;
    const taskDestroyId = ctx.taskId;
  }, { once: true });

  ctx.insertHtmlBefore('mainparams', '<div class="test">test</div>');
  ctx.insertHtmlAfter('extparams', '<div class="test">test</div>');
  ctx.insertHtmlBefore('comments', '<div class="test">test</div>');
  ctx.insertHtmlBefore('block:2490', '<div class="test">test</div>');
});
` Вставка в категорию осуществляется в настройках категории. Если одну и ту же вставку нужно подключить к нескольким категориям, можно сделать это через БД, добавив записи в таблицу `SubcatIncludes`.
 Вставки для пользователей определённых групп настраиваются на странице рабочего места группы по кнопке JS/CSS.

JS и CSS вставки, используемые в категории и на портале, можно сделать общими — доступными для использования в других категориях и порталах. Для этого их нужно добавить в список общих вставок и отметить контекст использования.
 ⚠️ Невозможно удалить вставку, если она используется в категории. При попытке удалить используемую вставку система выведет сообщение с перечнем категорий, где она подключена.
 При написании JS-вставок придерживайтесь рекомендаций:

- Текст JS-вставки дробить на отдельные мелкие функции.

- Названия переменных, объектов и функций должны быть значимыми.

- Длинные названия записывать в стиле `camelCase`.

- Названия классов давать по принципу `Block__Element_Modificator`.

- Комментарии в коде — умеренно.

- Скрипты лучше подключать не из верстки Smart Html, а из JS-вставок к порталу или виджетам.

- После отладки убирать или комментировать вызовы `console.log()`, так как они замедляют исполнение.

При написании JS-вставок можно использовать русскоязычные аналоги методов и объявлять объекты сущностей (ДП) непосредственно по имени ДП в категории.
 Пример:
 `// Классический подход
const test = new ExtParam(2454);
const test2 = new ExtParam(7500);
test.val() = test2.val();

// Русскоязычный аналог
var test = ДП.Спринт();
var test1 = ДП.Сборка();
test.Значение() = test1.Значение();
` Метод Аналог `val()` `Значение()` `textVal()` `ТекстовоеЗначение()` `getAvailableValues()` `ВозможныеЗначения()` `change()` `ИзменитьЗначение()` `save()` `Сохранить()` `update()` `Обновить()` `isHidden()` `Скрыть()` `freeze()` `Заблокировать()`

В системе есть возможность использовать JS при инициализации и авторизации в приложении. Для этого в `custom-app-settings` необходимо добавить ключ `spaResources`, в котором указать JS-ресурсы, загружаемые при инициализации или успешном входе.
 `"spaResources": [{ "type": "js", "src": "https://адрес_ссылки.js" }]
`

Глобальные события приложения:
 Событие Описание Пример `init` После инициализации приложения `spaEvent('init', e => console.log('init', e))` `auth` После авторизации в приложении `spaEvent('auth', e => console.log('auth', e))` Глобальные команды (`spaCommand`):
 Команда Описание Пример `closeActiveModal` Закрыть открытое модальное окно `spaCommand('closeActiveModal');` `openPortal` Открыть портал `spaCommand('openPortal', { portalId: 123 });` `open-file-preview` Открыть превью файла `spaCommand('open-file-preview', { previewLink: '...' });` `openTask` Открыть задачу `spaCommand('openTask', { taskId: 123, modal: true });` `navigate` Переход по ссылке `spaCommand('navigate', { url: '/user/profile/1234' });` Для получения информации о текущем пользователе:
 `await spaApi.getSessionUser().data
`

В JS-вставках в карточке задачи можно обращаться к номеру задачи — `taskId`, и к статусу задачи — `StateID`.
 ID пользователя, от имени которого ведётся работа, доступно в `SessionUserID`.
 ⚠️ Названия основных параметров регистрозависимые.
 Методы JS-объекта ДП:
 Метод Аналог Что делает `var ep = new ExtParam(1234)` `ДП.{Имя_ДП}()` Получает JS-объект для управления ДП с ID=1234 `ep.get()` — Возвращает jQuery-объект, содержащий ДП `ep.label()` — Возвращает jQuery-объект подписи `ep.label().html()` — Возвращает текст подписи `ep.label("text")` — Меняет текст подписи и возвращает jQuery-объект `ep.show()` — Показывает ДП и подпись `ep.hide()` — Скрывает ДП и подпись `ep.val()` `Значение()` Получает значение ДП `ep.textVal()` `ТекстовоеЗначение()` Получает значение ДП в текстовом виде `ep.val(param)` — Устанавливает значение ДП, вызывает событие `change` `ep.getAvailableValues(handler)` `ВозможныеЗначения()` Получает список возможных значений ДП Lookup и Multilookup. Handler принимает массив `{ text, value }` `ep.change()` `ИзменитьЗначение()` Запускает обработчик `change()` для ДП `ep.change(handler)` — Подписывает на изменение ДП `ep.change(null)` — Отписывает все обработчики на изменение ДП `ep.save()` `Сохранить()` Сохраняет изменения ДП на сервере `ep.save(handler)` — Подписывает на сохранение ДП `ep.save(null)` — Отписывает все обработчики на сохранение `ep.update()` `Обновить()` Обновляет ДП с сервера `ep.update(handler)` — Подписывает на обновление ДП с сервера `ep.update(null)` — Отписывает все обработчики на обновление `SaveEPsWithIDs([1234, 1235])` — Сохраняет все ДП по массиву индексов. Срабатывают триггеры на `save()` для каждого ДП `ep.isHidden()` `Скрыть()` Принимает `true`, если ДП нет на форме или он скрыт настройками категории `ep.freeze()` `Заблокировать()` Делает ДП доступным только на чтение (`true` / `false`) `ep.adaptDesign()` — Устаревшее, не работает с версии 2.256. Ранее при `true` верстка адаптировалась при скрытии/показе. ⚠️ При использовании `ep.hide()` в разметке останется пустое пространство. Чтобы скрыть ДП без пустого места, используйте CSS-селектор со свойством `display: none`:
 `document.querySelector('vh-ext-param-control-wrapper[ep-wrapper-id="XXX"]').style.display = 'none'
` ⚠️ Для ДП «Выпадающий список» и «Выпадающий список с редактированием» в режиме только для чтения событие, вызванное смарт-автоматизацией, запускается дважды. Чтобы отследить это, можно проверять `new ExtParam().val()` — первый раз возвращается старое значение, второй — новое.
 ⚠️ Если на карточке расположены два связанных ДП, доступных для редактирования, то при вызове обработчика `change()` для родительского ДП значение подчинённого ДП сбрасывается.

Методы, не указанные в таблице, работают для всех ДП.
 Тип ДП `ep.val()` `ep.val("text")` `ep.save()` `ep.update()` Флажок (checkbox) ✅ ✅ (`"да"` / `"on"` / `true` / `1`) ✅ ✅ Дата ✅ ✅ ✅ ✅ Дата и время ✅ ✅ ✅ ✅ Файл ❌ ❌ ❌ ❌ Lookup ✅ (ID задачи) ✅ (ID задачи) ✅ ✅ Деньги ✅ ✅ ✅ ✅ Multilookup ✅ (JSON массив ID) ✅ (JSON массив ID) ✅ ✅ Нумератор ✅ ❌ ❌ ❌ Число ✅ ✅ ✅ ✅ Выпадающий список ✅ ✅ ✅ ✅ Выбор нескольких задач из категории ✅ (ID задачи) ✅ (ID задачи) ✅ ✅ Выбор пользователей ✅ (JSON массив) ❌ ✅ ✅ Таблица ✅ (HTML таблица) ❌ ❌ ❌ Текст ✅ ✅ (без шаблона номера телефона) ✅ ✅ Большой текст с форматированием ✅ ✅ ✅ ✅ Большой текст без форматирования ✅ ✅ ✅ ✅ Сквозной ✅ ✅ ✅ ✅ Дерево ❌ ❌ ❌ ❌ URL ✅ ✅ ✅ ✅

Структура объектов ДП «Таблица»:
 `object EpTable: {
  savedRowsCount: numeric,
  filteredRows: [ object Row ],
  multiWindow: object MultiWindow
}

object Row: {
  cells: [ object Cells ],
  id: number
}

object Cell: {
  columnId: numeric,
  columnValue: mixed,
  tooltip: function
}

object MultiWindow: {
  filteredRows: [ object Row ]
}
` Методы:

- `onTableLoaded` — загрузка/перезагрузка страницы, переключение страниц;

- `onTableRowAdded` — добавление строки (передаётся объект строки);

- `onTableRowChanged` — изменение значения ячейки;

- `onTableSaved` — сохранение таблицы;

- `onTableMWLoaded` — загрузка/перезагрузка в режиме множественного выбора;

- `onTableMWRowSelected` — выбор строки;

- `onTableMWRowChanged` — изменение значения ячейки в режиме множественного выбора;

- `onTableMWClosed` — закрытие окна множественного выбора;

- `tooltip()` — подсказка при наведении на ячейку.
  ⚠️ Методы обновления ДП «Таблица» вызываются после отработки метода обновления карточки задачи (`MTFMainLoadFinished`).
 Пример:
 `function setTooltip() {
  var ept = window.firstForma.ep.table(11);
  ept.filteredRows.forEach(function(item) {
    item.cells.forEach(function(cell, i) {
      var tmp = 'TOOLTIP:' + i;
      cell.tooltip(tmp);
    });
  });
}
document.addEventListener('load', function() { setTooltip(); });
`

Для полной замены текущего значения в функцию `ep.val()` нужно передать JSON:
 `{"Users":[], "Groups":[], "OrgUnits":[]}
` Пример:
 `var ep123 = new ExtParam(123);
ep123.val('{"Users":[55,66], "Groups":[77], "OrgUnits":[]}');
ep123.save();
` При работе с ДП «Выбор пользователей» функция `ep.val()` возвращает JSON:
 `{"Users":{"Deleted":[],"Added":[],"Selected":[]},"Groups":{"Deleted":[],"Added":[],"Selected":[]},"OrgUnits":{"Deleted":[],"Added":[],"Selected":[]}}
` Пример получения ID выбранного пользователя:
 `var obj = JSON.parse(ep123.val());
var userid = obj.Users.Selected[0];
` Текущий (сессионный) пользователь:
 `// В JS-вставках доступна переменная UserID
// Также можно получить через:
spaApi.getSessionUser().data.__zone_symbol__value.id
` Методы JS-объекта кнопки перехода:
 Метод Что делает `var step = new TaskStepAPI(1234)` Получает JS-объект для управления кнопкой перехода `step.hide()` Скрывает кнопку `step.show()` Отображает кнопку `step.click()` Эмулирует клик по кнопке `step.click(handler)` Подписывает обработчик на клик `step.click(null)` Отписывает все обработчики (кроме стандартного) `step.makeStep()` Выполняет стандартное действие по клику `step.cancelDefaultAction()` Снимает стандартный обработчик клика `step.restoreDefaultAction()` Восстанавливает стандартный обработчик клика

Для порталов flex:
 `(window.firstForma.portal.block(123)).onLoaded(function(event) {
  // event содержит portal {id, title} и block {id, title}
});
` Для порталов dashboard:
 `(window.firstForma.portal.block(123)).onLoaded(function(event) {
  var blockId = event.data.blockId;
});
` Методы виджетов:
 Метод Что делает `refresh()` Обновляет виджет `spaApi.loadApexCharts()` Загружает библиотеку ApexCharts из SPA-бандла и возвращает Promise с конструктором `ApexCharts`. Доступен только в JS Include, привязанных к SmartHtml-блоку портала. `spaCommand(command, data)` Выполняет команду Возможные команды `spaCommand`:

- `openInContentArea` — открыть модальное окно (`content` = html-элемент или `'iframe'`, `context` = `{ url }`);

- `openNewsInContentArea` — открыть карточку просмотра новости (`taskId`).
  Для виджетов «Таблица» и «Поиск задач»:
 Метод Что делает `reload()` Перезагружает виджет `freeze()` Делает виджет недоступным для работы `unfreeze()` Делает виджет доступным `setFilters({param: 999})` Устанавливает значение параметров фильтра `filters()` Возвращает параметры фильтра в виде массива Из JS-вставок портала можно вызывать галерею для просмотра изображений:
 `window.firstForma.fileViewer(fileArray, fileIndex)
` Параметр Описание `fileArray` Массив `{ url, fileType, name, mime }` `url` URL источника файла `fileType` Тип файла: `audio`, `video`, `image`, `pdf`, `eml` (определяется автоматически по имени/URL, по умолчанию `image`) `name` Название при наведении `mime` MIME-тип `fileIndex` Индекс файла для отображения (по умолчанию 0) Используются при настроенной интеграции с IP-телефонией.
 Открыть окно вызова:
 `GetTopWindow().TCHeader.CallHistory.Open(1234)
` где `1234` — короткий номер.
 В портальном блоке:
 `window.parent.TCHeader.CallHistory.Open(1234)
`

Вместо стандартных функций `alert()`, `prompt()` и `confirm()` можно использовать следующие функции:
 Функция Описание `dialogs.alert('Заголовок', 'Текст', function())` Окно с сообщением и кнопкой подтверждения `dialogs.prompt('Заголовок', 'Текст', НачальноеЗначение, function(val))` Окно с полем ввода и кнопками подтверждения/отмены. При отмене передаётся `false`. `dialogs.custom('Заголовок', 'Текст', {параметры окна}, function(val))` Окно с двумя кнопками. В функцию передаётся `true` (OK) или `false` (Отмена). Параметры окна соответствуют SweetAlert. `dialogs.error('Заголовок', 'Текст', function())` Окно с иконкой ошибки `dialogs.success('Заголовок', 'Текст', function())` Окно с иконкой успеха ⚠️ Если заголовок или текст не нужен, вместо них передаётся `null`. В заголовке и тексте можно задавать CSS-стили. Если функция не задана, действий по нажатию на кнопку не выполняется.

API веб-сервиса доступно по адресу `/swagger`.
 Для вызова методов Valhalla проще всего использовать jQuery-методы `$.get()` и `$.post()`.
 Пример создания задачи:
 `$.post('/app/v1.1/api/tasks', {
  "subcatId": 123,
  "text": "",
  "owner": 3,
  "createFromParentOwner": true,
  "performers": [],
  "separateTaskForEachPerformer": true,
  "subscribers": [],
  "subscribeGroups": [],
  "notifyUsers": [],
  "orderedTime": "",
  "lockOrderedTime": false,
  "taskStartTime": "",
  "plannedStartTime": "",
  "plannedEndTime": "",
  "priority": 0,
  "extParams": [
    { "id": 44, "value": "Name" },
    { "id": 55, "value": "Address" }
  ],
  "parentTaskId": null,
  "linkedTaskId": null,
  "linkedEmailId": null,
  "parentCommentId": null,
  "extParamSourceTaskId": null,
  "preuploadedFiles": [],
  "linkFiles": [],
  "parentFiles": "DoNotCopy",
  "isConfidential": false,
  "isConsisImplement": false,
  "location": "",
  "guid": "",
  "userComment": "",
  "onSuccessParams": {
    "onSuccessJs": "",
    "openInSecFrame": "",
    "returnUrl": "",
    "returnJs": "",
    "refreshRadWindowParent": ""
  }
});
` Для постановки задачи с заполнением ДП «Таблица» с колонкой типа «Мультифайл»:

- Загрузить файл в формате base64 через `/api/files/preupload/base64?initiatorUserId={initiatorUserId}`:
  `{
  "fileData": "0YLQtdGB0YLQvtCy0YvQuSDRhNCw0LnQuw==",
  "fileName": "1.txt"
}
` В ответе получается `preUploadFileId`.

- Создать задачу через `/app/v1.1/api/tasks`, передав в `extParams` значение таблицы с `preuploadId` в соответствующей колонке.

Если в API нет готового метода, можно использовать публикации пакетов действий. Чаще всего публикуются пакеты со смарт-действием «HTTP ответ». Подробнее о создании и вызове публикаций см. в документации по публикациям.
 Связанные документы:

- Справочник блоков

- Различия MTF/NTF

- Настройка ДП

- Категории

- Публикации — администрирование

При переходе на новый НТФ обязательно должно быть использовано событие `NewTaskLoadFinished`:
 `window.addEventListener('NewTaskLoadFinished', function() {
  // ...
});
` Конструкция `$(window).on('NewTaskLoadFinished', function ()` работать не будет.
 При переходе на новый МТФ обязательно должно быть использовано событие `MTFMainDestroyed`:
 `window.addEventListener('MTFMainDestroyed', function onDestroy(e) {
  // cleanup
});
` Для получения значения параметра необходимо обращаться к конкретному ключу в JSON-объекте:

- Lookup: `ep.val().taskId`

- Выпадающий список: `ep.val().text` или `ep.val().value`

- Галочка (checkbox): `ep.val() == true` (вместо `ep.val() == 'да'`)

Пример: после выбора даты и времени в ДП 74 установить в ДП 102 = ДП 74 + 1 час.
 `window.addEventListener('NewTaskLoadFinished', function(event) {
  let ep74 = new ExtParam(74);

  function f_change() {
    let ep102 = new ExtParam(102);
    if (ep74?.val()) {
      let date1 = new Date(ep74.val());
      let date2 = date1.setHours(date1.getHours() + 1);
      ep102.val(date2);
    }
  }

  ep74.change(function() { f_change(); });
  f_change();
});
` Пример скрытия/отображения ДП по условию (новый MTF):
 Пример:
 `function onLoad(event) {
  let ep21 = new ExtParam(21, event.detail.cardGuid);

  function toggleElement(id, action) {
    const card = document.querySelector(`[data-card-guid="${event.detail.cardGuid}"]`);
    if (card) {
      const element = card.querySelector(`vh-ext-param-control-wrapper[ep-wrapper-id="${id}"]`);
      if (element) {
        element.style.display = action === "show" ? "" : "none";
      }
    }
  }

  function epch() {
    if ([20706, 20662].includes(ep21.val()?.taskId)) {
      toggleElement(2181, 'show');
    } else {
      toggleElement(2181, 'hide');
    }
  }

  epch();
  ep21.change(() => epch());
}

function onDestroy() {
  window.removeEventListener('MTFMainLoadFinished', onLoad);
}

window.addEventListener('MTFMainDestroyed', onDestroy);
window.addEventListener('MTFMainLoadFinished', onLoad);
`

Пример:
 `let ep91800, ep91800content, ep91800label;

function onLoad() {
  ep91800 = new ExtParam(91800);
  ep91800content = document.querySelector('[vh-ext-param-id="91800"] > div');
  ep91800label = document.querySelector('[ep-wrapper-id="91800"] > div');

  const reactionColor = () => {
    if (ep91800?.val() > 0) {
      ep91800content.style.border = '2px solid green';
      ep91800label.style.color = 'green';
    } else {
      ep91800content.style.border = '2px solid red';
      ep91800label.style.color = 'red';
    }
  };

  reactionColor();
  ep91800.change(() => reactionColor());
}

function onDestroy() {
  window.removeEventListener('MTFMainLoadFinished', onLoad);
  ep91800?.destroy();
  ep91800?.change(null);
}

window.addEventListener('MTFMainDestroyed', onDestroy);
window.addEventListener('MTFMainLoadFinished', onLoad);
`

Пример:
 `var table22581;

function onLoad() {
  table22581 = window.firstForma.ep.table(22581);

  function changeRowColors() {
    let tableRows = document.querySelectorAll('[vh-ext-param-id="22581"] .ag-row');
    for (let i = 0; i < tableRows.length; i++) {
      let statusCell = tableRows[i].querySelector('[col-id="c10551"] > div > span');
      if (statusCell?.innerText == "Стандарт") {
        statusCell.style.color = "#00A86B";
        statusCell.style.fontWeight = 600;
      }
      if (statusCell?.innerText == "Нарушение") {
        statusCell.style.color = "#e63333";
        statusCell.style.fontWeight = 600;
      }
    }
  }

  table22581.onTableLoaded(() => changeRowColors());
  table22581.onTableRowChanged(() => changeRowColors());
  table22581.onTableRowAdded(() => changeRowColors());
  table22581.onTableSaved(() => changeRowColors());
}

function onDestroy() {
  window.removeEventListener("MTFMainLoadFinished", onLoad);
  table22581?.destroy();
}

window.addEventListener("MTFMainDestroyed", onDestroy);
window.addEventListener("MTFMainLoadFinished", onLoad);
`

Пример:
 `function showAlert5() {
  swal({
    title: 'Окончательно?',
    showCancelButton: true,
    confirmButtonColor: '#7ec9e1',
    confirmButtonText: 'Да',
    cancelButtonText: 'Нет',
    closeOnConfirm: false,
    closeOnCancel: false
  }, function(isConfirm) {
    if (isConfirm) {
      swal.close();
      document.querySelector('#stepBtnUnderTaskText1718').click();
    } else {
      swal.close();
      document.querySelector('#stepBtnUnderTaskText1902').click();
    }
  });
}

window.addEventListener('MTFMainLoadFinished', function() {
  document.querySelector('#stepBtnUnderTaskText1705').style.display = 'none';
  document.querySelector('#stepBtnUnderTaskText1718').style.display = 'none';
  document.querySelector('#stepBtnUnderTaskText1902').style.display = 'none';
});
`

Пример:
 `function onLoad(event) {
  let toe = new ExtParam(3036, event.detail.cardGuid);
  let sz = new ExtParam(99, event.detail.cardGuid);
  let dds = new ExtParam(21, event.detail.cardGuid);
  let AOorZRDS = new ExtParam(511, event.detail.cardGuid);

  function LoadSZandDDS() {
    if (toe.val()?.taskId && AOorZRDS.val()?.taskId) {
      fetch('/app/v1.2/api/publications/action/SZandDDSfromTypeOfExpenses?TOE=' + toe.val().taskId + '&AOorZRDS=' + AOorZRDS.val().taskId)
        .then(response => response.json())
        .then(function(Load) {
          sz.val({taskId: Load[0].SZ, taskText: Load[0].SZtask});
          dds.val({taskId: Load[0].DDS, taskText: Load[0].DDStask});
        });
    }
  }

  toe.change(() => LoadSZandDDS());
}

function onDestroy() {
  window.removeEventListener('NewTaskLoadFinished', onLoad);
}

window.addEventListener('NewTaskDestroyed', onDestroy);
window.addEventListener('NewTaskLoadFinished', onLoad);
`

Пример:
 `(function() {
  const participantsEpId = 10;

  function addSchedulingAssistantBtn(parentElement) {
    const assistantElement = `<div class="system-param-duedate--first ng-star-inserted"><div id="planner-button" class="vh-btn vh-btn--icon-left ng-star-inserted"> &#128197; Открыть планировщик </div></div>`;
    parentElement.insertAdjacentHTML('beforeend', assistantElement);

    document.querySelector("#planner-button").addEventListener('click', function() {
      let qsParts = [];
      let users = [];
      const participantsEp = new ExtParam(participantsEpId).val().users;
      participantsEp.forEach(item => users.push(item.userId));
      users = users.filter((value, index, self) => self.indexOf(value) === index);
      if (users.length) {
        qsParts.push('users=' + users.join(','));
      }
      const queryString = qsParts.length ? '?' + qsParts.join('&') : '';
      window.radopen('/spa/noframe/scheduling-assistant' + queryString);
    });
  }

  window.addEventListener('NewTaskLoadFinished', function() {
    const parentDivs = document.querySelectorAll('.system-param');
    const parentElement = Array.from(parentDivs).find(node =>
      node.classList.contains('system-param-duedate')
    );
    addSchedulingAssistantBtn(parentElement);
  });
})();
`

Скрыть кнопку «Удалить все файлы»:
 `i[title="Удалить все файлы"] {
  display: none;
}
` Увеличить текст задачи:
 `#txTaskText { font-size: 20px; height: 25px; padding-left: 4px; }
` Заблокировать ДП «Выбор пользователей» при создании:
 `.disabledparam64 {
  pointer-events: none;
  opacity: 0.4;
}
` Скрыть системные поля (исполнитель, срок, начало работы, кнопки):
 `#btnDelegateTask, #btnChangeDueDate { display: none; }
` Скрыть блок «Используется» (устаревший вариант):
 `#ctl00_formInner_Contenttd > div:nth-child(11) { display: none; }
` Ограничение ширины ДП:
 `vh-ext-param-control-wrapper[ep-wrapper-id="ХХХ"] {
  grid-column: span min(1, var(--task-card-params-grid-column)) !important;
}
` где `ХХХ` — ID ДП. Число `1` можно заменить на другое для изменения занимаемого места.
