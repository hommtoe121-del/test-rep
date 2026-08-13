# Импорт и публикации — паттерны

Источник: https://help.1forma.ru/domains/integrations/academy-patterns/

Документ собирает паттерны настройки импорта данных и публикаций объектов в Первой Форме. Импорт загружает задачи, значения ДП, пользователей и орг. единицы из CSV, XLS, SQL или другого приложения 1Формы. Публикации создают внешние URL и обрабатывают входящие запросы через пакеты действий и Lua-скрипты.

Функционал импорта загружает в 1Форму данные из CSV, XLS или таблиц SQL. Типичные сценарии: справочники контрагентов, списки подразделений, классификаторы. Поддерживается импорт из другого приложения 1Формы.
 Импортируемые сущности: задачи, значения ДП, учётные записи пользователей, орг. единицы.
 Создание импорта: Администрирование → Общая бизнес-логика → Импорт данных → Создать.
 Параметры: - Название — описательное имя - Тип данных — что импортируем (задачи, пользователи и т.д.) - Модуль чтения — формат файла (CSV, XLS) - Отступ строк — сколько строк пропустить в начале файла
 Сопоставление полей: для каждого поля указать: - Внешний идентификатор — обозначение столбца в файле (для Excel — буква столбца) - Идентификатор в Первой Форме — ID соответствующего ДП
 Запуск импорта: импорт запускается вручную или автоматически. Для ручного запуска через смарт-кнопку:

- Создать ДП типа "Файл" в категории (для загрузки файла импорта)

- Создать смарт-кнопку с пакетом действий

- В пакете: действие "Выполнить импорт данных" → выбрать импорт → в поле "Файл" через смарт указать ID файла из ДП

Публикация объектов позволяет использовать их вне 1Формы. Создаются URL для внешнего доступа.
 Тип Назначение SQL View Наборы данных по SQL-запросу. Выгрузка в XML/Excel. Поддержка XSLT-преобразований Пакеты действий Обработка POST/GET запросов от внешних систем. Создание задач, изменение параметров Создание публикации (пакет действий): Администрирование → Общая бизнес-логика → Публикации → Добавить публикацию.
 Параметры: - Описание — свободная форма - Алиас — уникальное имя (для URL) - Тип публикации — пакет действий - Тип содержимого — `application/json` - URL — формируется автоматически - Активность — галочка "активна"
 Входящие параметры: добавить параметр - выбрать тип: - QueryString — для GET, PUT, DELETE - RequestBody — для POST (может содержать любые входящие/исходящие параметры)
 Доступ к публикации: - Анонимный доступ — если данные не конфиденциальны - Встроенная авторизация: `/app/v1.0/api/auth/token?login=...&password=...&isPersistent=true` - Пользовательская проверка — проверка ключей в заголовке запроса

Пакет действий использует действие "Выполнить смарт-скрипт" с Lua.
 Парсинг входящего JSON:
 `local JsonParameters = UTILS:json_decode(EVENTPARAMS["PublishedObjectParameters"])
local rb = (JsonParameters or {}).requestBody or {}
` Функция создания задачи:
 `function createTask(subcat, params)
  res = SMART:execute_action('CreateTask', taskid, "task", {
    Owner = 3,
    Subcat = subcat,
    TaskText = nil,
    CreateLink = false,
    CreateSubtask = false,
    ExtParams = params,
    NewTaskCopySubscribers = false,
    CreateCopyFiles = false,
    CopyParentText = false,
    Priority = 1,
    CreateCopiesForEachPerformer = false
  })
  return res
end
` Таблица ДП для создания задачи:
 `local extparams = {
  {ExtParamId = 156, FixedValue = rb.type},
  {ExtParamId = 157, FixedValue = rb.inn},
  {ExtParamId = 139, FixedValue = rb.c_name},
}
` HTTP-ответ:
 `function HTTPresponse(content, code)
  local res = SMART:execute_action('Response', CONTEXT_ID, 'task', {
    ResponseType = 'Json',
    Content  = content,
    StatusCode = code,
    ListOfHeaders = "[{'ParameterName': 'Content-Type', 'FixedValue': 'application/json'}]",
  })
  return res
end
` Валидация и роутинг:
 `if (not rb.type or tostring(rb.type):match("^%s*$"))
   or (not rb.inn or tostring(rb.inn):match("^%s*$"))
   or (not rb.c_name or tostring(rb.c_name):match("^%s*$"))
then
  cod = '400'
  otv = 'Внимание: переданы не все обязательные параметры.'
else
  cod = '200'
  otv = 'Контрагент успешно создан.'
  createTask(50, extparams)
end
RESULT = HTTPresponse(otv, cod)
`
