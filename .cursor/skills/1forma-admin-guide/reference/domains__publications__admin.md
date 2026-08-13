# Публикации — Администрирование

Источник: https://help.1forma.ru/domains/publications/admin/

Публикации (Published Objects) — способ опубликовать бизнес-логику как внешний REST-эндпоинт: смарт-скрипт оборачивается в пакет действий, а тот — в публикацию, к которой внешние системы обращаются по HTTP. Так делаются интеграции, когда 1Форма должна отдавать данные или принимать вызовы извне.
 Цепочка создания: смарт-скрипт (Script) → пакет действий (Pack) → действие (PackAction) → публикация (PublishedObject) → доступ (ExternalObject). Всё настраивается через Admin API — отдельных форм автоадминки у публикаций нет.



Пакет действий создаётся через Admin API:
 `# DTO: PackPostDto = {pack: PackDto, editorContext: PackEditorContextDto}
# ОБЯЗАТЕЛЬНО: editorContext с noContextMode: true. Без него -- 500!
curl -s -X POST "$HOST/api/admin/smart/packs/" \
  -H "1F-Pat: $PAT" -H "Content-Type: application/json" \
  -d '{
    "pack": {
      "packId": null,
      "subcatId": null,
      "name": "my-pack-alias",
      "description": "Описание пакета",
      "isCyclic": false,
      "isPublished": false
    },
    "editorContext": {
      "noContextMode": true,
      "eventId": null,
      "subcatId": null,
      "origin": null,
      "cycleContext": false,
      "isForMail": false,
      "isForQueue": false
    }
  }'
# → {"data": {"packId": 90160, ...}}
` Критические ловушки создания Pack:

- `POST /api/admin/smart/packs/` (не `/api/admin/smart/packs/0`) — packId в body, не в URL

- `editorContext` обязателен даже для создания без категории — без него 500

- `noContextMode: true` — для автономного пакета (публикации, глобальные правила)

- `packId: null` (не 0!) — 0 → EntityNotFound

- Тело `{description: "..."}` без обёртки → 500. Нужна полная структура `{pack: {}, editorContext: {}}`

В пакет добавляется действие:
 `# DTO: PackActionPostDto = {packAction: PackActionDto, editorContext: ...}
curl -s -X POST "$HOST/api/admin/smart/packs/{packId}/actions/1" \
  -H "1F-Pat: $PAT" -H "Content-Type: application/json" \
  -d '{
    "packAction": {
      "actionId": "ExecuteSmartScript",
      "id": null,
      "order": 1,
      "parameters": [{
        "order": 0,
        "showInPreview": false,
        "scriptId": 563
      }]
    },
    "editorContext": {
      "noContextMode": true,
      "eventId": null, "subcatId": null, "origin": null,
      "cycleContext": false, "isForMail": false, "isForQueue": false
    }
  }'
# → {"data": {"actionId": "ExecuteSmartScript", "id": 102070, ...}}
` `actionId: "ExecuteSmartScript"` — строка, не число 106! API принимает enum-строку. `parameters[0].scriptId` = ID SmartScript.
 Type ExecuteSmartScript vs Response: `ExecuteSmartScript` — платформа разворачивает `{StatusCode, Content}`. `Response` — возвращает как есть → двойная обёртка. Для публикаций с данными: только `ExecuteSmartScript`.

Публикация создаётся и активируется в три шага:
 `# Шаг 1: Создать (POST без id)
curl -s -X POST "$HOST/app/v1.2/api/publishedObjects" \
  -H "1F-Pat: $PAT" -H "Content-Type: application/json" \
  -d '{
    "alias": "my-api",
    "description": "Описание",
    "httpMethod": "Get",
    "actionsPackId": '$PACK_ID',
    "parameters": []
  }'
# → HTTP 204 (пустое тело!)

# Шаг 2: Найти ID через gridData
curl -s "$HOST/app/v1.2/api/publishedObjects/gridData" -H "1F-Pat: $PAT" \
  | python3 -c "import json,sys; [print(i) for i in json.load(sys.stdin) if i.get('alias')=='my-api']"
# → {"id": 2001, "externalObjectId": "guid", "isActive": false, ...}

# Шаг 3: Активировать + установить тип Action + привязать Pack
curl -s -X POST "$HOST/app/v1.2/api/publishedObjects/$PUB_ID" \
  -H "1F-Pat: $PAT" -H "Content-Type: application/json" \
  -d '{
    "description": "Описание",
    "alias": "my-api",
    "publicationType": "Action",
    "httpRequestType": "Get",
    "isActive": true,
    "externalObjectId": "'$GUID'",
    "parameters": [],
    "publicationObject": {
      "publicationId": '$PUB_ID',
      "actionsPackId": '$PACK_ID',
      "id": 0
    },
    "id": '$PUB_ID'
  }'
# → HTTP 204
` Критические ловушки Publication:

