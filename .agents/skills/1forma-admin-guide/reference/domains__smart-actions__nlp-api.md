# NLP API в смарт-скриптах (v2.268)

Источник: https://help.1forma.ru/domains/smart-actions/nlp-api/

Нативный неймспейс `NLP` доступен в Lua, JavaScript и C# смарт-скриптах без дополнительных настроек. Python-скрипты получают URL NLP API через контекстную переменную `nlp_api_url`.
 Раньше склонение требовало ручного HTTP-запроса — теперь оно доступно напрямую через `NLP`.
 Регистр имён различается по языку. Lua — `snake_case` (`NLP.decline`, `NLP.html_to_text`); JavaScript — `camelCase` (`NLP.decline`, `NLP.htmlToText`); C# — `PascalCase` (`NLP.Decline`, `NLP.HtmlToText`). В таблицах ниже колонка «JavaScript / C#» приведена в camelCase (как в JS); в C# то же имя пишется с заглавной буквы. Источник истины: `NlpScriptApi.cs` (Lua), `JsApi/JsNlpApi.cs` (JS), `CSharpApi/CSharpNlpApi.cs` (C#).

Падежи (`case`)
 Код Падеж `"nom"` Именительный `"gen"` Родительный `"dat"` Дательный `"acc"` Винительный `"ins"` Творительный `"prep"` Предложный Тип склонения (`type`)
 Код Что склоняет `"auto"` Автоопределение (по умолчанию) `"fio"` ФИО `"noun"` Существительное / словосочетание `"title"` Должность / звание `"date"` Дата прописью `"number"` Число с единицей измерения Склонение
 Lua JavaScript / C# Описание `NLP.decline(text, case)` `NLP.decline(text, case)` Склонение с автоопределением типа `NLP.decline(text, case, type)` `NLP.decline(text, case, type)` Склонение с явным типом `NLP.decline_fio(name, case)` `NLP.declineFio(name, case)` Склонение ФИО `NLP.decline_noun(phrase, case)` `NLP.declineNoun(phrase, case)` Склонение существительного / словосочетания `NLP.decline_date(date, case)` `NLP.declineDate(date, case)` Дата прописью в нужном падеже `NLP.decline_number(num, unit, case)` `NLP.declineNumber(num, unit, case)` Число с единицей измерения в нужном падеже Числа прописью
 ⚠️ Методы `NumWords` / `NumWordsRub` / `NumWordsCurrency` в скрипт-API не зарегистрированы — их нет ни в `NlpScriptApi` (Lua), ни в `JsNlpApi` (JS), ни в `CSharpNlpApi` (C#). Не использовать до реализации; для числа с единицей в нужном падеже — `decline_number` (выше).
 Конвертация форматов
 Lua JavaScript / C# Описание `NLP.html_to_text(html)` `NLP.htmlToText(html)` HTML → plain text `NLP.html_to_markdown(html)` `NLP.htmlToMarkdown(html)` HTML → Markdown `NLP.markdown_to_html(md)` `NLP.markdownToHtml(md)` Markdown → HTML `NLP.sanitize_html(html)` `NLP.sanitizeHtml(html)` Очистка HTML от опасных тегов Спеллчек
 Lua JavaScript / C# Описание `NLP.spellcheck(text)` `NLP.spellcheck(text)` Спеллчек, язык определяется автоматически `NLP.spellcheck(text, lang)` `NLP.spellcheck(text, lang)` Спеллчек с явным языком (`"ru"` / `"en"`) Результат `spellcheck` — массив объектов `{ word, suggestions[] }`.
 ⚠️ Метода `Transliterate` нет ни в одном скрипт-API. В C# есть `NLP.EnToRu(text)` / `NLP.RuToEn(text)` (исправление раскладки между en и ru); в Lua и JavaScript их нет.

Лемматизация — приведение слова к начальной (словарной) форме: «воронки» → «воронка», «задачи» → «задача». Используется для нормализации текста перед поиском, сравнением, индексацией.
 Использует словари Hunspell (`ru_RU`, `en_US`). Индекс строится один раз при инициализации и кэшируется в памяти. Регистр первой буквы исходного слова сохраняется в результате. Неизвестные слова (нет в словаре) возвращаются без изменений.
 Lua JavaScript / C# Описание `NLP.lemmatize(word, lang)` `NLP.lemmatize(word, lang)` Начальная форма слова. Если форм несколько — возвращается первая. Параметр `lang` по умолчанию `"ru"`, поддерживается также `"en"` — `NLP.getAllLemmas(word, lang)` Все возможные начальные формы для омонимичных словоформ (например, «стали» → [«стать», «сталь»]). Возвращает массив строк. Доступно в JavaScript и C# `NLP.normalize_text(text, lang)` `NLP.normalizeText(text, lang)` Нормализация всего текста: каждое слово приводится к начальной форме, пунктуация и числа сохраняются Доступные коды языка (`lang`):
 Код Язык `"ru"` Русский (по умолчанию) `"en"` Английский

Полнотекстовый поиск по набору строк-кандидатов с учётом морфологии языка. На каждый вызов строится временный полнотекстовый индекс в памяти, выполняется поиск с учётом морфологии языка, возвращаются top-K совпадений с оценкой релевантности. Подходит для поиска по коротким спискам (названия задач, категории, компании) непосредственно в смарт-скрипте — без внешнего индекса и HTTP-запроса.
 Lua JavaScript / C# Описание `NLP.search_in_memory(query, candidates, topK, lang)` `NLP.searchInMemory(query, candidates, topK, lang)` Полнотекстовый поиск по массиву строк-кандидатов Параметры:

- `query` (string) — поисковый запрос

- `candidates` (string[]) — массив строк-кандидатов

- `topK` (int, по умолчанию 10) — максимальное количество результатов

- `lang` (string, по умолчанию `"ru"`) — язык анализатора: `"ru"` — русская морфология, `"en"` — английская, любое другое значение — базовый анализатор
  Результат — массив совпадений, отсортированный по убыванию релевантности. Каждый элемент содержит:

- `index` — позиция строки в исходном массиве `candidates`

- `text` — текст строки-кандидата

- `score` — оценка релевантности (чем выше — тем лучше)
  В Lua и JavaScript поля результата в нижнем регистре (`index`, `text`, `score`). В C# результат — `IList<InMemoryFullTextMatch>`, поля записи в PascalCase: `Index`, `Text`, `Score`.
 Примеры:
 `-- Lua — найти наиболее релевантную компанию по неполному названию
local companies = {"ООО Ромашка", "ЗАО Ромашка-Сервис", "ИП Иванов А.А."}
local results = NLP.search_in_memory("ромашка", companies, 3, "ru")

for i = 1, #results do
  local r = results[i]
  SMART.post_comment(TaskID, r.text .. " (score: " .. r.score .. ")", {})
end
` `// JavaScript — поиск по списку категорий
var categories = ["Воронка продаж B2B", "Архив сделок", "Настройка воронки в CRM"];
var results = NLP.searchInMemory("воронка продаж", categories, 5, "ru");

// results: [{index: 2, text: "Настройка воронки в CRM", score: 1.23}, ...]
` `// C# — выбор наиболее подходящего шаблона
var templates = new[] { "Договор поставки", "Договор подряда", "Акт выполненных работ" };
var matches = NLP.SearchInMemory("договор", templates, topK: 2, lang: "ru");

// matches — IList<InMemoryFullTextMatch>; поля Index, Text, Score (PascalCase)
var best = matches.Count > 0 ? matches[0].Text : null;
`

Ограничения:

- Русский анализатор — лёгкий стеммер: учитывает основные падежные и числовые окончания, но не обрабатывает беглые гласные («воронка» → найдёт «воронки», но не «воронок»).

- Индекс строится в памяти на каждый вызов — для больших массивов (тысячи строк) используйте предварительную лемматизацию с последующим точным сравнением.

- Для поиска по данным платформы (задачи, файлы, справочники) используйте AI Search, а не поиск в памяти.
  Раскладка и язык
 Lua JavaScript / C# Описание `NLP.switch_layout(text)` `NLP.switchLayout(text)` Переключение раскладки клавиатуры `NLP.detect_language(text)` `NLP.detectLanguage(text)` Определение языка (`"ru"` / `"en"`) `NLP.is_wrong_layout(text)` `NLP.isWrongLayout(text)` Эвристика: текст набран в неверной раскладке

Lua — формирование текста обращения со склонением `local fio      = SMART.get_ext_param_value(TaskID, 42)
local position = SMART.get_ext_param_value(TaskID, 43)

local text = "Уважаемый " .. NLP.decline_fio(fio, "nom") .. "!\n"
          .. "Ваша заявка передана "
          .. NLP.decline(position, "dat", "title") .. " на согласование."

SMART.post_comment(TaskID, text, {recipients = {responsible_user}})
`
 JavaScript — договор с именительным и родительным падежами `var director = SMART.getExtParamValue(TaskID, 8);
var company  = SMART.getExtParamValue(TaskID, 10);

RESULT = "Договор заключён от лица " + NLP.declineFio(director, "gen")
       + ", в интересах " + NLP.declineNoun(company, "gen") + ".";
`
 Lua — спеллчек перед отправкой комментария `local text   = SMART.get_ext_param_value(TaskID, 50)
local errors = NLP.spellcheck(text, "ru")

if #errors > 0 then
    local msg = "Обнаружены ошибки: "
    for _, err in ipairs(errors) do
        msg = msg .. err.word .. " → "
            .. table.concat(err.suggestions, "/") .. "; "
    end
    SMART.post_comment(TaskID, msg, {recipients = {owner_user}})
end
`
 JavaScript — нормализация текста перед поиском `const query = SMART.getExtParamValue(TaskID, 100);
const normalized = NLP.normalizeText(query, "ru");
// «найти все воронки продаж» → «найти весь воронка продажа»
// — устойчиво к словоформам при сравнении с эталоном
`
 JavaScript — омонимичные формы `const lemmas = NLP.getAllLemmas("стали", "ru");
// ["стать", "сталь"] — глагол и существительное
`
 Python — склонение через nlp_api_url `import requests

r = requests.post(
    nlp_api_url + "/declension",
    json={"text": ctx["name"], "case": "dative", "type": "fio"}
)
# Контроллер отдаёт стандартную обёртку ApiResultDto — результат в data
declined = r.json()["data"]["result"]
`

Группа методов исполняется локально в backend через ONNX-стек `Valhalla.Nlp` — без HTTP к `cpu-search:3010`, `mm-rerank:3009` или облачному gateway `/nli`. Подходит для closed-network стендов. Архитектура, размеры моделей, гейт concurrency, артефакты в Nexus — в `domains/ai/architecture/inprocess-onnx-stack.md`.
 Прекондиция. Методы возвращают пустой результат (no-op fallback), пока в `appsettings.json` не задан мастер-флаг и пути моделей:
 `{ "Nlp": { "VectorSearch": {
    "Enabled": true,
    "EmbedModelPath":  "/opt/1f/nlp-models/e5-base.onnx",
    "RerankModelPath": "/opt/1f/nlp-models/mmini-rerank-int8-avx512.onnx",
    "TokenizerPath":   "/opt/1f/nlp-models/sentencepiece.bpe.model",
    "NliModelPath":    "/opt/1f/nlp-models/nli/model.onnx",
    "NliTokenizerPath":"/opt/1f/nlp-models/nli/spm.model"
} } }
` ⚠️ ONNX-методы доступны только в C#-смартах. В Lua (`NlpScriptApi`) и JavaScript (`JsNlpApi`) они не зарегистрированы — вызов вернёт `nil` / `undefined`.
 Lua JavaScript C# Описание — — `NLP.EmbedQuery(text)` Эмбеддинг одного query-текста, e5-base 768D, L2-нормирован → `cosine == dot` — — `NLP.EmbedPassagesBytes(string[])` Батч-эмбеддинг с возвратом байт-формата (LE `<f4`, 3072 байта/вектор) для записи в `DocsChunks.Embedding` — — `NLP.RerankTexts(query, docs, topN)` Кросс-энкодер реранк mMiniLM, возвращает `topN` пар `(index, score)` отсортированных по убыванию — — `NLP.SearchDocsVector(query, k)` Vector-поиск по корпусу `DocsChunks` через `DocsVectorSearchRouter` (`pgvector` на PG / `inmemory` на MSSQL) — — `NLP.NliClassify(premise, hyp)` 3-class NLI на mDeBERTa-v3, результат `{entailment, neutral, contradiction}` (softmax, сумма ≈ 1) — — `NLP.NliClassifyBatch(pairs)` Батч-вариант NLI; `pairs` — массив пар `(premise, hypothesis)` Примеры ниже — на C#, так как ONNX-методы доступны только в C#-смартах.
 C# — батч-эмбеддинг с записью в `DocsChunks.Embedding` `var chunks = new[] { "Глава 1. Введение...", "Глава 2. Установка..." };
var blobs  = NLP.EmbedPassagesBytes(chunks); // byte[][] — по 3072 байта на вектор
// blobs[i] записывается в DocsChunks.Embedding (varbinary) для соответствующего чанка
`

Смежные разделы:

- Jint — JS-интерпретатор

- Python в смарт-скриптах

- C# в смарт-скриптах (Roslyn)

- In-process ONNX-стек NLP (`Valhalla.Nlp`) — архитектура моделей, настройки `Nlp:VectorSearch:*`, артефакты в Nexus
