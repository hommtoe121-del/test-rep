# Синхронизация с ERP CATI

Инструкция по автоматической **отправке данных БЦ в ERP CATI** из 1Формы (фильтр, правило, смарт-расписание, HTTP-пакет, ручной перезапуск), разобранной на записи ВКС с таймкода **36:57**.

**Источник:** запись ВКС от 29.07.2026 (фрагмент с 36:57)  
https://vks.oro.moscow/records/3f9b19c6-daa7-46c3-a583-b04e468fe89b/9f10fczreo5gnbdf_2026-07-29-16-47-43.mp4

**Системы:**
- 1Форма: https://1forma.oro.moscow
- Автоматизация БЦ (65): https://1forma.oro.moscow/spa/administration/smart-packs/65?smartGridTab=smartPacksOnEvents
- Смарт-расписания БЦ: https://1forma.oro.moscow/spa/administration/smart-packs/65?smartGridTab=smartRecurrences
- HTTP endpoint (как на экране действия): https://backend-dev.oro.moscow/folder-access-controller/integration/erp

> На той же встрече рядом разобран сервис **Qual** (пакет `Отправить HTTP qual-intergation-adm`). Ниже — только **CATI**; Qual устроен по той же схеме (правило + расписание + фильтр), но с другим методом сбора и пакетом.

---

## 1. Общая схема

| Элемент | Значение (с записи / UI) |
|---|---|
| Категория | **Бизнес-циклы (65)** |
| Фильтр | **`cati filter`** |
| Основной триггер | **После перехода** → **Перевести в проект** |
| Правило | ID **506** → пакет **CATI ERP Отправка api** |
| Страховка | Смарт-расписание ID **84** (фильтр `cati filter`, тот же пакет) |
| Ручной перезапуск | ДП **Комментарий 13 (375)** + нужный пакет/фильтр |

Администрирование → Правила:

