# Agent Harness Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Собрать и опубликовать публичный репозиторий `agent-harness` с переиспользуемым ядром AI-харнесса и `INSTALL.md`, по которому агент в целевом проекте сам накатывает харнесс на себя.

**Architecture:** Плоское ядро в корне (AGENTS.md, .agents/, opencode.jsonc, docs/, scripts/) плюс `adapters/` для опциональных харнессов. Контент копируется из `1pg_gall` и адаптируется: стек-нейтральность, скруб личных данных, лицензионные нотисы. Своего кода нет — проверки состоят из `grep`, парсинга JSONC/YAML и подсчёта файлов.

**Spec:** `docs/design/2026-11-12-agent-harness-design.md`

**Источник контента:** `C:\My_projects_GIT\1pg_gall` (далее — «источник»).
**Рабочая директория:** `C:\My_projects_GIT\agent-harness` (далее — «репо»).

## Global Constraints

- Репозиторий публичный: `frostiks777/agent-harness`, ветка `main`.
- Ядро стек-нейтрально: `AGENTS.md`, `opencode.jsonc`, `.agents/shared/checks.md`, `docs/agents/*` не содержат `bun`, `npm`, `next`, `react`, `eslint`, `playwright`, `tsc`, `shadcn`, `Java` как данность.
- Скиллов ровно **84**: `apply-design` и `apply-design-v2` исключены.
- Личные данные и секреты не попадают: `95.81.124.37`, `duckdns`, `pgagent`, `1pg_gall`, `1pgWebGallery`, `frostiks777/1pg`, `Фото-галерея`, `.env`, `*.jsonl`, приватные ключи.
- `LICENSE` — MIT © 2026 frostiks777. `THIRD-PARTY-NOTICES.md` обязателен.
- Коммиты — Conventional Commits; один task = минимум один commit.
- Плейсхолдеры `<!-- заполни: ... -->` разрешены **только** в шаблонах (`AGENTS.md`, `.agents/shared/checks.md`, `docs/agents/*`); в `README.md`, `INSTALL.md`, `docs/updating.md`, `THIRD-PARTY-NOTICES.md` их быть не должно.

## Review Focus

1. **В целевом проекте уже есть харнесс** (`AGENTS.md`, `CLAUDE.md`, `.agents/`) → установка обязана интегрировать, а не перезаписывать. Закрепляется текстом INSTALL (Task 11, grep «не перезаписывай») и пунктом 9 чек-листа.
2. **Целевой проект не на JS/вебе** (Java, Python, Go) → ни одна команда или конвенция источника не должна просочиться в ядро. Закрепляется grep'ом Task 13 по core-файлам.
3. **Апстрим-скиллы с чужими лицензиями** → каждый внешний источник либо получает нотис, либо исключается. Закрепляется Task 5 и проверкой Task 13.
4. **Утечка личной инфраструктуры в публичный репо** → финальный grep по маркерам и `git status` перед push (Task 13), плюс ручное ревью диффа владельцем.
5. **Повторная установка поверх изменённого харнесса** → INSTALL требует дифф-обновление, а не слепое копирование. Закрепляется grep'ом «дифф» в Task 11.

---

### Task 1: Скелет репозитория

**Files:**
- Create: `.gitignore`
- Create: `LICENSE`
- Create: `README.md`

**Interfaces:**
- Produces: корень репо, куда следующие tasks кладут файлы; `.gitignore` защищает telegram-state и `.env`.

- [ ] **Step 1: Создать `.gitignore`**

```gitignore
# Секреты
.env
.env.*
!.env.example

# Состояние Telegram-моста (личное, не коммитится никогда)
telegram-bot/inbox.jsonl
telegram-bot/outbox.jsonl
telegram-bot/decisions.jsonl
telegram-bot/.bot.pid
telegram-bot/.bot.lock

# Зависимости
node_modules/

# ОС
.DS_Store
Thumbs.db

# SDD-workspace (черновики исполнения плана)
.superpowers/
```

- [ ] **Step 2: Создать `LICENSE` (MIT)**

```text
MIT License

Copyright (c) 2026 frostiks777

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

- [ ] **Step 3: Создать `README.md` (краткая версия; финальная — Task 12)**

```markdown
# Agent Harness

Переиспользуемое ядро AI-харнесса для OpenCode: инструкции агенту, 84 скилла,
процессные документы и опциональные адаптеры (Claude Code, Cursor, GitHub Actions,
Telegram).

## Как накатить

Открой этот репозиторий в целевом проекте и скажи агенту:

> Прочитай INSTALL.md и накати харнесс на этот проект.

Агент задаст обязательные вопросы (модель, адаптеры), скопирует ядро, адаптирует
плейсхолдеры и отчитается по чек-листу верификации.

## Лицензия

MIT. Сторонние скиллы — см. `THIRD-PARTY-NOTICES.md`.
```

- [ ] **Step 4: Проверить**

Run:
```powershell
git status --short
Get-Content LICENSE -TotalCount 3
```
Expected: три untracked-файла; первая строка LICENSE — `MIT License`.

- [ ] **Step 5: Commit**

```powershell
git add .gitignore LICENSE README.md
git commit -m "chore: scaffold agent-harness repository"
```

---

### Task 2: Шаблон `AGENTS.md` (стек-нейтральный)

**Files:**
- Create: `AGENTS.md`

**Interfaces:**
- Produces: шаблон, который INSTALL (Task 11) копирует в целевой проект; ссылается на `.agents/shared/checks.md` и `docs/agents/code-comments.md`.

- [ ] **Step 1: Создать `AGENTS.md`**

Полный текст:

````markdown
# <PROJECT NAME> — инструкция для AI-агентов

<!-- заполни: 2–3 строки — что это за проект, для кого, что считается работающим. -->

Этот файл — единственный источник правил для агентов в этом репозитории. Долговременная
память между сессиями — `MEMORY.md`; глоссарий предметной области — `CONTEXT.md`;
архитектурные решения — `docs/adr/`. Если правила расходятся с кодом — прав код, правило
надо обновить.

## Команды

<!-- заполни: реальные команды проекта (установка, dev, сборка, тесты). Чек-чейн перед
коммитом — `.agents/shared/checks.md`; здесь не дублируй его. -->

## Layout & data flow

<!-- заполни: где лежит входная точка, где бизнес-логика, где данные; что агент должен
прочитать до первой правки. -->

## Conventions

- **Комментарии объясняют «почему», а не «что».** Приёмы и формулировки запроса —
  `docs/agents/code-comments.md`. Сгенерированный комментарий — черновик: сверяй с
  `git log`/`blame` и тестами.
- **Язык UI:** <!-- заполни: например «все пользовательские строки — русские» или
  «интерфейс двуязычный: en + ru». -->
- <!-- заполни: именование, префиксы классов/пакетов, запреты, настройки линтера/форматтера (если есть). -->

## AI-assisted development cycle

Небольшая механическая работа: `ai/inbox.md` → задачи в `ai/tasks/<date>/<slug>/`.
Крупная фича или баг (публичный API, верхнеуровневое UX, формат данных, > ~2 файлов) —
сначала GitHub Issue, потом работа.

Внутри задачи процессные скиллы идут по порядку:
`interview → plan → (ponytail) → (tdd) → verify → commit-push`.

- **One issue, one change.** Один PR закрывает один issue (`Closes #NN` в теле PR).
  Если работа выросла за рамки issue — стоп и новый issue, а не раздувание PR.
