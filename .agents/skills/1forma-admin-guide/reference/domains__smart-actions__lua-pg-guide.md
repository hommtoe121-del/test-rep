# Lua-скрипты — особенности PG-совместимости

Источник: https://help.1forma.ru/domains/smart-actions/lua-pg-guide/

Lua-смарт-скрипты работают на обеих СУБД (MS SQL Server и PostgreSQL). Для кросс-платформенного SQL используется функция `get_mspg_query()` с тегами `{MS}...{/MS}` и `{PG}...{/PG}` — внутри них указывается синтаксис для соответствующей СУБД.

Различия MS SQL и PostgreSQL, которые нужно учитывать в Lua-скриптах:
 Проблема MSSQL PG Решение `[Column]` quoting Работает Ошибка синтаксиса `{MS}sc.[Key]{/MS}{PG}sc.key{/PG}` Ключи в результатах `SQL:query_one` / `SQL:query` Регистр сохраняется (`Value`) Всё в нижнем регистре (`value`) `data['Value'] or data['value']` `CACHE:set` с nil `json_encode(nil)` → `"null"` То же Всегда проверять переменную перед кэшированием Устаревший кэш — — `CACHE:get` проверять на `~= nil` и `~= 'null'`. Ручного сброса нет — менять имя ключа или ждать TTL Таблицы без схемы `dbo.` Работает (`dbo` — схема по умолчанию) 42P01: relation not found Всегда указывать `dbo.TableName` `NULL` в `UNION ALL` Чистый `NULL` → OK 42804: UNION types text and bigint `CAST(NULL AS BIGINT)` — кросс-платформенно Boolean и integer колонки `= 1` везде работает 42883: integer = boolean Не все колонки `Is*`/`Allow*` на PG имеют тип boolean. Проверять тип через `information_schema.columns` При преобразовании результата запроса в словарь платформа использует доступ к ключам без учёта регистра. Поэтому в большинстве случаев обращение к полям результата по именам вроде `Value`/`value` от регистра не зависит. Однако в «сыром» SQL, при экранированных идентификаторах и явной работе со строковыми ключами различие MSSQL/PG по регистру по-прежнему нужно учитывать.

Получение существующего скрипта — `GET /api/admin/smart/scripts/{scriptId}/editor`; сохранение — `POST /api/admin/smart/scripts/editor` (тело: `{script: <объект скрипта>, context: null}`).
 Не путать: `GET /api/admin/smart/scripts/editor?id=...` — это создание нового скрипта (возвращает пустой объект).
