# Architecture-first Documentation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Перестроить документацию так, чтобы русскоязычный `Architecture.md` нормативно определял технологически независимую архитектуру, а русскоязычный `Implementation.md` описывал проверяемое отображение архитектурных портов на конкретные технологии.

**Architecture:** Документация строится в направлении `Architecture → Implementation`: domain model, lifecycle, invariants и contracts не зависят от базы данных или execution platform. Технологические решения получают явный статус зрелости, критерии PoC и ссылки на первичную документацию; значимые принятые решения в дальнейшем фиксируются ADR.

**Tech Stack:** Markdown, Git, `rg`, стандартные shell-проверки, официальная документация выбранных open-source проектов.

**Spec:** `docs/superpowers/specs/2026-09-17-architecture-first-documentation-design.md`

## Global Constraints

- `Architecture.md` является нормативным и не содержит выбора PostgreSQL, Kubernetes, Kueue, controller-runtime, OSS-Fuzz, CASR, BuildKit, gVisor или других конкретных реализаций.
- `Implementation.md` полностью написан на русском языке; английский допустим только для названий проектов, API identifiers, enum values и общепринятых технических терминов.
- Control Plane владеет единственным авторитетным domain state; execution backend предоставляет только observed state.
- `ExecutionAttempt` заменяет прежнюю сущность `Instance` и используется для continuous и finite workloads.
- Основной stop criterion — отсутствие роста coverage в течение `N` часов активного выполнения каждого `FuzzJob`.
- Все внешние утверждения сопровождаются прямыми ссылками на первичные официальные источники; служебные маркеры вида `cite...` запрещены.
- Документы не содержат служебных заглушек или неопределённых обязательных решений.

---

### Task 1: Навигация и правила документации

**Files:**
- Create: `README.md`
- Create: `docs/adr/README.md`
- Reference: `docs/superpowers/specs/2026-09-17-architecture-first-documentation-design.md`

**Interfaces:**
- Consumes: согласованную структуру документации из design spec.
- Produces: точки входа и правила ADR, на которые ссылаются последующие документы.

- [ ] **Step 1: Зафиксировать отсутствие текущей навигации**

Run:

```bash
test -f README.md && test -f docs/adr/README.md
```

Expected: FAIL, потому что оба файла отсутствуют.

- [ ] **Step 2: Создать корневой README**

Создать `README.md` со следующими разделами:

```markdown
# Distributed Fuzzing Platform

## Назначение
## Статус проекта
## Документация
## Порядок принятия решений
```

В `Документация` добавить относительные ссылки на `Architecture.md`, `Implementation.md`, `docs/adr/` и согласованную design spec. Явно написать: архитектура нормативна; реализация не может изменять архитектурные инварианты и обязана либо соответствовать им, либо инициировать отдельное изменение архитектуры.

- [ ] **Step 3: Создать правила ADR**

Создать `docs/adr/README.md` со статусами `PROPOSED`, `ACCEPTED`, `SUPERSEDED`, `REJECTED` и шаблоном:

```markdown
# ADR-NNNN: Название

- Статус:
- Дата:
- Связанные архитектурные контракты:

## Контекст
## Рассмотренные варианты
## Решение
## Последствия
## Проверка соответствия архитектуре
```

Зафиксировать, что ADR выбирает реализацию порта, но не вводит новые domain entities или lifecycle transitions.

- [ ] **Step 4: Проверить навигацию**

Run:

```bash
test -s README.md
test -s docs/adr/README.md
rg -n 'Architecture\.md|Implementation\.md|docs/adr' README.md
rg -n 'PROPOSED|ACCEPTED|SUPERSEDED|REJECTED' docs/adr/README.md
git diff --check
```

Expected: все команды завершаются успешно; `rg` выводит ссылки и четыре ADR-статуса; `git diff --check` не выводит ошибок.

- [ ] **Step 5: Commit**

```bash
git add README.md docs/adr/README.md
git commit -m "docs: add documentation navigation and ADR policy"
```

---

### Task 2: Нормативная технологически независимая архитектура

**Files:**
- Modify: `Architecture.md`
- Reference: `docs/superpowers/specs/2026-09-17-architecture-first-documentation-design.md`