- **Перед коммитом** — прогнать `.agents/shared/checks.md`. Красная проверка от твоего
  диффа означает, что задача не сделана.
- **Коммиты** — Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`).

## Agent skills

Скиллы лежат в `.agents/skills/<name>/SKILL.md` и подхватываются по `description` из
frontmatter. Имена директорий должны совпадать с `name`.

- Процессные: `interview`, `plan`, `ponytail`, `tdd`, `verify`, `commit-push`,
  `process-inbox`, `perform-task`, `continue`.
- Вспомогательные — по триггеру из описания.

Новый скилл: создать `.agents/skills/<name>/SKILL.md` с YAML-frontmatter
(`name` = имя директории, `description` = триггер).

## Worktrees (опционально)

<!-- заполни или удали секцию: если несколько агентов работают одновременно — каждому
свой worktree, ветка не занимается двумя копиями, чужой checkout не переключается. -->

## Hygiene of context window

- **Правило двух итераций:** если одна и та же проверка красная после двух попыток —
  сформулируй, почему подход неверен, предложи альтернативу и спроси владельца.
- Если диалог вырос, а прогресса нет — предложи `/compact` или новую сессию со ссылками
  на `AGENTS.md`, `MEMORY.md`, `CONTEXT.md`, `docs/adr/`.
- Долговременный контекст живёт только в файлах; всё, что не записано, теряется.

## Long-term memory

- `MEMORY.md` — состояние проекта между сессиями: стек, критические фиксы, результаты
  проверок. Обновляй при изменении стека или после крупной работы.
- `CONTEXT.md` — глоссарий предметной области; используй его словарь в issue, коммитах,
  коде и UI.
- `docs/adr/` — по одному ADR на трудно обратимое решение.

## Safety gates

- **Read-only в проде:** если окружение — production/staging, разрешены только чтение и
  диагностика. Никаких записей, миграций и удалений.
- **Human-in-the-loop:** force-push, `git reset --hard`, перезапись `MEMORY.md`, удаление
  ADR, правки CI-воркфлоу и действия на прод-сервере — только по явному запросу владельца;
  сначала покажи команду и последствия.
- **Destructive operations:** `rm -rf`, `rsync --delete`, очистка пользовательских данных —
  сначала dry-run/дифф и подтверждение.
- **Секреты:** токены и пароли не пишутся в код и коммиты; `.env` остаётся gitignored.

## Notifications

Уведомляй владельца только в двух случаях: (1) нужно решение — вопрос, развилка, блокер;
(2) успешный релиз — push прошёл, CI/деплой зелёные, задача закрыта.

Каналы: Windows-тост `powershell.exe -NoProfile -ExecutionPolicy Bypass -File
scripts/notify.ps1 "Заголовок" "Текст"` (сам файл `scripts/notify.ps1` обязан быть UTF-8 с
BOM; запускать через `powershell.exe` 5.1, не `pwsh` 7); Telegram — если установлен адаптер
`telegram/`.

## Agent behavior

- **Task routing:** мелкая механическая работа → `ai/inbox.md` → `ai/tasks/`; крупная —
  GitHub Issue первым.
- **Один issue — одно изменение.** План — это порядок, а не единица работы.
- Не коммить сгенерированный мусор (кэши, state, артефакты сборки и зависимостей).
- Все пользовательские строки — на языке проекта (см. Conventions).
- Перед отметкой «готово» — `verify` и отчёт с фактическим выводом команд.

## Known baseline

<!-- заполни: какие проверки сейчас зелёные/красные и почему; что считается
«известным исключением», а что — регрессией твоего диффа. -->
````

- [ ] **Step 2: Проверить стек-нейтральность**

Run:
```powershell
Select-String -Path AGENTS.md -Pattern 'bun|npm|next|react|eslint|playwright|tsc|shadcn|gallery|галере|photo|1pg|node_modules|css'
```
Expected: пусто.

- [ ] **Step 3: Commit**

```powershell
git add AGENTS.md
git commit -m "feat: add stack-neutral AGENTS.md template"
```

---

### Task 3: Ядро-конфиг `opencode.jsonc`

**Files:**
- Create: `opencode.jsonc`

**Interfaces:**
- Produces: валидный JSONC-конфиг; INSTALL (Task 11) меняет в нём `model` по ответу владельца.

- [ ] **Step 1: Создать `opencode.jsonc`**

```jsonc
{
  "$schema": "https://opencode.ai/config.json",

  // Модель основного агента. При установке замени на модель, которую выбрал владелец.
  // Бесплатные модели opencode отключаются и блокируются по IP — это регулярное событие,
  // а не разовая поломка: проверяй доступность командой `opencode run -m <model> "ping"`
  // и держи запасную модель наготове.
  "model": "opencode/mimo-v2.6-flash-free",

  // Модель субагентам здесь НЕ задаётся: без неё они наследуют модель родительской сессии.
  // Пин вида `provider/model` пережил бы смену провайдера и уронил бы прогон в CI.
  "agents": {
    // Исследовательский субагент: только чтение и веб, без edit/bash/task,
    // чтобы разведка не могла случайно что-то изменить.
    "explore": {
      "permission": {
        "read": "allow",
        "glob": "allow",
        "grep": "allow",
        "list": "allow",
        "webfetch": "allow",
        "websearch": "allow",
        "skill": "allow",
        "edit": "deny",
        "bash": "deny",
        "task": "deny",
        "external_directory": "deny"
      }
    }
  }

  // MCP-серверы добавляются здесь, когда проекту нужен конкретный сервер, например:
  //
  // "mcp": {
  //   "my-server": {
  //     "type": "local",
  //     "command": ["<команда запуска>", "mcp"],
  //     "enabled": true,
  //     "timeout": 60000
  //   }
  // }
}
```

- [ ] **Step 2: Проверить парсинг JSONC**

Run:
```powershell
node -e "const fs=require('fs');const s=fs.readFileSync('opencode.jsonc','utf8').replace(/^\s*\/\/.*$/gm,'');JSON.parse(s);console.log('jsonc ok')"
```
Expected: `jsonc ok`.

- [ ] **Step 3: Commit**

```powershell
git add opencode.jsonc
git commit -m "feat: add stack-neutral opencode config"
```

---

### Task 4: Шаблон `checks.md` и переиспользуемые `docs/agents/*`

**Files:**
- Create: `.agents/shared/checks.md`
- Create: `docs/agents/issue-tracker.md`
- Create: `docs/agents/triage-labels.md`
- Create: `docs/agents/code-comments.md`
- Source: `1pg_gall/.agents/shared/checks.md`, `1pg_gall/docs/agents/{issue-tracker,triage-labels,code-comments}.md`

**Interfaces:**
- Produces: `.agents/shared/checks.md`, на который ссылаются `AGENTS.md` и `verify`; `docs/agents/*`, на которые ссылается `AGENTS.md`.

- [ ] **Step 1: Создать `.agents/shared/checks.md` (шаблон без зашитых команд)**

````markdown
# Чек-чейн перед коммитом

Единый источник команд проверок. Используется скиллами `perform-task` и `verify`.
Заполни реальные команды проекта — ниже плейсхолдеры, а не рекомендации.

## 1. Установка зависимостей (только если нужно)

```bash
<команда установки зависимостей проекта>
```

Пропусти, если окружение уже готово. Если в проекте есть lock-файл, установка обязана
идти по нему, иначе проверки измеряют не ту версию.

## 2. Линт + типы

```bash
<команда линтера>
<команда проверки типов, если есть>
```

Baseline — ноль ошибок. Красный запуск после твоего диффа — твоя регрессия, а не
известное исключение.

## 3. Сборка

```bash
<команда сборки>
```

Если сборка требует сети или внешних сервисов, это фиксируется в Known baseline
`AGENTS.md`; недоступность среды — не «зелёная» сборка, а честный блокер.

## 4. Тесты

```bash
<команда тестов>
```

Если в проекте нет unit-фреймворка, используй имеющийся уровень (e2e/интеграционные) и
дополняй его, а не пиши одноразовую ручную проверку.

## 5. Визуальная проверка (только для UI-изменений)

Сделай скриншот и посмотри на него. «Сборка прошла» и «тесты зелёные» не доказывают, что
интерфейс выглядит правильно.

## Done means

Пункты 2–4 зелёные; для UI-изменений — ещё и 5. Либо в задаче записан конкретный
честный блокер, а не «готово» при красной проверке.
````

- [ ] **Step 2: Скопировать `docs/agents/issue-tracker.md` и вычистить проектное**

Взять `1pg_gall/docs/agents/issue-tracker.md`, затем заменить:

- строку 3 → `Issues and specs for this repo live as GitHub issues. Use the `gh` CLI for all operations.`
- абзац «Как это связано с `ai/tasks`» — оставить гибрид, но заменить `.data/` на
  «публичный API или формат данных проекта» и убрать ссылку на ADR-0002.
- абзац «Репозиторий single-maintainer…» → `Правило: если push в основную ветку деплоит, пушить можно только с зелёным локальным verify (`.agents/shared/checks.md`).`
- остальное (gh-команды, PR-flow `Closes #NN`, wayfinding-операции) — без изменений.

- [ ] **Step 3: Скопировать `docs/agents/triage-labels.md` и вычистить проектное**

Оставить таблицу пяти канонических ролей и абзац «When a skill mentions a role…».
Удалить раздел «Additional labels used in this repo (2026-09-28)» целиком.
Фразу `the actual label strings used in this repo's issue tracker` заменить на
`the actual label strings used in the project's issue tracker`.

- [ ] **Step 4: Скопировать `docs/agents/code-comments.md` и заменить проектные разделы**

- Разделы «Что комментировать», «Комментарий как лакмус», «Как просить агента»,
  «Зачем это агенту» — оставить как есть.
- Раздел «Примеры в этом репозитории» (строки 61–80 источника) заменить на:

```markdown
## Примеры в проекте

<!-- заполни: 3–5 эталонных «почему»-комментариев из кода проекта со ссылками
file:line и одной строкой — какую причину они фиксируют. Пока примеров нет, этот
раздел остаётся пустым: выдуманный пример хуже отсутствующего. -->
```

- Раздел «Правила репозитория» (строки 82–87) заменить на:

```markdown
## Правила проекта

- Язык комментариев: <!-- заполни: например «английский в коде, русский в UI-строках» -->.
- Чужой «почему»-комментарий сохраняется при правке соседнего кода: это чья-то
  реконструкция причины, а не мусор. Устаревший — поправь по факту, осознанно.
```

- [ ] **Step 5: Проверить**

Run:
```powershell
(Get-ChildItem .agents\shared -File).Count
Select-String -Path .agents\shared\checks.md,docs\agents\*.md -Pattern 'галере|photo|webdav|1pg|bun|next|playwright|\.data'
```
Expected: `1`; второй вывод пуст.

- [ ] **Step 6: Commit**

```powershell
git add .agents/shared docs/agents
git commit -m "feat: add checks template and process docs"
```

---

### Task 5: Лицензионный аудит скиллов + `THIRD-PARTY-NOTICES.md`

**Files:**
- Create: `THIRD-PARTY-NOTICES.md`
- Read: `1pg_gall/skills-lock.json`, `1pg_gall/.agents/skills/**/SKILL.md`

**Interfaces:**
- Produces: список скиллов, разрешённых к публикации; `THIRD-PARTY-NOTICES.md`, который Task 6 копирует в целевые проекты вместе со скиллами.

- [ ] **Step 1: Найти внешние атрибуции во всех скиллах**

Run (в источнике):
```powershell
Select-String -Path .agents\skills\*\SKILL.md -Pattern 'github\.com/[A-Za-z0-9_.-]+/[A-Za-z0-9_.-]+|author:|source:|homepage:|adapted from' | Select-Object Path,Line
```
Ожидаемые внешние источники (уже найдены): `gh-address-comments` (openai/skills),
`browser-skill` (Tencent/BrowserSkill), `open-code-review` (alibaba/open-code-review),
`pr` (humanlayer/skills), `coding-guidelines` (Karpathy Guidelines), наборы `tlc-*`
(tech-leads-club). Полный список — по факту вывода команды.

- [ ] **Step 2: Определить лицензию каждого внешнего источника**

Для каждого источника — проверить LICENSE апстрима (`webfetch` на
`https://raw.githubusercontent.com/<owner>/<repo>/main/LICENSE` или `gh api
repos/<owner>/<repo>/license`). Правило:

- MIT / Apache-2.0 / BSD / ISC → скилл остаётся, источник и лицензия попадают в нотис.
- Лицензия не найдена или запрещает перераспространение → скилл **исключается** из
  публикуемого набора и перечисляется в нотисе как «excluded pending license».
- Скиллы без внешней атрибуции и без записи в `skills-lock.json` считаются авторскими
  (импортированы из собственного проекта-донора `frostiks777/ai-for-developers-project-386`,
  commit `c84967c`).
- Если исключение случилось, зафиксируй новое число скиллов в результате Task 5 и используй
  его вместо `84` в Task 6 и Task 13; в отчёте владельцу перечисли исключённые скиллы и
  причину. Если исключений нет — целевое число 84.

- [ ] **Step 3: Создать `THIRD-PARTY-NOTICES.md`**

Шаблон (дополнить фактическими источниками из шага 1):

```markdown
# Third-Party Notices

