# FastReport — Настройка и решение проблем

Источник: https://help.1forma.ru/domains/reports/support-guide-fastreport/

Назначение: ответы на типовые вопросы по настройке FastReport и работе с отчётами. Связан с: администрированием отчётов.

FastReport — движок генерации отчётов в 1Форме. Пользователь запускает отчёт через веб-интерфейс, API вызывает FastReport Engine, который выполняет SQL-запрос и формирует документ.

 `Пользователь → Веб-интерфейс → API (reports/view/{id}) → FastReport Engine → SQL Server / PG → Готовый отчёт
                                                                ↑
                                                    Reports.ContentXml (шаблон)
` 1Ф поддерживает два способа создания отчётов через FastReport:

- Win-дизайнер (десктопное приложение) — полный функционал, требует установки

- Онлайн-дизайнер (в браузере) — ограниченный функционал, не требует установки

Что нужно для работы Win-дизайнера:

- FastReport Designer — поставляется в ZIP-архиве из категории «Репозиторий версий приложения» (не скачивается из интерфейса)

- .NET Framework 4.7.1 или выше — скачать

- Сетевой доступ к SQL Server — дизайнер подключается напрямую к БД
  Ресурс Порт Зачем SQL Server 1Ф 1433 (TCP) Дизайнер подключается к БД для получения данных Сервер 1Ф (web) 443/80 Для сохранения шаблона обратно в 1Ф Дизайнер использует connection string из 1Ф. Два варианта:

- Windows Authentication — учётная запись Windows пользователя должна иметь доступ к БД 1Ф (`db_datareader` минимум)

- SQL Authentication — логин/пароль SQL Server (задаётся в connection string отчёта)
  Важно: учётная запись дизайнера должна иметь:

- `SELECT` на таблицы, которые используются в отчётах

- `EXECUTE` на хранимые процедуры, если отчёт их вызывает

- Рекомендуемая роль: `db_datareader` + точечные `EXECUTE` на нужные SP

Администрирование → Отчёты → Создать отчёт
 Поле Описание Название Отображаемое имя отчёта Режим `FastReports` (для FastReport-шаблонов) Контекст Откуда доступен: Задача, Категория, Раздел, Пользователь, Группа Блок Группировка в интерфейсе Скрытый Если `true` — не показывается пользователям Администрирование → Отчёты → [отчёт] → Права
 Права настраиваются по группам:
 Право Описание Просмотр Кто видит отчёт в списке и может генерировать Аннотация Кто может редактировать описание Специальные Полный доступ (администратор отчёта) Если права не назначены — отчёт виден только администраторам.
 Контекст определяет, какие параметры автоматически доступны в шаблоне:
 Контекст Автопараметр Где отображается Задача `taskId` В карточке задачи Категория `categoryId` В гриде категории Раздел — В разделе Пользователь `userId` В профиле пользователя Группа `groupId` В настройках группы Параметр `CurrentUserId` доступен всегда — ID текущего пользователя.
 Фильтры позволяют пользователю задавать параметры перед генерацией. Настраиваются в Администрирование → Отчёты → [отчёт] → Фильтр.

Порядок работы с Win-дизайнером:

- Распаковать ZIP-архив дизайнера

- Запустить `TCReportDesigner.exe`

- В списке отчётов выбрать нужный → «Win-дизайнер»

- Дизайнер скачает шаблон и откроется
  В дизайнере доступны:

- Таблицы БД — прямой доступ к таблицам 1Ф

- SQL-запросы — произвольные SELECT

- Хранимые процедуры — с параметрами

- Несколько Connection — можно подключить дополнительные БД
  Connection string подставляется автоматически из настроек 1Ф. Таймаут SQL-команд: 120 секунд (настройка `FastReportCommandTimeoutSeconds`).
 При сохранении дизайнер отправляет XML-шаблон обратно в 1Ф через маршрут `~/MVC/ReportDesigner/Save`. Шаблон записывается в `Reports.ContentXml`.

Онлайн-дизайнер доступен из интерфейса 1Формы (Отчёты → [отчёт] → Онлайн-дизайнер) и требует права администратора. Ограничения по сравнению с Win-дизайнером: нет полного набора компонентов, нет прямого доступа к connection strings, не все форматы экспорта доступны.

FastReport поддерживает экспорт отчётов в несколько форматов: PDF, Word, Excel, PowerPoint, RTF и другие.
 Формат Расширение Особенности PDF .pdf Основной формат. Встраивание шрифтов, сжатие Word .docx Несколько режимов: WYSIWYG, постраничный, абзацный Excel .xlsx Режимы: постраничный, бесшовный, только данные PowerPoint .pptx Изображения в PNG или JPEG RTF .rtf С разрывами страниц HTML, CSV, XML, TXT, DBF, ODS, ODT — Дополнительные форматы



Симптом: при открытии Win-дизайнера ошибка подключения к SQL Server.
 Проверить:

- Сеть: с компьютера пользователя `telnet sql-server 1433` — порт должен быть открыт

- Учётная запись: если Windows Auth — пользователь должен быть в домене и иметь доступ к БД. Если SQL Auth — проверить логин/пароль

- Firewall: SQL Server может слушать на нестандартном порту или блокировать внешние подключения

- Named Pipes vs TCP/IP: проверить что TCP/IP включен в SQL Server Configuration Manager
  Проверить:

- `Reports.IsHidden` — не скрыт ли отчёт?

