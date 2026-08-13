# Опросы в комментариях

Источник: https://help.1forma.ru/domains/comments/polls/

Лёгкие голосования с одиночным или множественным выбором, встраиваемые в комментарии. Один опрос на комментарий, 1..N вопросов, каждый с набором вариантов. Поддерживаются анонимный и открытый режимы, а также множественный выбор на уровне вопроса.
 Не путать с Survey (полноценные опросы/тестирования с конструктором, таймерами, логикой ветвления) — это отдельная подсистема (`/api/survey/`).

Связи таблиц опроса:
 `Polls 1──N PollsQuestions 1──N PollsQuestionsAnswersOptions 1──N PollsResponses
  │                                                                    │
  └── CommentId (FK)                                      UserId + AnswerOptionId (PK)
  │
  └── CommentsPolls (связка comment ↔ poll, cascade delete)
`

Основная таблица опроса:
 Колонка Тип Default Описание Id INT IDENTITY PK Guid UNIQUEIDENTIFIER newsequentialid() Внешний идентификатор Title NVARCHAR(250) NULL Заголовок опроса IsAnonymous BIT 1 Анонимный (по умолчанию — да) UserId INT FK Users Создатель CommentId INT FK Comments Привязка к комментарию (UNIQUE) Color TINYINT 0 Цвет (0-11, enum Color) Индексы: PK на Id, UNIQUE на CommentId, UNIQUE на Guid, IX на UserId.
 dbo.PollsQuestions
 Колонка Тип Описание Id INT IDENTITY PK Guid UNIQUEIDENTIFIER Question NVARCHAR(100) Текст вопроса IsMultipleChoiceAllowed BIT (default 0) Можно ли выбрать несколько вариантов QuestionOrder INT Порядок отображения PollId INT FK Polls (CASCADE) Родительский опрос dbo.PollsQuestionsAnswersOptions
 Колонка Тип Описание Id INT IDENTITY PK Guid UNIQUEIDENTIFIER AnswerOption NVARCHAR(100) Текст варианта ответа AnswerOrder INT Порядок отображения QuestionId INT FK PollsQuestions (CASCADE) Родительский вопрос dbo.PollsResponses
 Колонка Тип Описание Guid UNIQUEIDENTIFIER UserId INT PK(1) FK Users Кто проголосовал AnswerOptionId INT PK(2) FK PollsQuestionsAnswersOptions (CASCADE) Выбранный вариант Составной PK (AnswerOptionId, UserId) — один голос на вариант на пользователя.
 dbo.CommentsPolls
 Связка комментарий-опрос (v2.266). Cascade delete в обе стороны.
 Колонка Тип Описание Guid UNIQUEIDENTIFIER CommentId INT PK FK Comments (CASCADE) PollId INT FK Polls (CASCADE)

Маршрут API: `api/polls`.
 Метод Маршрут Назначение Ответ GET `/api/polls/{pollId}` Получить опрос (вопросы + результаты для текущего пользователя) `ApiResultDto<PollDto>` / 404 GET `/api/polls/widgets/{blockId}/active` Получить активный опрос для виджета портала. Модератору виджета возвращает `results` всегда; обычному пользователю — только после завершения опроса `ApiResultDto<PollDto>` / 404 GET `/api/polls/widgets/{blockId}/history` Получить историю деактивированных опросов виджета. Доступно только модераторам виджета; `results` возвращается всегда `ApiResultDto<List<PollDto>>` / 404 POST `/api/polls/response` Проголосовать 204 / 400 / 404 / 409 POST `/api/polls/unvoite` Отменить голос 204 / 400 / 404 Создание опроса — не через отдельный маршрут, а в составе запроса на создание комментария.
 Body: PollResponsDto (голосование и отмена)
 `{
  "pollId": 123,
  "questions": [
    {
      "questionId": 1,
      "answersIds": [5, 7]
    }
  ]
}
` Response: PollDto
 `{
  "pollId": 123,
  "title": "Выбери день для митапа",
  "isAnonymous": false,
  "color": 5,
  "isComplete": true,
  "commentId": 456,
  "respondedUsersCount": 12,
  "questions": [
    {
      "questionId": 1,
      "question": "Какой день удобнее?",
      "isMultipleChoiceAllowed": false,
      "answers": [
        { "answerId": 5, "answerOption": "Понедельник" },
        { "answerId": 6, "answerOption": "Среда" }
      ]
    }
  ],
  "results": [
    {
      "answers": [
        { "answerId": 5, "percent": 67, "countVoites": 8, "users": [...] },
        { "answerId": 6, "percent": 33, "countVoites": 4, "users": [...] }
      ]
    }
  ]
}
` `results` = null если пользователь ещё не голосовал. В анонимном режиме `users` отсутствует, возвращается `PollAnonymousAnswersResultDto` (без списка голосовавших).
 Для модератора виджета опроса результаты возвращаются всегда — независимо от того, проголосовал ли он сам. Проверка выполняется на эндпоинтах виджета `GET /api/polls/widgets/{blockId}/active` и `GET /api/polls/widgets/{blockId}/history`; для обычных эндпоинтов опросов в комментариях поведение прежнее.
 Создание опроса (PollCreateRequestDto)
 `{
  "title": "Заголовок",
  "isAnonymous": true,
  "color": 5,
  "questions": [
    {
      "question": "Текст вопроса",
      "isMultipleChoiceAllowed": false,
      "answers": [
        { "answerOption": "Вариант 1" },
        { "answerOption": "Вариант 2" }
      ]
    }
  ]
}
` Передаётся как часть запроса создания комментария, не как отдельный API-вызов.