Этот репозиторий содержит скиллы и документы из сторонних источников. Ниже — источники,
лицензии и условия.

## Скиллы из mattpocock/skills

- Источник: https://github.com/mattpocock/skills
- Лицензия: MIT, Copyright (c) 2026 Matt Pocock
- Состав: 38 скиллов; точные пути и хеши — в `skills-lock.json`.

## gh-address-comments

- Источник: https://github.com/openai/skills
- Лицензия: Apache License 2.0; полный текст — `.agents/skills/gh-address-comments/LICENSE.txt`.
- Файл лицензии сохраняется рядом со скиллом при копировании.

## Прочие скиллы

Скиллы без внешней атрибуции — авторские, импортированы из собственного
проекта-донора и адаптированы под этот харнесс. Они распространяются на условиях
MIT этого репозитория.
```

- [ ] **Step 4: Проверить**

Run:
```powershell
Get-Content THIRD-PARTY-NOTICES.md -TotalCount 5
Select-String -Path THIRD-PARTY-NOTICES.md -Pattern 'mattpocock|Apache|MIT'
```
Expected: заголовок на месте; все три маркера найдены.

- [ ] **Step 5: Commit**

```powershell
git add THIRD-PARTY-NOTICES.md
git commit -m "docs: add third-party license notices"
```

---

### Task 6: Копирование и адаптация 84 скиллов

**Files:**
- Create: `.agents/skills/**` (84 директории)
- Create: `skills-lock.json`
- Modify: `.agents/skills/{verify,interview,commit-push,tdd,process-inbox,plan,ponytail}/SKILL.md` (по фактическим попаданиям grep)

**Interfaces:**
- Consumes: список разрешённых из Task 5.
- Produces: каталог, который OpenCode читает нативно (проверено в источнике: `skill`-инструмент видит `.agents/skills/`).

- [ ] **Step 1: Скопировать скиллы и lock**

```powershell
New-Item -ItemType Directory -Force .agents | Out-Null
Copy-Item C:\My_projects_GIT\1pg_gall\.agents\skills .agents\ -Recurse
Remove-Item .agents\skills\apply-design -Recurse
Remove-Item .agents\skills\apply-design-v2 -Recurse
# Дополнительно удалить скиллы, исключённые лицензионным аудитом Task 5 (если такие есть).
Copy-Item C:\My_projects_GIT\1pg_gall\skills-lock.json .
```

- [ ] **Step 2: Найти проектные упоминания**

```powershell
Select-String -Path .agents\skills\*\SKILL.md -Pattern '1pg|галере|photo|gallery|duckdns|ap-p-g|webdav|bun run|bunx|next dev|\.data|playwright|eslint|tsc --noEmit' -AllMatches | Group-Object Path | ForEach-Object { $_.Name }
```
Ожидаемые файлы: `verify`, `interview`, `commit-push`, `tdd`, `process-inbox`, `plan`,
`ponytail` (по факту вывода).

- [ ] **Step 3: Адаптировать каждый найденный скилл**

Правила замен (смысл сохраняется, проектная привязка уходит):

- `bun run lint`, `bunx tsc --noEmit`, `bun run build`, `bun run test:e2e`,
  `bunx playwright test` → `команды из .agents/shared/checks.md` (соответствующий пункт).
- «галерея», `src/app/api/*`, `.data/`, WebDAV, фото/видео → «проект», «публичный API»,
  «формат данных».
- «push в `main` деплоит прод» → «если push в основную ветку деплоит, без зелёного verify
  не пушить».
- «в проекте нет unit-тестов, только e2e» → «используй имеющийся уровень тестов проекта;
  если unit-фреймворка нет — e2e/интеграционные, а не одноразовая ручная проверка».

- [ ] **Step 4: Проверить количество и чистоту**

```powershell
(Get-ChildItem .agents\skills -Directory).Count
Select-String -Path .agents\skills\*\SKILL.md -Pattern '1pg|галере|photo|gallery|duckdns|ap-p-g|webdav|bun run|bunx|next dev|\.data|playwright|eslint|tsc --noEmit'
```
Expected: `84`; второй вывод пуст. Если пусто не стало — править до пустого.

- [ ] **Step 5: Проверить frontmatter каждого скилла**

```powershell
Get-ChildItem .agents\skills -Directory | ForEach-Object {
  $head = Get-Content "$($_.FullName)\SKILL.md" -TotalCount 3 -ErrorAction SilentlyContinue
  if (-not ($head -match 'name:')) { "NO name: $($_.Name)" }
}
```
Expected: пусто.

- [ ] **Step 6: Commit**

```powershell
git add .agents/skills skills-lock.json
git commit -m "feat: import 84 stack-neutral skills"
```

---

### Task 7: `scripts/notify.ps1` без проектного AppId

**Files:**
- Create: `scripts/notify.ps1`
- Source: `1pg_gall/scripts/notify.ps1:16` (`CreateToastNotifier('Фото-галерея')`)

- [ ] **Step 1: Скопировать и заменить AppId**

```powershell
New-Item -ItemType Directory -Force scripts | Out-Null
(Get-Content C:\My_projects_GIT\1pg_gall\scripts\notify.ps1 -Raw).Replace("'Фото-галерея'", "'Agent Harness'") |
  Set-Content scripts\notify.ps1 -Encoding utf8
```

- [ ] **Step 2: Восстановить UTF-8 с BOM**

`pwsh` 7 пишет `-Encoding utf8` **без** BOM, поэтому шаг 2 обязателен всегда:

```powershell
$p = (Resolve-Path scripts\notify.ps1).Path
$t = [IO.File]::ReadAllText($p)
[IO.File]::WriteAllText($p, $t, (New-Object System.Text.UTF8Encoding($true)))
```

- [ ] **Step 3: Проверить**

```powershell
Format-Hex scripts\notify.ps1 -Count 3
Select-String -Path scripts\notify.ps1 -Pattern 'Фото|галере'
```
Expected: первые три байта `EF BB BF`; второй вывод пуст.

- [ ] **Step 4: Commit**

```powershell
git add scripts/notify.ps1
git commit -m "feat: add neutral windows toast helper"
```

---

### Task 8: Адаптеры `claude/` и `cursor/`

**Files:**
- Create: `adapters/claude/CLAUDE.md`
- Create: `adapters/claude/.claude/settings.json` (копия `1pg_gall/.claude/settings.json`)
- Create: `adapters/cursor/commands/security-audit.md` (копия `1pg_gall/.cursor/commands/security-audit.md`)
- Create: `adapters/cursor/README.md`

**Interfaces:**
- Consumes: ядро из Task 2 (`@AGENTS.md` указывает на него).
- Produces: адаптеры, которые INSTALL копирует по согласию владельца.

- [ ] **Step 1: Создать `adapters/claude/CLAUDE.md`**

```markdown
# CLAUDE.md

Правила для Claude Code живут в корневом `AGENTS.md`, чтобы у OpenCode и Claude Code
был один источник истины:

@AGENTS.md
```

- [ ] **Step 2: Скопировать `settings.json`**

```powershell
New-Item -ItemType Directory -Force adapters\claude\.claude | Out-Null
Copy-Item C:\My_projects_GIT\1pg_gall\.claude\settings.json adapters\claude\.claude\settings.json
```

Не копировать `.claude/rules/security.md`: это generic boilerplate из шаблона, не политика
проекта (источник сам так про него пишет).

- [ ] **Step 3: Скопировать cursor-команду и создать README**

```powershell
New-Item -ItemType Directory -Force adapters\cursor\commands | Out-Null
Copy-Item C:\My_projects_GIT\1pg_gall\.cursor\commands\security-audit.md adapters\cursor\commands\
```

`adapters/cursor/README.md`:

```markdown
# Cursor adapter

Скопируйте `.cursor/` в корень проекта.

- `commands/security-audit.md` — команда аудита безопасности.
- `rules/security.mdc` и `hooks.json` **не включены**: в проекте-источнике это generic
  boilerplate, а `hooks.json` ссылался на несуществующий `.cursor/hooks/gitleaks-hook`
  (битая конфигурация). Если нужен gitleaks-хук — настройте его отдельно и осознанно.
```

- [ ] **Step 4: Проверить**

```powershell
node -e "JSON.parse(require('fs').readFileSync('adapters/claude/.claude/settings.json','utf8'));console.log('settings ok')"
Test-Path adapters\cursor\hooks.json
Select-String -Path adapters\claude\CLAUDE.md -Pattern '@AGENTS.md'
```
Expected: `settings ok`; `False`; строка `@AGENTS.md` найдена.

- [ ] **Step 5: Commit**

```powershell
git add adapters/claude adapters/cursor
git commit -m "feat: add claude and cursor adapters"
```

---

### Task 9: Адаптер `github-actions/`

**Files:**
- Create: `adapters/github-actions/.github/workflows/opencode.yml` (копия)
- Create: `adapters/github-actions/.github/workflows/opencode-triage.yml` (копия + правки)
- Create: `adapters/github-actions/.github/workflows/opencode-review.yml` (копия + правки)
- Create: `adapters/github-actions/.github/workflows/opencode-audit.yml` (копия + правки)
- Create: `adapters/github-actions/scripts/lighthouse-summary.mjs` (копия)
- Create: `adapters/github-actions/scripts/audit.sh.example`
- Create: `adapters/github-actions/README.md`

**Interfaces:**
- Consumes: ничего из ядра (воркфлоу автономны).
- Produces: адаптер, устанавливаемый только по согласию; требует секрет `OPENROUTER_API_KEY` и variable `OPENCODE_MODEL`.

- [ ] **Step 1: Скопировать воркфлоу и скрипт**

```powershell
New-Item -ItemType Directory -Force adapters\github-actions\.github\workflows, adapters\github-actions\scripts | Out-Null
Copy-Item C:\My_projects_GIT\1pg_gall\.github\workflows\opencode.yml,`
          C:\My_projects_GIT\1pg_gall\.github\workflows\opencode-triage.yml,`
          C:\My_projects_GIT\1pg_gall\.github\workflows\opencode-review.yml,`
          C:\My_projects_GIT\1pg_gall\.github\workflows\opencode-audit.yml `
          adapters\github-actions\.github\workflows\
Copy-Item C:\My_projects_GIT\1pg_gall\scripts\lighthouse-summary.mjs adapters\github-actions\scripts\
```

- [ ] **Step 2: Адаптировать `opencode-triage.yml`**

Найти блок про структуру проекта (`src/app/page.tsx`, `src/components/gallery/`,
`src/app/api/*/route.ts`) и заменить на:

```text
Опиши структуру проекта по его собственным правилам: где входная точка, где бизнес-логика,
где данные. Не делай предположений о стеке — прочитай `AGENTS.md` и манифест зависимостей.
```

Найти блок про `bun run lint`/`bunx tsc --noEmit`/`npx` и заменить на:

```text
Команды проверок — в `.agents/shared/checks.md`; используй реальные команды проекта,
не подставляй менеджер пакетов по памяти.
```

- [ ] **Step 3: Адаптировать `opencode-review.yml`**

Удалить правила про `devEngines`/`bun`/`npx` и список путей галереи
(`src/app/api`, `.data/`, «публичный API»). Вместо них:

```text
- Сверяйся с `AGENTS.md` и `.agents/shared/checks.md` целевого проекта: правила проверок
  и язык UI — оттуда, а не из общих представлений о стеке.
