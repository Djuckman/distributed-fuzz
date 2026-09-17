# Architecture-first документация distributed fuzzing platform

## Статус

Согласованный дизайн реструктуризации `Architecture.md` и `Implementation.md`.

## Цель

Сделать архитектуру системы нормативной и независимой от выбора технологий. Документ реализации должен следовать архитектуре и показывать, как конкретные компоненты удовлетворяют архитектурным контрактам, но не определять domain model или lifecycle системы.

## Принцип зависимости

```text
Architecture
  domain model
  invariants
  lifecycle
  consistency guarantees
  ports and contracts
  security requirements
          ↓
Implementation
  technology mapping
  adapters
  operational constraints
  validation and PoC results
```

`Architecture.md` не зависит от выбора базы данных, orchestrator, object storage, очереди, fuzzing engine или execution platform.

## Структура документации

```text
README.md
Architecture.md
Implementation.md
docs/adr/
```

### README.md

Кратко описывает назначение системы, текущий scope и навигацию по документации.

### Architecture.md

Нормативный русскоязычный документ. Содержит:

1. цели и non-goals;
2. термины и архитектурные принципы;
3. trust boundaries и multi-tenancy;
4. domain model и ownership;
5. desired и observed state;
6. lifecycle и state machines;
7. reconciliation и consistency guarantees;
8. build и artifacts;
9. execution model;
10. scheduling, admission и preemption;
11. corpus и snapshots;
12. coverage и stop policies;
13. findings и triage;
14. security requirements;
15. observability;
16. ports и module boundaries;
17. failure semantics и invariants.

### Implementation.md

Ненормативный русскоязычный документ. Содержит:

- отображение архитектурных портов на технологии;
- альтернативы и trade-offs;
- статус каждого решения: `PROPOSED`, `VALIDATING`, `VALIDATED` или `REJECTED`;
- требования к PoC;
- ограничения версий и лицензий;
- MVP и последующие этапы;
- conformance matrix относительно `Architecture.md`.

Названия сторонних проектов, API identifiers и общепринятые технические термины могут сохраняться на английском языке.

### ADR

Значимые технологические решения фиксируются отдельными ADR. ADR не меняет архитектурные инварианты и описывает выбор конкретной реализации порта.

## Авторитетное состояние

Control Plane владеет единственным авторитетным domain state.

Execution backend:

- исполняет запрошенные операции;
- сообщает observed state;
- может потерять или пересоздать физические ресурсы;
- не определяет lifecycle domain entities;
- не является source of truth.

Технология хранения metadata архитектурой не задаётся. Persistence ports должны обеспечивать атомарные изменения агрегатов, optimistic concurrency и идемпотентную обработку команд.

## Multi-tenancy и ownership

```text
Tenant
 ├── Teams
 ├── Projects
 └── Quotas

User ── membership/role ── Team
Team ── role ───────────── Project
```

Каждый `Project` принадлежит ровно одному `Tenant`. Доступ между tenants запрещён. Несколько teams одного tenant могут иметь разные роли в одном Project.

## Основная domain model

```text
Tenant → Project → Campaign → FuzzJob → ExecutionAttempt
                         │
                         └── Build → BuildArtifact

Corpus → CorpusSnapshot
FindingOccurrence → CrashReport → Finding
Task → ExecutionAttempt
Workload → ResourceLease → ExecutionBackend
```

### Campaign и FuzzJob

`Campaign` является пользовательской единицей управления. Она объединяет `FuzzJob` для выбранных targets, revision, build parameters, resource policy и stop policy.

`FuzzJob` является логической непрерывной fuzzing-задачей. Термин отличает domain entity от Kubernetes Job.

### ExecutionAttempt

`ExecutionAttempt` представляет конкретную попытку физического выполнения `FuzzJob` или конечной `Task`.

- Один backend resource UID соответствует одной попытке.
- Restart контейнера внутри того же backend resource остаётся той же попыткой.
- Новый backend resource создаёт новую попытку.
- Потеря backend resource завершает попытку, но не удаляет её историю.
- Retry всегда создаёт новый `ExecutionAttempt`.
- Backend-specific идентификатор хранится как opaque `BackendResourceRef`.

### Continuous и finite workloads

`FuzzJob` и конечные `Task` имеют разные domain lifecycle, но используют общий execution mechanism через `ExecutionAttempt`.

К конечным tasks относятся build, reproduce, minimize, normalization, coverage, corpus merge/prune, regression analysis и fix verification.

## Desired и observed state

Пользовательские команды изменяют desired state. Reconciler и adapters обновляют observed state.

Основные entities содержат:

- `generation`;
- `observedGeneration`;
- version для optimistic concurrency;
- structured conditions с `reason`, `message` и `lastTransitionAt`.

Campaign поддерживает desired states `RUNNING`, `PAUSED` и `STOPPED`. Observed phase отражает фактический процесс перехода, включая `BUILDING`, `QUEUED`, `PAUSING`, `DEGRADED` и terminal states.