Уведомления
 При завершении голосования платформа рассылает обновление опроса подписчикам задачи в реальном времени — результаты обновляются без перезагрузки страницы.
 Правила (бизнес-логика)
 При голосовании:

- Нельзя голосовать дважды в одном опросе (409 Conflict)

- Все вопросы должны иметь ответы

- Вопрос с одиночным выбором — максимум 1 ответ

- AnswerIds должны принадлежать вопросам этого опроса
  На уровне БД:

- Один опрос на комментарий (UNIQUE на CommentId)

- Один голос на пользователь+вариант (PK)

- CASCADE DELETE: удаление комментария удаляет опрос и все голоса

Доступные значения поля `color`:
 Значение Имя CSS-переменная (полоска и бары результата) 0 Default `var(--base-primary)` 1 Primary `var(--base-primary)` 2 Red `var(--extramarkers-red)` 3 Pink `var(--extramarkers-pink)` 4 Purple `var(--extramarkers-purple)` 5 Green `var(--extramarkers-green)` 6 Yellow `var(--extramarkers-yellow)` 7 Orange `var(--extramarkers-orange)` 8 Blue `var(--extramarkers-blue)` 9 Cyan `var(--extramarkers-cyan)` 10 Brown `var(--extramarkers-brown)` 11 Grey `var(--extramarkers-grey)`

С 2.268.x опрос в комментариях и в ленте задач отображается в компактном виде (по аналогии с другими вложениями — ссылками и задачами): серый фон, цветная полоска слева, размер шрифта текста ответа — 13px. В лентах ширина опроса ограничена сверху 700px.
 В шаблоне опроса:

- Вариант с одиночным выбором (`isMultipleChoiceAllowed = false`) рендерится как радио-кнопка рядом с текстом ответа. Голос отправляется сразу после выбора.

- Вариант с множественным выбором (`isMultipleChoiceAllowed = true`) рендерится как чекбокс. После выбора одного или нескольких чекбоксов появляется кнопка «Голосовать».
  В отвеченном опросе бары результата располагаются под текстом каждого варианта и окрашиваются в акцентный цвет опроса; рядом с текстом выводится процент голосов и (для не-анонимных) аватары проголосовавших.
 Компонент опроса принимает входной параметр `context` с двумя значениями: `comment` (компактный вид, описанный выше — применяется в комментариях задач и в ленте) и `publication` (старый стиль с полным цветным фоном — применяется в публикациях соцсети). Цвет полоски и баров результата передаётся через CSS-переменную `--poll-accent-color`, что позволяет одному компоненту обслуживать оба сценария.

Ограничения

- Привязка к комментарию обязательна (`CommentId NOT NULL`) — автономные опросы без комментария невозможны без ALTER

- Один опрос на комментарий

- Нет редактирования опроса после создания (только удалить комментарий + создать заново)

- Текст вопроса — макс. 100 символов, вариант ответа — макс. 100 символов, заголовок — макс. 250 символов

- Нет API для получения списка всех опросов или фильтрации

- Маршрут отмены голоса содержит опечатку: `/api/polls/unvoite` (вместо `unvote`)
  Расширения
 Проект mini-poll-widget расширяет Polls для портальных виджетов: добавляет `dbo.WidgetPolls`, делает `Polls.CommentId` nullable и вводит жизненный цикл активации/деактивации.
 Для виджета опроса модератору (пользователю из групп `ModeratorGroupIds` настроек блока) результаты (`PollQuestionResultDto`) возвращаются всегда, даже если модератор ещё не голосовал — на активном опросе (`GET .../active`) и в истории деактивированных опросов (`GET .../history`, доступна только модераторам). Обычные пользователи видят результаты только после завершения опроса (`isComplete = true`). В анонимных опросах модератор, как и обычный пользователь, получает только агрегированную статистику — без списка проголосовавших.