```

- [ ] **Step 4: Адаптировать `opencode-audit.yml`**

- Верхний комментарий про `eslint`/`tsc` заменить на: `code-audit — запускает
  .github/audit.sh (команды проекта задаёт владелец)`.
- В job `code-audit` убрать шаги установки bun и запуска `bun run lint`/`bunx tsc --noEmit`;
  оставить один шаг:

```yaml
      - name: Run project audit script
        run: |
          if [ -f .github/audit.sh ]; then
            bash .github/audit.sh
          else
            echo "::warning::.github/audit.sh не найден — code-audit пропущен"
          fi
```

- Job `lighthouse` не трогать: он уже пропускается, если variable `APP_URL` не задана.
- Остальное (artifact, вызов агента, `if: always()`) — без изменений.

- [ ] **Step 5: Создать `audit.sh.example`**

```bash
#!/usr/bin/env bash
# Пример: замени команды на реальные команды проекта.
# Скопируй в целевой проект как .github/audit.sh и сделай исполняемым.
set -u

# Пример для Node/bun-проекта:
bun install --frozen-lockfile
bun run lint
bunx tsc --noEmit

# Пример для Java/Maven:
# mvn -q -B verify

# Пример для Python:
# python -m pip install -r requirements.txt && python -m pytest -q
```

- [ ] **Step 6: Создать `adapters/github-actions/README.md`**

```markdown
# GitHub Actions adapter

