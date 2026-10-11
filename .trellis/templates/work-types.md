# Work Type Workflows (feat / issue / refactor)

Три универсальных воркфлоу (из FTM/CodeStable, поставляются с плагином trellis-workflow, MIT).
**Носитель**: `.trellis/tasks/{slug}/`
**Вход**: через таблицу маршрутизации воркфлоу, с переиспользованием нативных Trellis skill (`trellis-feat` / `trellis-issue` / `trellis-refactor`)
**Шаблоны**: `.trellis/templates/{feat,issue,refactor}/` (инициализируются автоматически соответствующим навыком при первом использовании)

> **Строгий формат slug** (плагин проверяет и предупреждает в breadcrumb / `trellis_state`):
> `<work-type>-<mm-dd>-<короткое имя>`, например `feat-01-15-billing-export`. `<mm-dd>` — дата создания,
> `<work-type>` берётся из `work.type` (feat|issue|refactor). Несоответствующие имена каталогов дают предупреждение.

## Общие правила

1. Новая работа — только Trellis task, все артефакты кладутся в `.trellis/tasks/`.
2. **Создание task — встроенным инструментом `trellis_task_create`**: он пишет `.trellis/tasks/<slug>/task.json`
   (status=planning), сеет шаблоны артефактов и **синхронно пишет `current_task` в
   `.trellis/.runtime/sessions/`**, чтобы breadcrumb/фаза/Web-бейдж заработали сразу. Не создавайте каталоги вручную и не пишите session-файлы руками.
3. **Обновление статуса и фазы task — встроенным инструментом `trellis_task_update`**: безопасно меняет `status`, `stage`, `mode`, `title` и `description`, проверяет трек воркфлоу и обновляет кэш Web-бейджа (без `slug` действует на активную задачу текущей сессии).
4. Структура расширенных полей `task.json` (поддерживается `trellis_task_create` / `trellis_task_update`, не ломает нативный status):
   ```json
   {
     "work": {
       "type": "feat",
       "mode": "standard",
       "stage": "design",
       "execution_lane": "standard"
     }
   }
   ```
5. Нативный `status` по-прежнему только: `planning` → `in_progress` → `completed` (archive).
6. **Архивация — встроенным инструментом `trellis_task_archive`**: атомарно переносит `.trellis/tasks/<slug>/`
   в `.trellis/tasks/archive/<yyyy-mm>/<slug>/` — ключ месяца `<yyyy-mm>` = `mm` из slug + текущий год
   (единая логика с чтением канбана; legacy-slug без `mm-dd` попадает в `other/`), и отвязывает
   указатели сессий (архивные задачи только для чтения и не активируются). Архив **только перемещает, не удаляет** записи.
   Если в рабочей области есть несвязанные с задачей незакоммиченные изменения, можно передать `modified_files`
   — тогда Git-проверка будет локальной (файлы из списка должны быть закоммичены; остальное пропускается с предупреждением).
7. Мелкие фазы восстанавливаются по `work.stage` + файлам артефактов; **артефакты репозитория главнее истории чата**.
8. Перед кодом читайте `.trellis/spec/` (соответствующий слой + направляющие `guides/`).
9. Планирование/уточнение требований — `trellis-brainstorm`; перед реализацией — `trellis-before-dev`; качественный гейт — `trellis-check`;
   завершение — нативный жизненный цикл Trellis (commit → `trellis-finish-work` / `trellis_task_archive`); накопление знаний — `trellis-update-spec`.
10. Пока человеческий барьер не пройден, запрещено писать `status=in_progress` (или входить в фазу кода).
11. **Стандарт slug**: имя каталога новой задачи обязано быть `<work-type>-<mm-dd>-<короткое имя>` (например `feat-01-15-billing-export`,
    mm-dd — дата создания); плагин проверяет в breadcrumb / `trellis_state` каждый раунд и предупреждает при несоответствии.

## Маршрутизация