**Interfaces:**
- Consumes: все согласованные domain decisions из design spec.
- Produces: normative contracts и terminology, которыми обязан пользоваться `Implementation.md`.

- [ ] **Step 1: Запустить отрицательные проверки старого документа**

Run:

```bash
test "$(rg -c '^# ' Architecture.md)" -eq 1
! rg -n '\bInstance\b' Architecture.md
rg -n '^## .*Цели|^## .*Инварианты|^## .*Порты' Architecture.md
```

Expected: как минимум одна проверка FAIL — старый документ не имеет единственного H1, использует `Instance` и не содержит полного нормативного каркаса.

- [ ] **Step 2: Заменить структуру Architecture.md**

Документ должен начинаться с `# Архитектура распределённой fuzzing-платформы` и содержать ровно эти H2-разделы:

```markdown
## 1. Статус и область действия
## 2. Цели и ограничения
## 3. Термины и принципы
## 4. Trust boundaries и multi-tenancy
## 5. Domain model
## 6. Авторитетное, desired и observed state
## 7. Lifecycle и state machines
## 8. Команды, события и reconciliation
## 9. Build и артефакты
## 10. Execution model
## 11. Scheduling, admission и preemption
## 12. Corpus и snapshots
## 13. Coverage и условия завершения
## 14. Findings и triage
## 15. Security requirements
## 16. Observability
## 17. Порты и границы модулей
## 18. Failure semantics
## 19. Архитектурные инварианты
```

В разделе 1 пометить документ как нормативный и объяснить направление зависимости `Architecture → Implementation`. В разделе 2 перечислить goals и non-goals, включая отсутствие требования к микросервисам, конкретной БД и конкретному cluster backend.

- [ ] **Step 3: Описать ownership и domain entities**

В разделах 4–5 определить `Tenant`, `Team`, `Membership`, `Project`, `SourceRevision`, `FuzzTarget`, `Campaign`, `FuzzJob`, `Build`, `BuildArtifact`, `Task`, `ExecutionAttempt`, `ResourceLease`, `Corpus`, `CorpusSnapshot`, `FindingOccurrence`, `CrashReport` и `Finding`.

Для каждой сущности указать назначение и owner. Зафиксировать:

```text
Tenant → Project → Campaign → FuzzJob → ExecutionAttempt
Task → ExecutionAttempt
FindingOccurrence → CrashReport → Finding
```

Каждый Project принадлежит одному Tenant; межtenant-доступ запрещён; `ExecutionAttempt` является попыткой выполнения, а не зеркалом backend resource.

- [ ] **Step 4: Описать state machines**

Добавить отдельные таблицы transitions для Campaign, FuzzJob, ExecutionAttempt, Build и Task. В таблицах указывать current phase, trigger, next phase и обязательный side effect.

Campaign должна различать desired states `RUNNING`, `PAUSED`, `STOPPED` и observed phases, включая `PAUSING`, `DEGRADED`, `COMPLETED_WITH_ERRORS` и `FAILED`. ExecutionAttempt должен иметь terminal phases `SUCCEEDED`, `FAILED`, `LOST`, `CANCELLED`.

Pause/preemption: checkpoint best effort до deadline, затем остановка независимо от результата; resume использует последний успешно опубликованный snapshot.

- [ ] **Step 5: Описать consistency и execution contracts**

В разделах 6, 8 и 10 зафиксировать:

- единственный authoritative domain state принадлежит Control Plane;
- `generation`, `observedGeneration`, optimistic concurrency и idempotency key;
- атомарное изменение desired state и durable domain event;
- at-least-once delivery и обязательную идемпотентность consumers/adapters;
- retry создаёт новый ExecutionAttempt;
- opaque `BackendResourceRef` не проникает в domain semantics;
- API не ждёт завершения долгих операций.

- [ ] **Step 6: Описать build, resource и corpus contracts**

Зафиксировать модель `SourceRevision + BuildRecipe → Build → BuildArtifact`, content-addressed artifact, provenance и tenant isolation.

Описать три уровня scheduling: Domain Scheduler, Resource Admission, Execution Backend. `ResourceLease` должен включать workload, tenant, resource request, priority и expiry. Preemption является поддерживаемым контрактом, даже если policy отключена.