Скопируйте `.github/` и `scripts/` в корень проекта.

## Что нужно настроить

- Секрет `OPENROUTER_API_KEY` (Settings → Secrets and variables → Actions → Secrets).
  Обычный ключ `sk-or-v1-…`, не management/provisioning: management-ключ проходит
  `/api/v1/key`, но `/chat/completions` отвечает `401 User not found`.
- Variable `OPENCODE_MODEL` (тот же раздел, Variables) — записывается **без префикса
  провайдера**; воркфлоу подставляют `openrouter/` сами.
- Variable `APP_URL` — только для job `lighthouse`; без неё job пропускается.
- `.github/audit.sh` — команды линта/типов проекта (пример: `scripts/audit.sh.example`).

## Поведение

- `opencode.yml` — ручной вызов командой `/oc` в комментарии.
- `opencode-triage.yml` — разбор нового issue.
- `opencode-review.yml` — замечания к PR; PR от ботов пропускаются.
- `opencode-audit.yml` — ежедневный аудит: отчёт артефактом + issue.
- Пуш в основную ветку может деплоить — ветку с агентными изменениями открывайте PR.
```

- [ ] **Step 7: Проверить YAML**

```powershell
bun -e "const fs=require('fs');for(const f of fs.readdirSync('adapters/github-actions/.github/workflows')){Bun.YAML.parse(fs.readFileSync('adapters/github-actions/.github/workflows/'+f,'utf8'));console.log('ok',f)}"
Select-String -Path adapters\github-actions\.github\workflows\*.yml -Pattern '1pg|галере|photo|webdav|src/app|devEngines'
```
Expected: четыре `ok`; второй вывод пуст.

- [ ] **Step 8: Commit**

```powershell
git add adapters/github-actions
git commit -m "feat: add github actions adapter"
```

---

### Task 10: Адаптер `telegram/`

**Files:**
- Create: `adapters/telegram/telegram-bot/{bot,notify,reply,unread,setup-menu}.mjs`
- Create: `adapters/telegram/telegram-bot/{package.json,README.md,start-bot.bat,.env.example}`
- Create: `adapters/telegram/scripts/telegram-autostart-smoke.mjs`
- Create: `adapters/telegram/.opencode/plugins/telegram-autostart.js`
- Create: `adapters/telegram/.opencode/package.json`

**Interfaces:**
- Consumes: `.opencode/package.json` пинит `@opencode-ai/plugin` версией из источника (1.18.33).
- Produces: адаптер, ставящийся только по согласию; состояние никогда не коммитится.

- [ ] **Step 1: Скопировать скрипты (без state)**

```powershell
New-Item -ItemType Directory -Force adapters\telegram\telegram-bot, adapters\telegram\scripts, adapters\telegram\.opencode\plugins | Out-Null
Copy-Item C:\My_projects_GIT\1pg_gall\telegram-bot\bot.mjs,`
          C:\My_projects_GIT\1pg_gall\telegram-bot\notify.mjs,`
          C:\My_projects_GIT\1pg_gall\telegram-bot\reply.mjs,`
          C:\My_projects_GIT\1pg_gall\telegram-bot\unread.mjs,`
          C:\My_projects_GIT\1pg_gall\telegram-bot\setup-menu.mjs,`
          C:\My_projects_GIT\1pg_gall\telegram-bot\package.json,`
          C:\My_projects_GIT\1pg_gall\telegram-bot\README.md,`
          C:\My_projects_GIT\1pg_gall\telegram-bot\start-bot.bat,`
          C:\My_projects_GIT\1pg_gall\telegram-bot\.env.example `
          adapters\telegram\telegram-bot\
Copy-Item C:\My_projects_GIT\1pg_gall\scripts\telegram-autostart-smoke.mjs adapters\telegram\scripts\
Copy-Item C:\My_projects_GIT\1pg_gall\.opencode\plugins\telegram-autostart.js adapters\telegram\.opencode\plugins\
```

