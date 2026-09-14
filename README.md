# QAI-05-AEgorova — мультиагентная система обработки требований QA

Мультиагентная система (Agent Workflow) на базе [opencode](https://opencode.ai), которая автоматизирует полный цикл обработки требований: от ревью требований в Confluence до создания задач в Jira и риск-ориентированной тестовой модели в Qase.

## Назначение

Система принимает ссылку/ID страницы Confluence с требованиями и выполняет три этапа:

1. **Requirement Review** — систематический анализ требований, выявление неоднозначностей, конфликтов, пробелов и рисков.
2. **Jira Tasks** — декомпозиция подтверждённых требований на задачи и их фактическое создание в Jira.
3. **Test Model** — проектирование риск-ориентированной тестовой модели и создание тестовых сущностей в Qase.

Результаты передаются между этапами через проверенные артефакты, а не через историю диалога.

## Архитектура

### Агенты

Агенты описаны в `.opencode/agents/` и используют общие положения из `AGENTS.md`.

| Агент | Файл | Роль |
|-------|------|------|
| **orchestrator** | `qa-orchestrator.md` | Управляет Workflow, вызывает саб-агентов, владеет Quality Gates, финальным Output и статусом |
| **requirement-review-agent** | `requirement-review-agent.md` | Ревью требований из Confluence (Skill `requirement-review`), формирует Requirement Review Artifact |
| **jira-task-agent** | `jira-task-agent.md` | Декомпозиция требований и создание задач в Jira (Skill `task-design`) |
| **test-design-agent** | `test-design-agent.md` | Проектирование и создание тестовой модели в Qase (Skill `test-design`) |

Orchestrator вызывает саб-агентов как Tools (LLM-as-tool). Каждый саб-агент владеет только своими инструментами и читает только необходимые артефакты.

### Skills

Профессиональные методики вынесены в отдельные файлы и не дублируются в инструкциях агентов.

| Skill | Файл | Назначение |
|-------|------|------------|
| requirement-review | `.opencode/skills/requirement-review/SKILL.md` | Методика анализа требований и формирования Requirement Review |
| task-design | `.opencode/skills/task-design/SKILL.md` | Методика декомпозиции требований на Jira-задачи |
| test-design | `.opencode/skills/test-design/SKILL.md` | Методика риск-ориентированного тест-дизайна |

### Внешние системы (MCP)

- **Confluence** — Source of Truth по требованиям и бизнес-правилам (read / write comment).
- **Jira** — создание задач; источник информации о существующих задачах (анти-дубликат).
- **Qase** — создание сьютов и тест-кейсов; источник существующей тестовой модели (анти-дубликат).

## Структура репозитория

```
.
├── AGENTS.md                  # Общие положения для всех агентов (контракт)
├── .opencode/
│   ├── agents/                # Инструкции агентов (orchestrator, 3 саб-агента)
│   └── skills/                # Методики (requirement-review, task-design, test-design)
└── artifacts/                 # Артефакты (результаты этапов Workflow)
```

## Workflow

1. **Orchestrator** определяет цель запуска и входные данные (страница Confluence, проект Jira, проект Qase).
2. Вызывается **requirement-review-agent** → создаётся артефакт `/artifacts/requirement-review-{PageID}-{YYYYMMDDhhmm}.md`, на страницу Confluence добавляется комментарий с Missing Information / Conflicts / Open Questions.
3. Проверяется **Requirement Review Gate**.
4. Параллельно вызываются **jira-task-agent** и **test-design-agent** на основании проверенного Requirement Review.
5. Проверяются **Task Design Gate / Jira Gate** и **Test Design Gate / Qase Gate**.
6. Проверяется **Final Gate**, формируется итоговый Output и статус.

Этапы 4–5 (Jira и Qase) независимы и могут выполняться параллельно.

## Артефакты (контракт)

| Артефакт | Producer | Consumer | Path |
|---|---|---|---|
| Requirement Review | requirement-review-agent | jira-task-agent, test-design-agent | `/artifacts/requirement-review-{PageID}-{YYYYMMDDhhmm}.md` |
| Jira Tasks | jira-task-agent | orchestrator | `/artifacts/jira-task-{PageID}-{YYYYMMDDhhmm}.md` |
| Test Model | test-design-agent | orchestrator | `/artifacts/test-model-{PageID}-{YYYYMMDDhhmm}.md` |

Для каждого артефакта определены: структура, Completion Criteria (Validation), Error Handling, потребители. Артефакт не передаётся на следующий этап, пока не прошёл Quality Gate. История диалога не заменяет проверенный артефакт.

## Quality Gates

- **Requirement Review Gate** — артефакт существует, критерии Skill выполнены, неоднозначности/конфликты зафиксированы, пропуски не заполнены предположениями.
- **Task Design Gate** — артефакт существует, каждая задача имеет основание в Requirement Review, неразрешённые противоречия не превращены в критерии приёмки.
- **Jira Gate** — создание каждой задачи подтверждено Jira, получены ID.
- **Test Design Gate** — артефакт существует, требования имеют покрытие, отсутствующее поведение не выдумано.
- **Qase Gate** — создание каждой сущности подтверждено Qase, получены ID.
- **Final Gate** — все артефакты валидны, задачи и сущности фактически созданы, критических блокировок нет.

### Допустимые итоговые статусы

`SUCCESS` / `PARTIAL` / `FAILED` / `REQUIRES_HUMAN_DECISION`

`SUCCESS` запрещён, если хотя бы один обязательный этап не был успешно завершён.

## Ключевые принципы

- **Source of Truth:** Confluence — требования, Jira — задачи, Qase — тестовая модель. При конфликте источников — эскалация, а не самостоятельный выбор.
- **Anti-duplication:** перед созданием задач/тестов проверяются существующие сущности Jira и Qase; при полном покрытии новые сущности не создаются (эскалация пользователю, дополнение при подтверждении).
- **Context Management:** минимально достаточный контекст, компактные артефакты вместо полной истории.
- **Self-Refine:** каждый артефакт проверяется по Completion Criteria (максимум 2 итерации).
- **Escalation:** при критических противоречиях, отсутствии данных, рисках нарушения ограничений и т.п. останавливается зависимый этап, сохраняются успешные независимые результаты.
- **Безопасность:** пароли, API-токены и credentials не передаются между системами.

## Запуск

1. Настройте MCP-серверы в конфигурации opencode:
   - **jira-confluence** — доступ к Confluence/Jira (scope: read/write issue, page, comment).
   - **qase** — доступ к Qase (проект, например, код `QA` для `QAI-05-AEgorova`).
2. Убедитесь, что структура `.opencode/agents/`, `.opencode/skills/` и `AGENTS.md` на месте.
3. Запустите opencode в корне проекта и дайте задачу, например:

   > Получи требования из Confluence страницы с PageID `2326530`, выполни анализ, создай задачи в Jira и тестовую модель в Qase (проект QAI-05-AEgorova).

   Входными данными могут быть: ID страницы Confluence, связанная документация, ID/название Jira-проекта и Qase-проекта.

## Обработка ошибок

- Jira/Qase операция FAILED → результат не сообщается как успешный, фиксируется статус.
- Часть операций успешна → сохраняются успешные ID, итоговый статус `PARTIAL`.
- Автоматический rollback через удаление данных запрещён без отдельного разрешения.