Описать immutable CorpusSnapshot, parent/checksums/compatibility key, private campaign snapshots и автоматический policy-driven `CorpusMergeTask` с ручным on-demand trigger.

- [ ] **Step 7: Описать coverage, findings и terminal semantics**

Определить `StopPolicy.noCoverageGrowthFor` как основной stop criterion. Окно считается в активном времени отдельно для каждого FuzzJob; queue/build/pause/preemption/recovery исключаются. При несовместимой instrumentation начинается новая coverage epoch.

Описать pipeline `FindingOccurrence → CrashReport → Finding`, версионирование нормализатора, deduplication и `REOPENED`.

Campaign завершается как `COMPLETED`, `COMPLETED_WITH_ERRORS` или `FAILED` по правилам design spec.

- [ ] **Step 8: Описать security, observability, ports и invariants**

Security ограничить архитектурными trust boundaries и `ExecutionPolicy` с классами `STANDARD`, `STRONG`, `DEDICATED`; не выбирать sandbox implementation.

Observability должен включать metrics, structured logs, traces, audit events и ограничения cardinality. Отдельно определить normalized coverage/fuzzing telemetry и infrastructure telemetry.

Перечислить contracts портов: repositories, EventPublisher, BuildExecutor, ExecutionBackend, ResourceAdmission, CorpusStore, ArtifactStore, CoverageAnalyzer, FindingNormalizer, FindingDeduplicator, TaskExecutor, IdentityProvider и FindingSink.

Завершить документ нумерованным списком cross-entity invariants и failure/recovery matrix для потери backend resource, checkpoint failure, duplicate event, stale update, admission loss и недоступности artifact storage.

- [ ] **Step 9: Проверить нормативный документ**

Run:

```bash
test "$(rg -c '^# ' Architecture.md)" -eq 1
test "$(rg -c '^## ' Architecture.md)" -eq 19
! rg -n '\bInstance\b|PostgreSQL|controller-runtime|Kueue|OSS-Fuzz|CASR|BuildKit|gVisor|Kata Containers|google/go-cloud' Architecture.md
rg -n 'ExecutionAttempt|COMPLETED_WITH_ERRORS|noCoverageGrowthFor|FindingOccurrence|ResourceLease|at least once' Architecture.md
! rg -n 'T[O]DO|T[B]D|F[I]XME|cite' Architecture.md
git diff --check
```

Expected: один H1, 19 H2, запрещённые термины отсутствуют, обязательные термины найдены, служебные маркеры отсутствуют, whitespace errors отсутствуют.

- [ ] **Step 10: Commit**

```bash
git add Architecture.md
git commit -m "docs: define technology-independent platform architecture"
```

---

### Task 3: Русскоязычное отображение архитектуры на реализации

**Files:**
- Modify: `Implementation.md`
- Reference: `Architecture.md`
- Reference: `docs/adr/README.md`

**Interfaces:**
- Consumes: названия entities, ports, lifecycle и invariants из нового `Architecture.md`.
- Produces: technology mapping, validation status, risks, PoC criteria и phased delivery plan.

- [ ] **Step 1: Зафиксировать проблемы старого Implementation.md**

Run:

```bash
rg -n 'cite' Implementation.md
rg -n '^# [0-9]+\. (Domain|Control|Scheduling|Execution|Build|Generic|Corpus|Finding|Reproduce|Coverage)' Implementation.md
```

Expected: обе команды находят нарушения — служебные citation markers и англоязычные заголовки.

- [ ] **Step 2: Заменить структуру Implementation.md**

Документ должен начинаться с `# Реализация распределённой fuzzing-платформы` и содержать разделы:

```markdown
## 1. Статус и назначение
## 2. Связь с архитектурой
## 3. Статусы технологических решений
## 4. Сводная карта портов
## 5. Persistence и события
## 6. Reconciliation process
## 7. ExecutionBackend
## 8. ResourceAdmission
## 9. BuildExecutor и BuildArtifact
## 10. Generic Fuzz Runner
## 11. CorpusStore и ArtifactStore
## 12. Findings и triage
## 13. Coverage и telemetry
## 14. TaskExecutor
## 15. Identity и authorization
## 16. Workload isolation
## 17. External integrations
## 18. Компоненты, не используемые как platform core
## 19. MVP и этапы развития
## 20. Conformance matrix
## 21. Реестр PoC и открытых технологических решений
## 22. Репозитории и первичные источники
```