**Не копировать**: `.env`, `inbox.jsonl`, `outbox.jsonl`, `decisions.jsonl`, `.bot.pid`,
`.bot.lock`, `.opencode/.gitignore` (он игнорирует `package.json`, а нам нужен трекнутый).

- [ ] **Step 2: Создать трекнутый `.opencode/package.json`**

```json
{
  "dependencies": {
    "@opencode-ai/plugin": "1.18.33"
  }
}
```

- [ ] **Step 3: Проверить отсутствие личных данных и state**

```powershell
Get-ChildItem adapters\telegram -Recurse -Force -File | Select-Object -ExpandProperty FullName
Get-ChildItem adapters\telegram -Recurse -File -Include *.mjs,*.js,*.json,*.md |
  Select-String -Pattern '1pg|галере|duckdns|95\.81|pgagent'
```
Expected: в списке нет `.env`, `*.jsonl`, `.bot.pid/.lock`; второй вывод пуст.

- [ ] **Step 4: Commit**

```powershell
git add adapters/telegram
git commit -m "feat: add telegram bridge adapter"
```

---

### Task 11: `INSTALL.md` — задание агенту

**Files:**
- Create: `INSTALL.md`

**Interfaces:**
- Consumes: структуру из Task 1–10 (имена файлов и каталогов должны совпадать).
- Produces: главный документ; на него ссылается README.

- [ ] **Step 1: Создать `INSTALL.md`**

Полный текст (обязательные формулировки — «не перезаписывай», «дифф» — проверит Task 13):

````markdown
# Установка харнесса

Ты — AI-агент в целевом проекте. Перед тобой репозиторий `agent-harness` — переиспользуемое
ядро харнесса. Твоя задача: накатить его на текущий проект, **не потеряв того, что уже
настроено**, и доказать результат проверками.

Работай по шагам, ничего не пропуская.

## Предусловия

- OpenCode v2 (или устанавливаемый адаптер: Claude Code / Cursor).
- `git` в целевом проекте.
- Репозиторий харнесса доступен по URL (склонируй во временный каталог) или локальным
  путём (используй напрямую).

## Шаг 0 — разведка целевого проекта

