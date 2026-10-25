# Dual Review для Claude Code

[English version](README.md)

`/dual-review` — команда код-ревью для [Claude Code](https://code.claude.com).
Два независимых ревьюера смотрят твоё изменение параллельно, каждый в новой
сессии. Их находки сводятся в один нумерованный список, и свежие ревьюеры
перепроверяют его раунд за раундом, пока оба не согласятся с каждым фиксом.
С флагом `--fix` согласованные фиксы применяются, и ещё два новых ревьюера
проверяют результат.

Находка попадает в список, только если две независимые сессии согласны, что
проблема реальная. Поэтому ложных срабатываний меньше, чем у ревью в один
проход.

## Установка

Одинаково на macOS, Linux и Windows. Нужны Claude Code и `git`.

В сессии Claude Code выполни:

```
/plugin marketplace add Gmust/dual-review
/plugin install dual-review@dual-review
```

Когда спросит scope, выбери **user**, тогда команда будет работать во всех
проектах. Потом выполни `/reload-plugins` или перезапусти Claude Code.

То же самое из терминала:

```bash
claude plugin marketplace add Gmust/dual-review
claude plugin install dual-review@dual-review
```

В десктопном приложении Claude или в расширении для VS Code добавь
маркетплейс `Gmust/dual-review` в браузере плагинов и установи
**Dual Review**.

**Обновить:** `/plugin marketplace update dual-review`. Или один раз включи
автообновление: `/plugin` → **Marketplaces** → `dual-review` →
**Enable auto-update**.

**Удалить:** `/plugin uninstall dual-review@dual-review`

## Использование

Открой Claude Code в git-репо, на ветке, которую хочешь проверить:

```
/dual-review                         # ветка против основной + незакоммиченные изменения
/dual-review main                    # против конкретной базы
/dual-review --staged                # только staged
/dual-review #482                    # PR на GitHub (лучше сначала gh pr checkout 482)
/dual-review --fix                   # ревью, применить согласованное, проверить
/dual-review --depth deep            # глубина вручную: quick | standard | deep
/dual-review --security              # добавить security-ревьюера (на deep включён сам)
/dual-review --rounds 5              # свой лимит раундов
/dual-review main проверь логику refresh-токена   # любой текст — подсказка ревьюерам
```

Если имя `/dual-review` уже занято другой командой, используй полное
`/dual-review:dual-review`.

Отчёт начинается со счёта, например `To fix: 2 (1 blocker, 1 major)`, или
прямо пишет `Nothing to fix.` Дальше идут разделы:

- **Agreed fixes**: проблему и фикс подтвердили оба ревьюера.
- **Deadlocked**: не договорились. Тебе показывают аргументы обеих сторон,
  решаешь ты.
- **Suggestions**: мелочи без консенсуса.
- **Pre-existing**: реальные баги, которые не этот PR внёс. Показываются,
  но без твоей команды не чинятся.
- **Rejected**, **Nits** и **Prevent recurrence**: lint-правило, тест или
  строка в CLAUDE.md, которые поймали бы этот баг.

Без `--fix` скилл спрашивает, прежде чем что-то менять. Фиксы не
коммитятся, посмотреть их можно через `git diff`.

## Как это устроено

- Ревьюеры читают намерение изменения: PR, связанный issue (из `Fixes #N`
  или имени ветки вроде `fix/123-…`) и коммиты. Проверяют, что изменение
  делает то, что обещает, а фикс бага устраняет причину, а не прячет
  симптом.
- Учитываются `CLAUDE.md` и `REVIEW.md` в корне репо, если он есть.
- Глубина выбирается по тому, что меняется. Quick для документации и
  тестов, deep для auth, платежей, ключей, смарт-контрактов, миграций и CI.
- На deep или с `--security` подключается ревьюер S, который ищет только
  уязвимости. Его находки проверяют голосованием A и B.
- Начиная со второго раунда, новые находки принимаются, только если это
  blocker или major, внесённый этим изменением. Minor получает одно
  голосование и потом уходит в предложения.

Подробности, таблица глубины, настройка через `REVIEW.md` и установка без
плагинов описаны в [английском README](README.md).

## Без системы плагинов

Скачай репо (**Code** → **Download ZIP**), распакуй и запусти:

macOS / Linux: `bash install.sh`

Windows (PowerShell):
`powershell -ExecutionPolicy Bypass -File .\install.ps1`

После первой установки перезапусти Claude Code. Не ставь двумя способами
сразу.

## В CI

В фоновых запусках (`claude -p`, GitHub Actions) выставляй
`CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1`.

## Лицензия

[MIT](LICENSE)