Все пояснения и заголовки написать по-русски. Явно назвать документ ненормативным и указать, что конфликт разрешается в пользу `Architecture.md`.

- [ ] **Step 3: Ввести статусы решений и сводную карту**

Определить значения:

- `PROPOSED` — кандидат без достаточной проверки;
- `VALIDATING` — выполняется PoC;
- `VALIDATED` — критерии проверки выполнены и решение может получить ADR;
- `REJECTED` — кандидат не соответствует контракту.

Для каждого архитектурного порта указать candidate, status, rationale, основные риски и required validation. Не помечать интеграцию `VALIDATED`, если в репозитории нет результатов PoC.

- [ ] **Step 4: Исправить границу reconciliation**

Описать domain reconciliation как собственную application logic, не зависящую от controller-runtime. controller-runtime допускается только внутри Kubernetes adapter для watches, cache, leader election и lifecycle Kubernetes controllers.

Persistence technology оставить открытым решением; перечислить требуемые свойства из Architecture.md вместо выбора БД.

- [ ] **Step 5: Описать execution и admission candidates**

Для Kubernetes adapter указать mapping `ExecutionAttempt → Pod` и `Task → Kubernetes Job`, правила UID/retry и необходимость отключить либо наблюдать скрытые backend retries.

Kueue пометить `VALIDATING`: проверить long-running Pod admission, preemption/deletion semantics, ResourceLease mapping, fair sharing и tenant quotas. Сослаться на официальные страницы Kueue Concepts и Run Plain Pods.

- [ ] **Step 6: Описать build и runner candidates**

OSS-Fuzz и BuildKit пометить `VALIDATING`. Не называть `infra/helper.py` стабильным SDK: определить собственный versioned runner contract, pin commit/image digests и compatibility tests.

Разделить Build process и BuildArtifact. Добавить PoC-критерии: deterministic fingerprint, provenance, cache isolation, artifact restore, cancellation и timeout.

- [ ] **Step 7: Описать storage, findings и task execution**

Сохранить собственные `CorpusStore` и `ArtifactStore` interfaces. Go CDK blob указать как `PROPOSED`, а не как обязательный выбор. Проверить conditional writes, checksums, large-object streaming и provider parity.

CASR пометить `VALIDATING`; описать parser/normalizer boundary и отдельный security review сценариев, требующих ptrace или ослабления sandbox.

Для Kubernetes Job описать `backoffLimit`, `podFailurePolicy`, idempotency и отображение каждого физического запуска на ExecutionAttempt.

- [ ] **Step 8: Описать этапы и conformance matrix**

MVP должен включать минимальные trust-boundary controls, а не откладывать всю изоляцию. Advanced isolation profiles допускаются в следующих этапах.

Добавить таблицу с одной строкой на каждый port/invariant: architecture reference, implementation candidate, status, validation evidence и known gaps.

Реестр PoC должен включать минимум: persistence atomicity/outbox, Kubernetes attempt mapping, Kueue long-running/preemption, OSS-Fuzz runner compatibility, immutable corpus publish, CASR security boundary и coverage epoch compatibility.

- [ ] **Step 9: Заменить citations прямыми первичными ссылками**

Использовать прямые Markdown-ссылки минимум на:

```text
https://github.com/kubernetes-sigs/controller-runtime
https://kueue.sigs.k8s.io/docs/concepts/
https://kueue.sigs.k8s.io/docs/tasks/run/plain_pods/
https://kubernetes.io/docs/concepts/workloads/controllers/job/
https://google.github.io/oss-fuzz/getting-started/new-project-guide/
https://github.com/google/oss-fuzz/blob/master/infra/helper.py
https://github.com/moby/buildkit
https://github.com/google/go-cloud
https://github.com/ispras/casr
https://github.com/google/clusterfuzz
https://github.com/google/clusterfuzzlite
https://github.com/google/fuzzbench
https://gvisor.dev/docs/user_guide/quick_start/kubernetes/
https://github.com/kata-containers/kata-containers
```

Не использовать search-result URLs или внутренние citation IDs.