| Намерение пользователя | Входной skill | Описание | Префикс slug |
|------------------------|---------------|----------|--------------|
| Новый функционал / переделка | `trellis-feat` | prd→design→design-review→impl→review→check (quick/standard) | `feat-<mm-dd>-...` |
| Баг / аномалия / регресс | `trellis-issue` | report→analyze→fix→fix-note; при зацикливании — `trellis-break-loop` | `issue-<mm-dd>-...` |
| Эквивалентный рефакторинг / разделение / оптимизация | `trellis-refactor` | scan→design→apply; изменение поведения — перевод в feat/issue | `refactor-<mm-dd>-...` |
| Начало сессии / восстановление контекста | Нативный Trellis (Active Task) | `trellis_state` / `trellis-continue` | — |
| Уточнение требований (планирование feat/issue) | `trellis-brainstorm` | загружается на этапе планирования | — |

## Фазы Feature

`mode/lane`: `quick` | `standard` (для долгих целей — parent+child task)

| stage | Артефакт | Человеческий барьер |
|-------|----------|---------------------|
| prd | `prd.md` | В quick — легче |
| design | `design.md` (с планом проверки и рисками/откатом) | Standard: approve пользователем |
| design-review | `design-review.md` (отдельный субагент) | Standard: reviewer passed |
| impl | Код; список шагов — `task.json.steps` | Сначала `status=in_progress`; шаги `verification: human` подтверждаются пользователем |
| review | `review.md` (отдельный субагент) | blocking — исправить; опционально `trellis-check` для самостоятельного ремонта |
| check | Результаты `trellis-check` и доказательства проверки (пишутся в `review.md`) | После прохождения — archive |

Quick: если `prd` ясен и локален, без новых межсистемных контрактов — design-review можно пропустить; всё же рекомендуются минимальный prd + check после реализации.

## Фазы Issue

| stage | Артефакт | Человеческий барьер / пропуск |
|-------|----------|-------------------------------|
| report | `report.md` | |
| analyze | `analysis.md` | Можно пропустить, если причина очевидна |
| fix | Код | После записи `status=in_progress` |
| fix-note | `fix-note.md` | Обязателен перед archive |

## Фазы Refactor

Нижняя черта: **эквивалентность поведения**. Меняет наблюдаемое поведение → переводить в feat/issue.

| stage | Артефакт | Человеческий барьер |
|-------|----------|---------------------|
| scan | `scan.md` | Пользователь отмечает включённые пункты |
| design | `refactor-design.md` (шаги вносятся на планировании в `task.json.steps`) | Approve пользователем |
| apply | Код + `apply-notes.md`; шаги ведутся по `task.json.steps` | Шаги `verification: human` требуют подтверждения |
| done | Полная проверка | После финального подтверждения пользователя — archive |

`mode: fastforward`: объём крошечный, откат лёгкий, поведение эквивалентно; можно сжать scan/design, но apply-notes + check обязательны.

## Соответствие фаз нативному Trellis

| Trellis status | Типичный work.stage |
|----------------|---------------------|
| planning | feat: prd/design/design-review; issue: report/analyze; refactor: scan/design |
| in_progress | feat: impl/review/check; issue: fix/fix-note; refactor: apply |
| completed | archived (`.trellis/tasks/archive/<yyyy-mm>/<slug>`) |

## Субагенты

| Роль | Механизм запуска | Назначение |
|------|------------------|------------|
| Ревью дизайна | основной сессией отдельный субагент (`trellis-channel` или subagent) | Ревью дизайна → `design-review.md` |
| Ревью кода | основной сессией отдельный субагент | Ревью кода → `review.md` |
| Реализация | субагент реализации (опционально) | По prd/design |
| Самопроверка/ремонт по спецификации | `trellis-check` | Сверка со спецификацией + ремонт/проверка |

Протокол отправки: каждый prompt субагента начинается с `Active task: <путь каталога задачи>`; если reviewer вернул `blocking` — остановка на текущем stage.