- Права — назначены ли группе пользователя права на просмотр?

- Контекст — если отчёт привязан к контексту (задача/категория), он отображается только в этом контексте

Проверить:

- SQL-запросы в шаблоне — оптимизировать (добавить индексы, упростить)

- Таймаут — по умолчанию 120 секунд. Увеличить `FastReportCommandTimeoutSeconds` в конфигурации

-  Объём данных — при большом объёме использовать фильтры для ограничения выборки


-  Win-дизайнер доступен только для отчётов с `Mode = FastReports`


- Требуется роль администратора
  Дизайнер поставляется в ZIP-архиве из категории «Репозиторий версий приложения» в 1Ф. Ссылка для скачивания из интерфейса убрана.

Основные поля таблицы `Reports`:
 Поле Тип Описание `id` int PK `Description` varchar(500) Название отчёта `ContentXml` varchar(max) XML-шаблон FastReport `Context` int Тип контекста (0=Task, 1=Subcat, 2=Category, 3=User, 4=Group) `Mode` int Режим (0=ExtForms, 2=FastReports) `IsHidden` bit Скрытый `IsSystemReport` bit Системный `ExternalObjectId` uniqueidentifier FK на ExternalObjects (права) `UserID` int Владелец `Block` varchar(200) Группа в интерфейсе `ExportSettings` nvarchar(max) JSON настроек экспорта `GUID` uniqueidentifier Уникальный ID Таблица Описание `ReportContextObjects` Привязка отчёта к контексту (категория, задача и т.д.) `ReportFilterLinks` Привязка фильтров к отчёту `ExternalObjects` + `ExternalObjectsViewRights` Права доступа к отчёту

Готовые запросы для диагностики отчётов:
 `set transaction isolation level read uncommitted;

-- Все FastReport-отчёты
select
        r.id,
        r.Description,
        r.Context,
        r.IsHidden,
        r.IsSystemReport,
        r.UserID,
        u.FullName as Владелец,
        len(r.ContentXml) as ШаблонSize
from dbo.Reports r with (nolock)
        left join dbo.Users u with (nolock)
            on u.UserID = r.UserID
where r.Mode = 2
order by r.Description;

-- Права на отчёт
select
        r.Description as Отчёт,
        g.Name as Группа,
        eovr.RightType
from dbo.Reports r with (nolock)
        join dbo.ExternalObjects eo with (nolock)
            on eo.GUID = r.ExternalObjectId
        join dbo.ExternalObjectsViewRights eovr with (nolock)
            on eovr.ExternalObjectId = eo.Id
        join dbo.Groups g with (nolock)
            on g.GroupID = eovr.GroupID
where r.id = @ReportId
order by g.Name;

-- Контекст отчёта
select
        r.Description as Отчёт,
        rco.ContextObjectType,
        rco.ContextObjectId
from dbo.Reports r with (nolock)
        join dbo.ReportContextObjects rco with (nolock)
            on rco.ReportId = r.id
where r.id = @ReportId;
`

REST API для управления отчётами и генерации: просмотр, создание, обновление, копирование и удаление отчётов через административные маршруты.
 Маршрут Метод Описание `reports/view/{id}` GET/POST Генерация отчёта `reportdesigner/{id}` GET Открытие Win-дизайнера `api/admin/reports` GET Список отчётов (админ) `api/admin/reports` POST Создание отчёта (админ) `api/admin/reports/{id}` POST Обновление отчёта (админ) `api/admin/reports/{id}` DELETE Удаление отчёта (админ) `api/admin/reports/{id}/copy` POST Копирование отчёта (админ)

FastReport поддерживает 4 режима экспорта в Excel, настраиваемые через ExportSettings отчёта (API: `PUT /app/v1.1/api/reports/{id}/exportSettings`):
 Режим Поле настроек Поведение Wysiwyg `Wysiwyg: true` Точное визуальное воспроизведение. FR воссоздаёт позиции элементов через объединённые ячейки. Дефолт DataOnly `DataOnly: true` Только данные без форматирования. Без объединённых ячеек. Подходит для дальнейшей обработки Seamless `Seamless: true` Без разрывов страниц, непрерывная таблица PrintOptimized `PrintOptimized: true` Оптимизация для печати (отступы, масштаб) Причина: режим Wysiwyg (по умолчанию). FastReport позиционирует каждый элемент шаблона абсолютно (X, Y, Width, Height). При экспорте в XLSX координаты конвертируются в ячейки сетки. Если элементы не выровнены по единой сетке — FR создаёт дополнительные колонки/строки и объединяет ячейки для сохранения визуального расположения.
 Решения:

- DataOnly=true — убирает все объединённые ячейки. Подходит если нужны только данные

- Выравнивание элементов в шаблоне — в FR Designer включить привязку к сетке, выровнять все элементы по одной сетке колонок. Чем меньше уникальных X-координат — тем меньше объединённых ячеек

- Table-объект вместо свободных Band-элементов — Table в FR не создаёт лишних колонок

- SplitPages=false — если объединённые ячейки появляются на границах страниц
  `{
  "xlsxExportSettings": {
    "wysiwyg": true,
    "pageBreaks": false,
    "splitPages": false,
    "dataOnly": false,
    "seamless": false,
    "printOptimized": false
  }
}
` Таблица БД: `Reports.ExportSettings` (NVARCHAR MAX, JSON). Миграция v2.178.