![](https://raw.githubusercontent.com/wiki/hommtoe121-del/test-rep/media/sincronizaciya-erp-cati/09-admin-rules-tab.jpg)

---

## 2. Где смотреть

1. Откройте https://1forma.oro.moscow/spa/administration/smart-packs/65?smartGridTab=smartPacksOnEvents  
2. В колонке **Smart-фильтр** введите `cati`.  
3. Найдите правило **506**.

![](https://raw.githubusercontent.com/wiki/hommtoe121-del/test-rep/media/sincronizaciya-erp-cati/01-rules-cati-filter.jpg)

На экране:
- Событие: **После перехода**
- Параметр: **Перевести в проект**
- Smart-фильтр: **cati filter**
- Пакет: **CATI ERP Отправка api**

Группа «Маршруты, переходы»:

![](https://raw.githubusercontent.com/wiki/hommtoe121-del/test-rep/media/sincronizaciya-erp-cati/10-rule-after-transition.jpg)

---

## 3. Фильтр `cati filter`

Выражение (SMART) открывается из правила / списка выражений. ID выражения на записи: **1034**.

![](https://raw.githubusercontent.com/wiki/hommtoe121-del/test-rep/media/sincronizaciya-erp-cati/03-cati-filter-expression.jpg)

Условия (как на экране / в речи): фильтр даёт `true`, если одновременно:

1. **Номер бизнес-цикла** > 0  
2. **Номер договора** есть значение  
3. **Наименование** есть значение  
4. **Начало проекта** есть значение  
5. **`СтрокаПуста( API cati ответ )`** — ответ CATI ещё пустой (ещё не отправляли / не записали)  
6. **Окончание проекта** есть значение  
7. **Руководитель проекта** есть значение  
8. **Основной метод сбора данных.**№ задачи = **1140**  
   **ИЛИ** в **Дополнительные методы сбора данных** есть элемент с № задачи = **1140**  

**Устно:** 1140 — это справочное значение метода сбора **«CATI»**. Если метод не CATI (например, фокус-группа / другое значение), фильтр не пройдёт и в ERP ничего не уйдёт — это штатно, а не «падение» интеграции.

**Устно:** по самому фильтру нельзя увидеть «что именно не заполнено» — нужно сверять поля БЦ с условиями по очереди.

---

## 4. Пакет **CATI ERP Отправка api**

Состав пакета (tooltip на правиле 506):

![](https://raw.githubusercontent.com/wiki/hommtoe121-del/test-rep/media/sincronizaciya-erp-cati/02-pack-cati-erp-tooltip.jpg)

| # | Действие | Назначение |
|---|---|---|
| 724 | **Отправить HTTP запрос** | Вызов ERP integration |
| 725 | **Изменить значение ДП** | служебная запись (на экране — пользователь/systemrobot) |
| 726 | **Изменить значение ДП** | запись ответа в ДП **Ответ api QUAL и ERP (494)** |

### HTTP-действие

![](https://raw.githubusercontent.com/wiki/hommtoe121-del/test-rep/media/sincronizaciya-erp-cati/04-http-erp-integration.jpg)

- **Метод:** `POST`  
- **URL (как в UI на записи):** https://backend-dev.oro.moscow/folder-access-controller/integration/erp  
- **Тело запроса:** оставить пустым (параметры уходят списком)  

Параметры (Smart/TSQL → поле БЦ):

| Параметр API | Источник в 1Форме |
|---|---|
| `job_id` | Номер БЦ |
| `contract` | Договор 1С |
| `job_name` | Имя бизнес-цикла |
| `start_date` | Начало проекта |
| `end_date` | Окончание проекта |
| `project_manager` | РП (имя) |

**Устно:** в ERP уходит job id, номер БЦ, номер договора, имя БЦ, начало/окончание проекта. Если какое-то из полей фильтра/параметров пустое или неверное — данные в сервис CATI не попадут.

> URL на скрине действия — `backend-dev.oro.moscow`. Админка открыта на https://1forma.oro.moscow; для боя уточните у владельцев интеграции боевой host, если он отличается.

---

## 5. Смарт-расписание (повторная проверка)

Помимо перехода «в проект», есть периодический прогон с тем же фильтром и пакетом — на случай, если пользователь **дозаполнил** поля уже после перевода в проект.

Список расписаний: https://1forma.oro.moscow/spa/administration/smart-packs/65?smartGridTab=smartRecurrences

![](https://raw.githubusercontent.com/wiki/hommtoe121-del/test-rep/media/sincronizaciya-erp-cati/05-smart-schedules-cati.jpg)

![](https://raw.githubusercontent.com/wiki/hommtoe121-del/test-rep/media/sincronizaciya-erp-cati/06b-schedule-cati-modal.jpg)

На записи видно расписание **ID 84**:
- фильтр: задачи по **`cati filter`**
- пакет: **CATI ERP Отправка api**
- активно

**Устно:** периодичность поставили **каждые 240 минут** (не каждые 5 — «это перебор»). Рядом по той же схеме крутится Qual (пример модалки расписания с интервалом 240 мин и «только в рабочее время»):

![](https://raw.githubusercontent.com/wiki/hommtoe121-del/test-rep/media/sincronizaciya-erp-cati/06-schedule-qual-example.jpg)

Итого **два триггера** для CATI:
1. сразу после перехода **Перевести в проект** (если фильтр true);  
2. позже — смарт-расписание, если поля дозаполнили.

---

## 6. Если «в CATI ничего не пришло»

Типовые причины (со записи):

1. Не выполнен фильтр `cati filter` (часто: не CATI в методе сбора / пустые договор, даты, РП, наименование).  
2. В поле **API cati ответ** уже что-то есть → `СтрокаПуста(...)` = false, повторно не шлёт.  
3. Сбой на стороне 1Формы или ERP (реже).  

### Диагностика

1. Откройте БЦ, пройдите условия фильтра сверху вниз.  
2. Проверьте метод сбора = **1140 (CATI)** (основной или доп.).  
3. Посмотрите ДП ответа API (на пакете фигурирует **Ответ api QUAL и ERP (494)** / «API cati ответ» в фильтре).  
4. При необходимости очистите ответ и перезапустите отправку вручную (ниже).

### Ручной перезапуск через Комментарий 13

Паттерн тот же, что у других интеграций БЦ:

1. Админка → БЦ (65) → **Правила**, фильтр по параметру **Комментарий** / **Комментарий 13**.  
2. Для нужного правила выставить пакет **CATI ERP Отправка api** (или временно нужный) + фильтр, **Активна**.  
3. В карточке БЦ изменить **Комментарий 13 (375)**.  
4. Проверить, что в ответе API появился успешный статус (на демо Qual показывали HTTP **200**).

Пример модалки правила на Комментарий 13:

![](https://raw.githubusercontent.com/wiki/hommtoe121-del/test-rep/media/sincronizaciya-erp-cati/07-comment13-rule.jpg)

![](https://raw.githubusercontent.com/wiki/hommtoe121-del/test-rep/media/sincronizaciya-erp-cati/08-comment13-qual-pack.jpg)

**Устно:** Комментарий 13 — это **ручной триггер** «выполнить выбранный пакет», а не отдельная бизнес-логика CATI. На одном ДП можно переключать разные пакеты (CATI / Qual / folder-adm и т.д.).

---

## 7. Чек-лист поддержки

- [ ] В БЦ метод сбора данных = CATI (**1140**)  
- [ ] Заполнены: договор, наименование, начало/окончание, РП, номер БЦ  
- [ ] Поле ответа CATI/ERP пустое, если нужна повторная отправка  
- [ ] Правило **506** активно (после «Перевести в проект» + `cati filter`)  
- [ ] Расписание **84** активно  
- [ ] При ручном рестарте: правило на **Комментарий 13** активно, значение ДП изменено  
- [ ] В ответе API ожидаемый HTTP-статус (например 200)

---

## 8. Ссылки

- Запись ВКС (с 36:57): https://vks.oro.moscow/records/3f9b19c6-daa7-46c3-a583-b04e468fe89b/9f10fczreo5gnbdf_2026-07-29-16-47-43.mp4  
- 1Форма: https://1forma.oro.moscow  
- Правила БЦ (65): https://1forma.oro.moscow/spa/administration/smart-packs/65?smartGridTab=smartPacksOnEvents  
- Расписания БЦ (65): https://1forma.oro.moscow/spa/administration/smart-packs/65?smartGridTab=smartRecurrences  
- HTTP ERP (как в действии пакета на записи): https://backend-dev.oro.moscow/folder-access-controller/integration/erp  
- Выражение `cati filter` (из URL на записи): https://1forma.oro.moscow/spa/administration/smart-expressions?SubcatId=65&ID=1034  
- HTTP-действие пакета (из URL на записи, ID=496): https://1forma.oro.moscow/spa/administration/smart-pack-action?SubcatId=65&ID=496  