1. Проверь, что уже есть: `AGENTS.md`, `CLAUDE.md`, `.agents/`, `.opencode/`, `.claude/`,
   `.cursor/`, `opencode.jsonc`, `ai/`, `.github/workflows/opencode*.yml`.
2. **Не перезаписывай существующие инструкции владельца.** Правила владельца приоритетны.
   Если секция конфликтует с шаблоном — покажи конфликт в отчёте и спроси, а не заменяй
   молча.
3. Определи стек проекта по манифесту зависимостей и существующим командам; ничего не
   предполагай по памяти.

## Шаг 1 — вопросы до установки

Задай владельцу вопросы (одним пакетом, через `question(...)`, если инструмент доступен):

1. **Модель** для `opencode.jsonc`: оставить `opencode/mimo-v2.6-flash-free` или указать
   другую? (Шаблон валиден с дефолтом, но подтверждение обязательно.)
2. **Telegram-мост**: ставить адаптер `adapters/telegram/`? Для него нужны токен бота и
   chat id, которые хранятся только в gitignored `.env`.
3. **GitHub Actions**: ставить адаптер `adapters/github-actions/`? Потребуются секрет
   `OPENROUTER_API_KEY` и variable `OPENCODE_MODEL`; без них воркфлоу не заработают.
4. **Claude Code / Cursor**: нужны ли адаптеры `adapters/claude/` и `adapters/cursor/`?

## Шаг 2 — копирование ядра

Скопируй в целевой проект, сохраняя dotfiles:

- `AGENTS.md`
- `.agents/` (скиллы и `shared/checks.md`)
- `opencode.jsonc`
- `docs/agents/` (`issue-tracker.md`, `triage-labels.md`, `code-comments.md`)
- `scripts/notify.ps1`
- `skills-lock.json`
- `THIRD-PARTY-NOTICES.md` — **обязательно**: MIT и Apache-2.0 требуют сохранять атрибуцию
  в копиях.

**Не копируй** `LICENSE` харнесса: это лицензия репозитория-источника, а не целевого
проекта; копирование ложно перелицензировало бы проект.

## Шаг 3 — адаптация

1. Заполни плейсхолдеры `<!-- заполни: ... -->` в `AGENTS.md`: описание проекта, команды,
   layout, конвенции, язык UI, worktrees (или удали секцию), Known baseline.
2. Заполни реальными командами `.agents/shared/checks.md`: установка, линт, типы, сборка,
   тесты. Не оставляй плейсхолдеры.
3. В `opencode.jsonc` поставь выбранную модель; MCP добавляй только если он реально нужен.
4. Если worktrees не используются — удали секцию, а не оставляй пустую.
5. Если Telegram-адаптер не выбран — не копируй `telegram/`, а из `AGENTS.md` убери
   упоминание Telegram-канала (тост оставь).
6. Адаптеры копируй поверх целевого проекта по согласию: `claude/` → `.claude/` +
   `CLAUDE.md`; `cursor/` → `.cursor/`; `github-actions/` → `.github/` + `scripts/`;
   `telegram/` → `telegram-bot/` + `.opencode/` + `scripts/`.
7. Дополни `.gitignore` целевого проекта, если в нём нет `.env*` и состояния Telegram.

## Шаг 4 — гигиена секретов

- Не копируй `.env` и state-файлы; токены не должны попасть в git.
- Проверь `git diff`/`git status`: в новых файлах не должно быть токенов, паролей,
  приватных ключей.
- Если владелец дал токены — они живут только в `.env`, который уже в `.gitignore`.

## Шаг 5 — верификация

Выполни каждый пункт и приложи доказательство (команда + вывод). Пункт без доказательства
не считается выполненным.

1. **Файлы на месте**: ядро и выбранные адаптеры; `.agents/skills` = 84; лишнего нет.
2. **Плейсхолдеры закрыты**: `grep` по `<!-- заполни`/`{{` в `AGENTS.md` и
   `.agents/shared/checks.md` не находит их (кроме сознательно пустых секций).
3. **`opencode.jsonc` валиден**, модель — из ответа владельца.
4. **Скиллы видны OpenCode**: `skill`-инструмент находит `interview`, `plan`, `ponytail`,
   `tdd`, `verify`, `commit-push` (доказательство — фактический вызов/листинг).
5. **`checks.md` реален**: каждая команда запущена, коды возврата в отчёте; незаполненное
   помечается «требует владельца», а не PASS.
6. **Секретов нет**: `grep` по `api[_-]?key|token|password|BEGIN.*PRIVATE KEY`; `.env`
   отсутствует и не закоммичен.
7. **Личных данных нет**: `grep` по известным маркерам (IP, домены, ники, SSH) = 0.
8. **Адаптеры только по согласию**: для `github-actions` — YAML валиден, README перечисляет
   секреты; если адаптер не выбран — его файлов нет.
9. **Интеграция**: прежние инструкции владельца сохранены; покажи дифф, а не «всё ок».
10. **Отчёт с доказательствами**: PASS/FAIL по каждому пункту + вывод команд. Любой FAIL
    блокирует «готово».

## Шаг 6 — отчёт

В чат владельцу: что скопировано, что адаптировано, что пропущено и почему; таблица
PASS/FAIL; открытые пункты «требует владельца».

## Повторная установка

Обновление — **диффом**, а не слепым копированием: сравни файлы харнесса с целевыми,
покажи расхождения и спроси, что применять. Локальные правки владельца не затирай вслепую.

Если репозиторий харнесса обновился — сначала прочитай `docs/updating.md` этого
репозитория: там перечислено, что из проекта-донора не переносится.
````

- [ ] **Step 2: Проверить**

```powershell
Select-String -Path INSTALL.md -Pattern 'не перезаписывай|дифф|THIRD-PARTY-NOTICES|84|question'
```
Expected: найдены все маркеры (`не перезаписывай`, `дифф`, `THIRD-PARTY-NOTICES`, `84`).

- [ ] **Step 3: Commit**

```powershell
git add INSTALL.md
git commit -m "docs: add agent-facing install guide"
```

---

### Task 12: Финальные `README.md` и `docs/updating.md`

**Files:**
- Modify: `README.md`
- Create: `docs/updating.md`

- [ ] **Step 1: Дописать `README.md` до финального вида**

Язык — **простой и деловой, для человека, который не знает внутренностей**: что это даёт,
зачем нужно, как начать. Без жаргона «стек-нейтральное ядро», «SDD», «плейсхолдеры».

Обязательные разделы:

1. **Что это** — 2–3 предложения: готовый набор правил и навыков для AI-агентов; подключил
   к проекту — агент знает, как работать, что проверять и как отчитываться.