Временная потеря одной попытки не переводит Campaign в `FAILED`. Частично успешная Campaign завершается как `COMPLETED_WITH_ERRORS`. `FAILED` означает, что Campaign не смогла получить полезное выполнение.

## Pause и preemption

Pause и preemption выполняют checkpoint best effort до заданного deadline. После deadline workload останавливается независимо от результата checkpoint. Resume использует последний успешно опубликованный snapshot.

Preemption является частью архитектурного контракта, но конкретная policy может быть отключена реализацией.

## Build model

```text
SourceRevision + BuildRecipe
            ↓
          Build
            ↓
      BuildArtifact
```

- `SourceRevision` является разрешённой immutable-ревизией.
- `BuildRecipe` содержит все значимые параметры сборки и ссылку на версию build environment.
- `Build` является процессом со своим lifecycle.
- `BuildArtifact` является immutable content-addressed результатом с digest и provenance.
- `FuzzJob` и `ExecutionAttempt` ссылаются на точный `BuildArtifact`.
- Повторное использование определяется fingerprint всех значимых входов.
- Между tenants artifacts и build cache по умолчанию не разделяются.

## Corpus model

`Corpus` является логической именованной линией, а `CorpusSnapshot` — immutable manifest с checksums, parent snapshot, compatibility key и ссылкой на создавшую его попытку.

Campaign публикует private snapshots. Несколько campaigns не изменяют общий mutable corpus.

Policy автоматически создаёт `CorpusMergeTask` по lifecycle event, интервалу или порогу накопленных изменений. Ручной запуск остаётся дополнительной возможностью. Merge/prune публикуют новый canonical snapshot атомарно; предыдущий snapshot сохраняется для rollback и retention policy.

## Findings model

```text
ExecutionAttempt
       │
       ▼
FindingOccurrence
       │ normalization
       ▼
CrashReport
       │ deduplication
       ▼
Finding
```

- `FindingOccurrence` — immutable-факт обнаружения с raw artifacts.
- `CrashReport` — версионированный результат нормализации.
- `Finding` — дедуплицированная пользовательская проблема.
- Повторное обнаружение исправленной проблемы может перевести Finding в `REOPENED`.
- Изменение нормализатора не изменяет исходный occurrence.

## Scheduling и admission

```text
Domain Scheduler
        ↓
Resource Admission
        ↓
Execution Backend
```

Domain Scheduler выбирает, что имеет право выполняться. Resource Admission применяет tenant/project quotas, priorities, fair sharing и preemption. Execution Backend выбирает физическое размещение.

`ResourceAdmission` возвращает `ResourceLease`, связанный с workload, tenant, resource request, priority и сроком действия. Lease освобождается при завершении или потере попытки.

## Coverage и завершение

Основной stop criterion — отсутствие роста coverage в течение `N` часов активного выполнения.

```text
StopPolicy {
    noCoverageGrowthFor
    maxActiveTime   // optional safety limit
    deadline        // optional absolute limit
}
```

Стагнация считается отдельно для каждого `FuzzJob`. Queue, build, pause, preemption и recovery не расходуют окно. Campaign завершается, когда все обязательные jobs завершились по stop policy или иному terminal condition.

Coverage growth определяется стабильным множеством features/edges для совместимой instrumentation. При несовместимой смене `BuildArtifact` начинается новая coverage epoch.

## Command и reconciliation flow

```text
API command
   ↓
validate + authorize
   ↓
atomically update desired state and durable event
   ↓
reconcilers and policies
   ↓
idempotent port calls
   ↓
update observed state
```

- API не ожидает завершения долгих операций.
- Команды используют idempotency key.
- Доставка событий имеет семантику at least once.
- Consumers и adapters обязаны быть идемпотентными.
- Exactly-once не является гарантией.
- Queue, polling и transactional outbox являются деталями реализации.

## Trust boundaries

Архитектура допускает взаимно недоверенных tenants и выполнение произвольного пользовательского кода, но не задаёт конкретный sandbox.

Обязательные инварианты:

- workload не получает credentials Control Plane;
- данные и операции всегда tenant-scoped;
- backend применяет `ExecutionPolicy`;
- поддерживаются isolation classes `STANDARD`, `STRONG` и `DEDICATED`;
- конкретные механизмы изоляции выбираются реализацией.

## Проверяемость

`Architecture.md` должен содержать:

- таблицы допустимых state transitions;
- список cross-entity invariants;
- failure и recovery scenarios;
- contracts портов;
- consistency и idempotency guarantees.

`Implementation.md` должен содержать:

- conformance matrix для портов;
- PoC criteria для спорных интеграций;
- ограничения и риски компонентов;
- ссылки на первичную документацию;
- отсутствие неподтверждённых формулировок о полной совместимости.

Документы проверяются на корректность Markdown, рабочие ссылки, единообразие терминов и отсутствие противоречий между нормативной архитектурой и реализацией.