- [ ] **Step 10: Проверить Implementation.md**

Run:

```bash
test "$(rg -c '^# ' Implementation.md)" -eq 1
test "$(rg -c '^## ' Implementation.md)" -eq 22
! rg -n 'cite|T[O]DO|T[B]D|F[I]XME' Implementation.md
! rg -n '^# [0-9]+\. (Purpose|Summary|Domain|Control|Scheduling|Execution|Build|Corpus|Finding|Coverage)' Implementation.md
rg -n 'PROPOSED|VALIDATING|VALIDATED|REJECTED' Implementation.md
rg -n 'ExecutionAttempt|FindingOccurrence|ResourceLease|noCoverageGrowthFor' Implementation.md
git diff --check
```

Expected: один H1, 22 H2, нет служебных citation-маркеров и англоязычных старых заголовков, статусы и архитектурные термины присутствуют, whitespace errors отсутствуют.

- [ ] **Step 11: Commit**

```bash
git add Implementation.md
git commit -m "docs: map architecture ports to implementation candidates"
```

---

### Task 4: Междокументная проверка и финальная редактура

**Files:**
- Modify: `README.md`
- Modify: `Architecture.md`
- Modify: `Implementation.md`
- Modify: `docs/adr/README.md`

**Interfaces:**
- Consumes: завершённые документы Tasks 1–3.
- Produces: согласованный комплект документации без битых локальных ссылок и терминологических расхождений.

- [ ] **Step 1: Проверить обязательные файлы и ссылки навигации**

Run:

```bash
test -s README.md
test -s Architecture.md
test -s Implementation.md
test -s docs/adr/README.md
test -s docs/superpowers/specs/2026-09-17-architecture-first-documentation-design.md
rg -n 'Architecture\.md|Implementation\.md|docs/adr' README.md
```

Expected: все файлы существуют и непусты; README содержит три навигационные ссылки.

- [ ] **Step 2: Проверить общий словарь**

Run:

```bash
! rg -n '\bInstance\b' Architecture.md Implementation.md
rg -n 'ExecutionAttempt' Architecture.md Implementation.md
rg -n 'FindingOccurrence' Architecture.md Implementation.md
rg -n 'ResourceLease' Architecture.md Implementation.md
rg -n 'COMPLETED_WITH_ERRORS' Architecture.md Implementation.md
```

Expected: `Instance` отсутствует; четыре обязательных термина присутствуют в обоих документах.

- [ ] **Step 3: Проверить отсутствие technology leakage**

Run:

```bash
! rg -n 'PostgreSQL|controller-runtime|Kueue|OSS-Fuzz|CASR|BuildKit|gVisor|Kata Containers|google/go-cloud' Architecture.md
rg -n 'controller-runtime|Kueue|OSS-Fuzz|CASR|BuildKit' Implementation.md
```

Expected: технологии отсутствуют в Architecture.md и присутствуют только в Implementation.md.

- [ ] **Step 4: Провести ручную consistency review**

Сопоставить каждую строку conformance matrix в `Implementation.md` с реальным разделом `Architecture.md`. Исправить:

- несовпадающие названия entities и ports;
- lifecycle phase, отсутствующую в normative state table;
- реализацию, объявленную обязательной без архитектурного контракта;
- архитектурный port без implementation candidate или явного `PROPOSED` gap;
- ссылку на раздел, которого не существует.

- [ ] **Step 5: Выполнить финальные статические проверки**

Run:

```bash
! rg -n 'cite|T[O]DO|T[B]D|F[I]XME|X[X]X' README.md Architecture.md Implementation.md docs/adr/README.md
test "$(rg -c '^# ' README.md)" -eq 1
test "$(rg -c '^# ' Architecture.md)" -eq 1
test "$(rg -c '^# ' Implementation.md)" -eq 1
git diff --check
git status --short
```

Expected: служебные маркеры отсутствуют; в каждом основном документе ровно один H1; `git diff --check` не выводит ошибок; `git status --short` показывает только ожидаемые документные изменения до commit.

- [ ] **Step 6: Commit**

```bash
git add README.md Architecture.md Implementation.md docs/adr/README.md
git commit -m "docs: verify architecture and implementation consistency"
```