2. **Кому подходит** — командам и одиночным разработчикам, которые используют AI-агентов
   в проектах любого стека.
3. **Как начать** — цитата для агента: «Прочитай INSTALL.md и накати харнесс на этот проект».
4. **Что внутри** — ядро (правила, 84 навыка, проверки, память проекта) и опциональные
   адаптеры (Claude Code, Cursor, GitHub Actions, Telegram) — по одному предложению на каждый.
5. **Требования** — OpenCode v2; для адаптеров — свои (кратко).
6. **Обновление** — ссылка на `docs/updating.md`.
7. **Лицензия** — MIT + ссылка на `THIRD-PARTY-NOTICES.md`.
8. **Чего здесь нет** — память и деплой конкретного проекта; это переносится отдельно.


- [ ] **Step 2: Создать `docs/updating.md`**

```markdown
# Обновление харнесса из проекта-источника

Источник истины — проект-донор харнесса (локально `1pg_gall`). Обновление ручное.

## Порядок

1. Сравни `1pg_gall` и этот репозиторий по списку: `AGENTS.md`, `.agents/skills`,
   `.agents/shared/checks.md`, `opencode.jsonc`, `docs/agents/*`, `scripts/notify.ps1`,
   `adapters/*`.
2. Перенеси изменения, **вычищая проектную специфику**: стек, команды, домены, IP,
   личные данные, `MEMORY.md`/`CONTEXT.md`/ADR, деплой и nginx.
3. Прогони проверки: количество скиллов (84), отсутствие плейсхолдеров в README/INSTALL,
   отсутствие личных маркеров и секретов.
4. Осознанные расхождения фиксируй в commit message: например,
   `skills: exclude apply-design (course project)`.

## Что не переносится никогда

- `MEMORY.md`, `CONTEXT.md`, `docs/adr/` проекта-донора.
- Конфиги деплоя, nginx, CI сборки/релиза проекта-донора.
- `.data/`, `.env`, состояние Telegram-моста.
- Личные данные владельца: серверы, SSH, chat id, токены.
```

- [ ] **Step 3: Проверить**

```powershell
Select-String -Path README.md,docs\updating.md -Pattern '<!-- заполни'
Test-Path docs\updating.md
```
Expected: пусто; `True`.

- [ ] **Step 4: Commit**

```powershell
git add README.md docs/updating.md
git commit -m "docs: finalize readme and update guide"
```

---

### Task 13: Финальная верификация и публикация

**Files:**
- Modify: `docs/design/2026-11-12-agent-harness-design.md` (убрать абсолютный локальный путь)

- [ ] **Step 1: Убрать абсолютный путь из дизайн-документа**

Заменить `C:\My_projects_GIT\1pg_gall` на `1pg_gall` (относительное имя источника).

- [ ] **Step 2: Прогнать проверки**

```powershell
# 1) Количество скиллов
(Get-ChildItem .agents\skills -Directory).Count

# 2) Проектные маркеры в ядре
Select-String -Path AGENTS.md,opencode.jsonc,.agents\shared\checks.md,docs\agents\*.md -Pattern 'bun|npm|next|react|eslint|playwright|tsc|shadcn|галере|photo|gallery|webdav'

# 3) Личные данные и секреты во всём репозитории
git grep -nE "95\.81|duckdns|pgagent|1pg_gall|1pgWebGallery|Фото-галерея|BEGIN.*PRIVATE KEY|api[_-]?key|password"

# 4) Плейсхолдеры в пользовательских документах
Select-String -Path README.md,INSTALL.md,docs\updating.md,THIRD-PARTY-NOTICES.md -Pattern '<!-- заполни'

# 5) JSONC и YAML
node -e "const fs=require('fs');JSON.parse(fs.readFileSync('opencode.jsonc','utf8').replace(/^\s*\/\/.*$/gm,''));console.log('jsonc ok')"
bun -e "const fs=require('fs');for(const f of fs.readdirSync('adapters/github-actions/.github/workflows')){Bun.YAML.parse(fs.readFileSync('adapters/github-actions/.github/workflows/'+f,'utf8'));console.log('yaml ok',f)}"

# 6) Чистое дерево
git status --short
```
Expected: `84`; (2) пусто; (3) — только безобидные совпадения вида `api_key` в тексте скиллов,
каждое просмотреть глазами и подтвердить, что это не секрет; (4) пусто; `jsonc ok` + четыре
`yaml ok`; `git status` чистый.

- [ ] **Step 3: Ревью диффа владельцем**

Показать владельцу `git log --oneline` и `git status`; **дождаться явного «публикуй»** —
репозиторий публичный, ошибка не откатывается.

- [ ] **Step 4: Влить в `main` и запушить в существующий репозиторий**

Репозиторий `https://github.com/frostiks777/harness` уже существует и публичен
(проверено: `gh repo view frostiks777/harness` → `PUBLIC`); `origin` на него настроен.

```powershell
git checkout main
git merge --ff-only feat/harness-core
git push -u origin main
```

- [ ] **Step 5: Заполнить поля проекта на GitHub**

Описание — простым деловым языком, без жаргона:

```powershell
gh repo edit frostiks777/harness `
  --description "Готовый набор правил и навыков для AI-агентов: подключите репозиторий к проекту — и агент сам настроит рабочие правила, проверки и помощников." `
  --add-topic ai-agents `
  --add-topic ai-coding `
  --add-topic opencode `
  --add-topic developer-tools `
  --add-topic claude-code `
  --add-topic cursor
```

- [ ] **Step 6: Подтвердить публикацию**

```powershell
gh repo view frostiks777/harness --json name,visibility,url,description,repositoryTopics
git ls-remote origin main
```
Expected: `"visibility":"PUBLIC"`; описание заполнено; темы содержат `ai-agents`, `opencode`;
`main` присутствует в remote.

---

## Acceptance criteria (из спеки §10)

- [ ] Публичный `frostiks777/harness` создан, `main` запушен.
- [ ] Описание репозитория и темы заполнены на GitHub, понятным бизнес-языком; `README.md`
      написан так же.
- [ ] `LICENSE` (MIT), `THIRD-PARTY-NOTICES.md`, `README.md`, `INSTALL.md` на месте.
- [ ] Ядро и адаптеры соответствуют спеке §3; скиллов 84; `skills-lock.json` на месте.
- [ ] `AGENTS.md` и `checks.md` — валидные шаблоны без стек-специфики и личных данных.
- [ ] `INSTALL.md` содержит обязательные вопросы, политику «не перезаписывай», дифф-обновление
      и 10-пунктовый чек-лист.
- [ ] `grep`-проверки Task 13.2 (2)–(5) — чистые.