- Создание возвращает 204 без body — ID нужно искать через `GET gridData`

- `publicationType` по умолчанию `"Data"` — для SS-публикаций нужно `"Action"`!

- Без `publicationObject.actionsPackId` — Publication отдаёт 404 при вызове

- `isActive: false` по умолчанию — нужно явно активировать через update

Доступ к публикации настраивается через объект ExternalObject:
 `# GET full → modify → POST (partial update = 500!)
curl -s "$HOST/app/v1.2/api/externalObjects/$GUID/full" -H "1F-Pat: $PAT" > /tmp/extobj.json

# Изменить visibleToAnyone и POST обратно весь DTO
curl -s -X POST "$HOST/app/v1.2/api/externalObjects/$GUID" \
  -H "1F-Pat: $PAT" -H "Content-Type: application/json" \
  -d '{
    "externalObjectsViewRights": [],
    "externalObjectsSpecialRights": [],
    "guid": "'$GUID'",
    "description": "Публикации: my-api",
    "visibleToAnyone": true,
    "isAnonymous": false,
    "id": '$EXT_ID'
  }'
# → HTTP 204
` Подробная логика проверки доступа вынесена в отдельный документ по контролю доступа к публикациям.

JS-вставки на портале обновляются через API:
 `# Include = JS-код вставки на портале. Обновляется через /api/includes/{id}/update
# GET код: через block includes API (содержимое в .content)
curl -s "$HOST/api/portals/block/{blockId}/includes" -H "1F-Pat: $PAT" > /tmp/include.json
# → [{id, type, name, content}]

# POST обновление
curl -s -X POST "$HOST/api/includes/{includeId}/update" \
  -H "1F-Pat: $PAT" -H "Content-Type: application/json" \
  -d '{"id": 5950, "name": "My Include", "type": "Js", "content": "...код..."}'
# → HTTP 200
`

Типичные ошибки
 Проблема Причина Решение 403 при вызове публикации ExternalObject access не настроен `visibleToAnyone: true` или добавить группы 500 при обновлении ExternalObject `groupIds` не существует как поле `externalObjectsViewRights: [{externalObjectId, groupId}]` ExternalObject partial update → 500 Требует полный DTO Паттерн: `GET .../externalObjects/{guid}/full` → изменить → `POST` обратно `POST /publishedObjects/{id}` Это UPDATE, не create `POST /publishedObjects` (без id) = создание Publication update → 500 Неполный DTO (забыли `name` у параметра) GET → modify → POST полный DTO. Параметры: `name` + `key` оба обязательны GET ExternalObject → 404 Путь без `/full` `GET .../externalObjects/{guid}/full` Виджет виден, данных нет Блок доступен (группы), публикация закрыта Настроить ExternalObject с теми же группами Дубль alias → 400 (не 409) alias должен быть уникален в пределах сочетания type+method Проверить: `GET /app/v1.2/api/publishedObjects/gridData` (GET, не POST!) `gridData` — 405 на POST Маршрут только GET `GET /app/v1.2/api/publishedObjects/gridData` Незарегистрированный параметр → молча отброшен Учитываются только параметры, зарегистрированные в публикации Все query-параметры должны быть зарегистрированы в `parameters[]` публикации. `id: 0` для нового параметра Pack создание → 500 Отсутствует `editorContext` или неправильная обёртка DTO: `{pack: {packId: null, ...}, editorContext: {noContextMode: true, ...}}`. Оба поля обязательны Publication → 404 при вызове `publicationType: "Data"` (по умолчанию) вместо `"Action"` UPDATE: `publicationType: "Action"`, `publicationObject.actionsPackId` = Pack ID Publication создание → нет ID в ответе 204 без тела — так задумано `GET /app/v1.2/api/publishedObjects/gridData`, фильтр по alias Pack admin API → пустые ответы Методы Pack API ненадёжны для проверки Запасной вариант: `SELECT * FROM PacksActions WHERE ActionsPackID = N`. Publication DTO → `actionsPackId` Связанные документы

- SmartScript Editor API — создание скриптов

- Виджеты на порталах, привязанные к публикациям
