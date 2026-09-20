# Убор Tenant/multi-tenancy и isolation-модели — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Привести `Architecture.md`, `Implementation.md`, `README.md` и `docs/open-questions-devops-security.md` в соответствие с решением: код доверен, multi-tenancy и isolation-модель убираются, авторизация переходит на `User`/`Group`/`Membership`/`ProjectAccessGrant`, добавляется `CampaignBatch`.

**Architecture:** Это документационный репозиторий (markdown), кода и тестов нет. "Тестом" для каждого шага служит grep-проверка на отсутствие устаревших терминов и визуальная сверка внутренней непротиворечивости (нумерация инвариантов, перекрёстные ссылки между `Architecture.md` и `Implementation.md`).

**Tech Stack:** Markdown, Mermaid-диаграммы. Правки через точные `old_string`/`new_string` замены.

**Spec:** `docs/superpowers/specs/2026-09-20-remove-tenant-trust-model-design.md`

## Global Constraints

- Код (fuzz target) считается доверенным; multi-tenancy и isolation-классы (`ExecutionPolicy`, `STANDARD`/`STRONG`/`DEDICATED`, gVisor/Kata) убираются полностью, а не откладываются.
- `Project` остаётся под тем же именем (не переименовывается в `Repository`).
- Новые сущности: `User`, `Group`, `Membership` (`User` ↔ `Group`, без роли), `ProjectAccessGrant` (`Group` ↔ `Project`, несёт `revisionAllowList` + `capabilities`), `CampaignBatch` (опциональная группировка `Campaign`, только просмотр, не владеет `FuzzJob`).
- `ResourceLease`/observability используют `projectId`, `tenantId` убирается везде.
- Аудит и запрет передачи Control Plane credentials в workload остаются без изменений.
- Коммиты в git выполняются только по явному запросу пользователя — эта задача коммитов не делает автоматически.

---

### Task 1: `Architecture.md` — убрать Tenant/Team, переписать Access control, инварианты

**Files:**
- Modify: `Architecture.md`

**Interfaces:**
- Consumes: решения из spec (раздел "Решение", пп. 1–5) и из раздела "Изменения по файлам → Architecture.md".
- Produces: финальные названия/поля сущностей (`User`, `Group`, `Membership`, `ProjectAccessGrant`, `CampaignBatch`, `projectId`), на которые ссылается Task 2 (`Implementation.md`).

- [ ] **Step 1: §2 Цели — убрать tenant isolation**

old_string:
```
- Обеспечивать tenant isolation, воспроизводимость сборок и трассируемость результатов.
```
new_string:
```
- Обеспечивать авторизованный доступ к Project, воспроизводимость сборок и трассируемость результатов.
```

- [ ] **Step 2: §3 Принципы — убрать принцип про tenant context, ренумеровать**

old_string:
```
1. domain identity не зависит от identity инфраструктурного ресурса;
2. intent сохраняется до вызова внешнего адаптера;
3. внешние вызовы и обработка событий идемпотентны;
4. immutable facts и artifacts не переписываются;
5. tenant context обязателен на каждой границе;
6. восстановление выполняется reconciliation, а не изменением истории;
7. длительные операции асинхронны относительно API.
```
new_string:
```
1. domain identity не зависит от identity инфраструктурного ресурса;
2. intent сохраняется до вызова внешнего адаптера;
3. внешние вызовы и обработка событий идемпотентны;
4. immutable facts и artifacts не переписываются;
5. восстановление выполняется reconciliation, а не изменением истории;
6. длительные операции асинхронны относительно API.
```

- [ ] **Step 3: §4 — заменить "Trust boundaries и multi-tenancy" на "Access control"**

old_string (весь §4, включая заголовок):
```
## 4. Trust boundaries и multi-tenancy

Платформа рассматривает tenants как взаимно недоверенные стороны, а пользовательский код — как недоверенный workload. Основные trust boundaries проходят между клиентом и Control Plane, Control Plane и adapters, workload и Control Plane, workload и storage, а также между workloads разных tenants.

`Tenant` является верхней границей владения и изоляции. `Team` группирует субъектов внутри одного Tenant. `Membership` связывает identity с Team и ролью. Team получает явную роль для Project; принадлежность к Tenant сама по себе не даёт доступ ко всем Project.

Каждый `Project` принадлежит ровно одному `Tenant`. Межtenant-доступ запрещён, включая чтение metadata, artifacts, corpus, findings, telemetry и audit records. Любая команда, событие, ключ хранения, lease и backend operation обязаны нести проверенный tenant context. Делегирование доступа возможно только внутри Tenant и фиксируется audit event.

Control Plane владеет domain entities. Execution Backend владеет лишь своими физическими ресурсами и их telemetry. Artifact и corpus adapters хранят bytes и manifests, но их domain ownership и права доступа определяет Control Plane.
```
new_string:
```
## 4. Access control

Платформа исполняет доверенный код в едином trust domain: запускаемый fuzzing-workload не рассматривается как источник угрозы, отдельной изоляции между запусками не требуется.

Доступ к данным и операциям авторизуется на уровне `Project`. `User` — идентичность, подтверждённая `IdentityProvider`. `Group` объединяет несколько `User`; `Membership` связывает `User` и `Group` без собственной роли. `ProjectAccessGrant` связывает `Group` и конкретный `Project` и несёт два поля: `revisionAllowList` — допустимые для запуска revision/ref-паттерны исходного кода (по умолчанию только `master`), и `capabilities` — разрешённые действия в этом Project (как минимум `RUN_CAMPAIGN`, `MANAGE_OTHERS_CAMPAIGN`, `TRIAGE_FINDINGS`, `MANAGE_ACCESS`).

Создание `Campaign` на `SourceRevision` X в `Project` P требует, чтобы пользователь состоял в `Group`, имеющей `ProjectAccessGrant` на P с capability `RUN_CAMPAIGN`, и чтобы X попадал в `revisionAllowList` этого grant. Другие операции (pause/stop чужой Campaign, изменение triage state Finding, изменение самих grants) проверяются по соответствующей capability аналогично.

Control Plane владеет всеми domain entities. Execution Backend владеет лишь своими физическими ресурсами и их telemetry. Artifact и corpus adapters хранят bytes и manifests, но их domain ownership и права доступа определяет Control Plane.
```

- [ ] **Step 4: §5 Domain model table — заменить Tenant/Team/Membership/Project, добавить User/Group/ProjectAccessGrant/CampaignBatch**

old_string:
```
| Сущность | Назначение | Владелец |
|---|---|---|
| `Tenant` | Верхняя граница изоляции, quotas и политик | Control Plane; корневой aggregate |
| `Team` | Группа субъектов одного Tenant | `Tenant` |
| `Membership` | Версионированная связь identity, Team и роли | `Tenant` |
| `Project` | Контекст исходного кода, targets, политик и доступа | `Tenant` |
| `SourceRevision` | Разрешённая immutable-ревизия исходного кода | `Project` |
| `FuzzTarget` | Адресуемая fuzzing-точка входа и её runtime contract | `Project` |
| `Campaign` | Пользовательская единица управления запуском | `Project` |
| `FuzzJob` | Логическая непрерывная задача одного target в Campaign | `Campaign` |
| `Build` | Процесс получения artifacts для revision и recipe | `Project` |
| `BuildArtifact` | Immutable content-addressed результат Build | Исходный `Build`, фактически опубликовавший artifact; cache-hit Builds только ссылаются на него |
| `Task` | Конечная работа: build, reproduce, minimize, normalization, coverage, corpus merge/prune, regression analysis или fix verification | Сущность, которая запросила работу; ссылка обязательна |
| `ExecutionAttempt` | Одна историческая попытка физического выполнения FuzzJob или Task | Соответствующий `FuzzJob` или `Task` |
| `ResourceLease` | Ограниченное по времени admission-разрешение на ресурсы | Resource Admission от имени `Tenant` |
| `Corpus` | Именованная логическая линия входных данных | `Project` и `FuzzTarget` |
| `CorpusSnapshot` | Immutable manifest конкретного состояния Corpus | `Corpus` |
| `FindingOccurrence` | Immutable-факт обнаружения с raw artifacts | `FuzzJob` и породивший `ExecutionAttempt` |
| `CrashReport` | Версионированный результат нормализации occurrence | `FindingOccurrence` |
| `Finding` | Дедуплицированная пользовательская проблема и её triage state | `Project` |
```
new_string:
```
| Сущность | Назначение | Владелец |
|---|---|---|
| `Project` | Контекст исходного кода, targets, campaigns, corpus и доступа | Control Plane; корневой aggregate |
| `User` | Identity, подтверждённая `IdentityProvider` | — |
| `Group` | Набор `User` | Control Plane |
| `Membership` | Связь `User` и `Group`, без собственной роли | `Group` |
| `ProjectAccessGrant` | Связь `Group` и `Project`: `revisionAllowList` и `capabilities` | `Project` |
| `SourceRevision` | Разрешённая immutable-ревизия исходного кода | `Project` |
| `FuzzTarget` | Адресуемая fuzzing-точка входа и её runtime contract | `Project` |
| `Campaign` | Пользовательская единица управления запуском; ровно один `Project` | `Project` |
| `CampaignBatch` | Необязательная группировка нескольких `Campaign` (в т.ч. из разных `Project`) только для совместного просмотра статуса/находок | Control Plane |
| `FuzzJob` | Логическая непрерывная задача одного target в Campaign | `Campaign` |
| `Build` | Процесс получения artifacts для revision и recipe | `Project` |
| `BuildArtifact` | Immutable content-addressed результат Build | Исходный `Build`, фактически опубликовавший artifact; cache-hit Builds только ссылаются на него |
| `Task` | Конечная работа: build, reproduce, minimize, normalization, coverage, corpus merge/prune, regression analysis или fix verification | Сущность, которая запросила работу; ссылка обязательна |
| `ExecutionAttempt` | Одна историческая попытка физического выполнения FuzzJob или Task | Соответствующий `FuzzJob` или `Task` |
| `ResourceLease` | Ограниченное по времени admission-разрешение на ресурсы | Resource Admission от имени `Project` |
| `Corpus` | Именованная логическая линия входных данных | `Project` и `FuzzTarget` |
| `CorpusSnapshot` | Immutable manifest конкретного состояния Corpus | `Corpus` |
| `FindingOccurrence` | Immutable-факт обнаружения с raw artifacts | `FuzzJob` и породивший `ExecutionAttempt` |
| `CrashReport` | Версионированный результат нормализации occurrence | `FindingOccurrence` |
| `Finding` | Дедуплицированная пользовательская проблема и её triage state | `Project` |
```

- [ ] **Step 5: §5 ownership chain code block и абзац про CampaignBatch**

old_string:
```
```text
Tenant → Project → Campaign → FuzzJob → ExecutionAttempt
Task → ExecutionAttempt
Build → build Task → ExecutionAttempt  (только при фактическом исполнении)
FindingOccurrence → CrashReport → Finding
```

`Campaign` фиксирует выбранные targets, `SourceRevision`, build parameters, resource policy и stop policy. `FuzzJob` не привязан к worker или размещению. `ExecutionAttempt` является попыткой выполнения, а не зеркалом backend resource. Физический идентификатор хранится только как opaque `BackendResourceRef`.
```
new_string:
```
```text
Project → Campaign → FuzzJob → ExecutionAttempt
Task → ExecutionAttempt
Build → build Task → ExecutionAttempt  (только при фактическом исполнении)
FindingOccurrence → CrashReport → Finding
Group → ProjectAccessGrant → Project
```

`Campaign` фиксирует выбранные targets, `SourceRevision`, build parameters, resource policy и stop policy. `FuzzJob` не привязан к worker или размещению. `ExecutionAttempt` является попыткой выполнения, а не зеркалом backend resource. Физический идентификатор хранится только как opaque `BackendResourceRef`. `CampaignBatch` не владеет `FuzzJob` и не участвует в lifecycle входящих `Campaign` — это client-facing группировка для совместного просмотра.
```

- [ ] **Step 6: §6 — убрать "tenant" из списка обязательных полей aggregate**

old_string:
```
- tenant и owner references, проверяемые при каждом изменении.
```
new_string:
```
- owner references, проверяемые при каждом изменении.
```

- [ ] **Step 7: §9 — убрать Tenant-качественник у fingerprint reuse и удалить абзац про изоляцию tenant-кэша**

old_string:
```
Одинаковый fingerprint может переиспользовать только artifact, доступный тому же Tenant и прошедший integrity check.
```
new_string:
```
Одинаковый fingerprint может переиспользовать только artifact, прошедший integrity check.
```

old_string:
```
Artifacts и build cache разных tenants по умолчанию изолированы. Любое ослабление изоляции требует отдельного явно разрешённого policy с проверяемым отсутствием утечки; базовый контракт не требует такого совместного использования.

Публикация считается успешной только после проверки digest и доступности manifest.
```
new_string:
```
Публикация считается успешной только после проверки digest и доступности manifest.
```

- [ ] **Step 8: §11 — `ResourceLease` bullets: убрать `tenantId`**

old_string:
```
- ссылку на workload (`FuzzJob` или `Task`);
- `tenantId` и при необходимости `projectId`;
- нормализованный resource request;
```
new_string:
```
- ссылку на workload (`FuzzJob` или `Task`);
- `projectId`;
- нормализованный resource request;
```

- [ ] **Step 9: §15 Security requirements — полностью переписать**

old_string:
```
## 15. Security requirements

Каждая операция аутентифицируется, авторизуется в tenant/project scope и оставляет audit event для security-sensitive изменений. Workload не получает credentials Control Plane и не может выбирать tenant context, lease, isolation class или backend reference.

`ExecutionPolicy` задаёт запрошенный класс изоляции и ограничения capabilities. Поддерживаются классы:

- `STANDARD` — базовая изоляция недоверенного workload;
- `STRONG` — усиленная изоляция для повышенного риска;
- `DEDICATED` — исключительное размещение в выделенном security scope.

Архитектура не выбирает sandbox implementation. Execution Backend обязан либо доказуемо применить требуемый класс, либо отклонить запуск до исполнения. Понижение класса без новой авторизованной команды запрещено.

Secrets передаются workload только в минимально необходимом scope, не сохраняются в artifacts, logs или corpus и имеют ограниченный срок действия. Все storage operations проверяют tenant ownership; content address сам по себе не даёт права чтения.
```
new_string:
```
## 15. Security requirements

Каждая операция аутентифицируется, авторизуется через `ProjectAccessGrant` в scope конкретного `Project` и оставляет audit event для security-sensitive изменений. Workload не получает credentials Control Plane и не может выбирать lease или backend reference.

Запускаемый fuzzing-код рассматривается как доверенный: платформа не выбирает и не применяет отдельный класс изоляции workload, sandbox implementation или runtime restrictions поверх стандартного исполнения в кластере.

Secrets передаются workload только в минимально необходимом scope, не сохраняются в artifacts, logs или corpus и имеют ограниченный срок действия. Все storage operations проверяют ownership на уровне `Project`; content address сам по себе не даёт права чтения.
```

- [ ] **Step 10: §16 Observability — убрать `tenantId`, tenant-scoped → project-scoped**

old_string:
```
с согласованными correlation identifiers: `tenantId`, `projectId`, aggregate identity, `taskId`/`fuzzJobId`, `executionAttemptId` и operation identity.
```
new_string:
```
с согласованными correlation identifiers: `projectId`, aggregate identity, `taskId`/`fuzzJobId`, `executionAttemptId` и operation identity.
```

old_string:
```
Retention не разрешает преждевременное удаление, а чтение и экспорт audit records всегда tenant-scoped и отдельно авторизованы.
```
new_string:
```
Retention не разрешает преждевременное удаление, а чтение и экспорт audit records всегда project-scoped и отдельно авторизованы.
```

- [ ] **Step 11: §17 Ports — убрать "tenant-scoped" у Repositories и AuditStore**

old_string:
```
| Repositories (`TenantRepository`, `ProjectRepository`, `CampaignRepository`, `WorkloadRepository`, `FindingRepository`) | Загружать и атомарно сохранять tenant-scoped aggregates с optimistic concurrency; поддерживать изменение desired state вместе с durable event |
```
new_string:
```
| Repositories (`ProjectRepository`, `CampaignRepository`, `WorkloadRepository`, `FindingRepository`) | Загружать и атомарно сохранять project-scoped aggregates с optimistic concurrency; поддерживать изменение desired state вместе с durable event |
```

old_string:
```
| `AuditStore` | Идемпотентно добавлять immutable tenant-scoped audit records; обеспечивать integrity, retention и авторизованный доступ; fail closed без изменения security-sensitive state, если обязательную запись нельзя надёжно сохранить |
```
new_string:
```
| `AuditStore` | Идемпотентно добавлять immutable project-scoped audit records; обеспечивать integrity, retention и авторизованный доступ; fail closed без изменения security-sensitive state, если обязательную запись нельзя надёжно сохранить |
```

- [ ] **Step 12: §19 Инварианты — убрать №3, переписать №2 и №19, ренумеровать весь список**

old_string (весь список 1–19):
```
1. Control Plane является единственным владельцем authoritative domain state; observations инфраструктуры не заменяют его.
2. Каждый Project принадлежит ровно одному Tenant, а любой доступ к данным и операциям tenant-scoped.
3. Цепочка `Tenant → Project → Campaign → FuzzJob → ExecutionAttempt` не может пересекать tenant boundary.
4. Любой ExecutionAttempt принадлежит ровно одному FuzzJob или Task; retry создаёт новую попытку и сохраняет историю предыдущей.
5. ExecutionAttempt описывает попытку исполнения, а `BackendResourceRef` остаётся opaque и не участвует в domain identity или lifecycle rules.
6. Изменение desired state и durable domain event атомарны; внешние side effects выполняются только после сохранения intent.
7. Consumers, reconcilers и adapters идемпотентны при повторной доставке команд, событий и observations.
8. `observedGeneration` никогда не опережает `generation`, а stale update не откатывает более новое или terminal состояние.
9. FuzzJob и Task не запускаются без действующего ResourceLease того же Tenant и совместимого resource request.
10. BuildArtifact и CorpusSnapshot immutable, content-verified и становятся видимыми только после полной успешной публикации.
11. Canonical CorpusSnapshot меняется только через CAS по expected snapshot identity и version; conflict не заменяет ссылку и требует повторного merge с актуальным canonical input.
12. Потребляющие FuzzJob/Task и их ExecutionAttempt ссылаются на точный `artifactDigest`; если сборка исполняется, build Task/attempt вместо будущего output фиксирует immutable SourceRevision, BuildRecipe, build environment и input references, а provenance опубликованного или найденного в cache результата ведёт к исходному исполнению.
13. Resume использует только последний успешно опубликованный совместимый CorpusSnapshot.
14. Окно `noCoverageGrowthFor` считается по active time отдельно для каждого FuzzJob и не включает queue, build, pause, preemption или recovery.
15. `artifactDigest` задаёт provenance, а `coverageCompatibilityKey` — семантическую совместимость; только несовместимый key начинает новую coverage epoch и сбрасывает её окно стагнации.
16. FindingOccurrence неизменяем; CrashReport и deduplication decisions версионированы; повтор исправленной проблемы может создать `REOPENED`.
17. Временная потеря одной попытки не переводит Campaign в `FAILED`; terminal outcome вычисляется по полезному исполнению и обязательным jobs.
18. Pause и preemption после checkpoint deadline обязательно останавливают workload; terminal attempt и release lease допустимы только после подтверждённой остановки либо execution fencing.
19. Ни один workload не получает credentials Control Plane и не может самостоятельно ослабить `ExecutionPolicy`.
```
new_string (18 пунктов):
```
1. Control Plane является единственным владельцем authoritative domain state; observations инфраструктуры не заменяют его.
2. Доступ к `Project` и операциям над его сущностями авторизуется через `ProjectAccessGrant` соответствующей `Group`.
3. Любой ExecutionAttempt принадлежит ровно одному FuzzJob или Task; retry создаёт новую попытку и сохраняет историю предыдущей.
4. ExecutionAttempt описывает попытку исполнения, а `BackendResourceRef` остаётся opaque и не участвует в domain identity или lifecycle rules.
5. Изменение desired state и durable domain event атомарны; внешние side effects выполняются только после сохранения intent.
6. Consumers, reconcilers и adapters идемпотентны при повторной доставке команд, событий и observations.
7. `observedGeneration` никогда не опережает `generation`, а stale update не откатывает более новое или terminal состояние.
8. FuzzJob и Task не запускаются без действующего ResourceLease того же Project и совместимого resource request.
9. BuildArtifact и CorpusSnapshot immutable, content-verified и становятся видимыми только после полной успешной публикации.
10. Canonical CorpusSnapshot меняется только через CAS по expected snapshot identity и version; conflict не заменяет ссылку и требует повторного merge с актуальным canonical input.
11. Потребляющие FuzzJob/Task и их ExecutionAttempt ссылаются на точный `artifactDigest`; если сборка исполняется, build Task/attempt вместо будущего output фиксирует immutable SourceRevision, BuildRecipe, build environment и input references, а provenance опубликованного или найденного в cache результата ведёт к исходному исполнению.
12. Resume использует только последний успешно опубликованный совместимый CorpusSnapshot.
13. Окно `noCoverageGrowthFor` считается по active time отдельно для каждого FuzzJob и не включает queue, build, pause, preemption или recovery.
14. `artifactDigest` задаёт provenance, а `coverageCompatibilityKey` — семантическую совместимость; только несовместимый key начинает новую coverage epoch и сбрасывает её окно стагнации.
15. FindingOccurrence неизменяем; CrashReport и deduplication decisions версионированы; повтор исправленной проблемы может создать `REOPENED`.
16. Временная потеря одной попытки не переводит Campaign в `FAILED`; terminal outcome вычисляется по полезному исполнению и обязательным jobs.
17. Pause и preemption после checkpoint deadline обязательно останавливают workload; terminal attempt и release lease допустимы только после подтверждённой остановки либо execution fencing.
18. Ни один workload не получает credentials Control Plane.
```

- [ ] **Step 13: Verify**

Run: `grep -in "tenant" Architecture.md`
Expected: no output (0 matches).

Run: `grep -in "executionpolicy\|STANDARD\|STRONG\|DEDICATED\|gvisor\|kata" Architecture.md`
Expected: no output.

---

### Task 2: `Implementation.md` — привести в соответствие с Task 1

**Files:**
- Modify: `Implementation.md`

**Interfaces:**
- Consumes: сущности/поля из Task 1 (`User`, `Group`, `Membership`, `ProjectAccessGrant`, `projectId`), новую нумерацию инвариантов 1–18 из `Architecture.md` §19.
- Produces: обновлённая матрица соответствия и PoC-реестр, консистентные с `Architecture.md`.

- [ ] **Step 1: §3 license-таблица — убрать строки gVisor и Kata Containers**

old_string:
```
| gVisor | [LICENSE](https://github.com/google/gvisor/blob/master/LICENSE) | [официальная установка](https://gvisor.dev/docs/user_guide/install/), [выпуски](https://github.com/google/gvisor/releases) | Release и runtime image digest не выбраны | License review, security review и compatibility/performance PoC |
| Kata Containers | [LICENSE](https://github.com/kata-containers/kata-containers/blob/main/LICENSE) | [модель выпусков](https://github.com/kata-containers/kata-containers/blob/main/docs/Release-Process.md), [поддерживаемые версии](https://github.com/kata-containers/kata-containers/security) | Release, runtime и guest image digests не выбраны | License review включая bundled dependencies, security review и compatibility/performance PoC |
```
new_string: (пусто — строки удаляются)

- [ ] **Step 2: §8 ResourceAdmission — tenant → project**

old_string:
```
Kueue имеет статус `PROPOSED`. Его официальная модель включает `ClusterQueue`, tenant-oriented `LocalQueue`, quota reservation, fair sharing и preemption
```
new_string:
```
Kueue имеет статус `PROPOSED`. Его официальная модель включает `ClusterQueue`, `LocalQueue` на уровне `Project`, quota reservation, fair sharing и preemption
```

old_string:
```
- `ResourceLease` сохраняет workload, `tenantId`, optional `projectId`, normalized resource request, priority, policy decision reference, `issuedAt`, `expiresAt` и lease identity;
```
new_string:
```
- `ResourceLease` сохраняет workload, `projectId`, normalized resource request, priority, policy decision reference, `issuedAt`, `expiresAt` и lease identity;
```

old_string:
```
- fair sharing между tenants, project quotas, priority и запрет cross-tenant borrowing без явной policy;
```
new_string:
```
- fair sharing между Project, priority и запрет borrowing квоты чужого Project без явной policy;
```

old_string:
```
- `Resource Admission` обязан валидировать и при необходимости клэмпить объявленный профиль по tenant `resource policy` до создания Kueue `Workload` — так же, как Kubernetes `LimitRange` подставляет default и отклоняет значения вне диапазона до `ResourceQuota`/`ClusterQueue` nominal quota;
- отсутствие профиля в репозитории получает safe default той же tenant policy, а не отказ в приёме заявки.
```
new_string:
```
- `Resource Admission` обязан валидировать и при необходимости клэмпить объявленный профиль по `resource policy` конкретного `Project` до создания Kueue `Workload` — так же, как Kubernetes `LimitRange` подставляет default и отклоняет значения вне диапазона до `ResourceQuota`/`ClusterQueue` nominal quota;
- отсутствие профиля в репозитории получает safe default той же project policy, а не отказ в приёме заявки.
```

- [ ] **Step 3: §12 CASR — убрать привилегированный isolation-review**

old_string:
```
Отдельный security review обязателен для режимов, требующих GDB/core analysis. Документация CASR указывает, что Docker-сценарий с GDB использует `SYS_PTRACE` и `seccomp=unconfined` ([CASR usage](https://github.com/ispras/casr#usage)). Такой workload не получает автоматического исключения из `ExecutionPolicy`: он запускается в отдельном isolation profile после оценки threat model либо соответствующая capability отклоняется.
```
new_string:
```
Docker-сценарий CASR с GDB использует `SYS_PTRACE` и `seccomp=unconfined` ([CASR usage](https://github.com/ispras/casr#usage)) — отдельного isolation profile или security review это не требует, так как запускаемый код доверенный.
```

- [ ] **Step 4: §15 Identity и authorization — переписать под User/Group/ProjectAccessGrant**

old_string:
```
## 15. Identity и authorization

Кандидат `IdentityProvider` — OIDC adapter со статусом `PROPOSED`; protocol contract определяется [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html). Adapter подтверждает issuer, subject, audience, signature, expiry и memberships/claims, но не принимает domain authorization decisions.

Authorization остаётся application logic Control Plane:

- каждый `Project` принадлежит одному `Tenant`;
- `Membership` связывает identity с Team и versioned role;
- Team получает явную роль на Project, а membership в Tenant не даёт автоматический доступ ко всем Projects;
- command, event, storage key, lease и backend operation несут проверенный tenant context;
- изменение роли, delegation и security-sensitive action создают immutable audit event через `AuditStore`; сбой append оставляет authoritative state неизменным.

PoC проверяет key rotation, expired/revoked token, issuer/audience confusion, stale membership, concurrent role update, cross-tenant identifiers и повтор idempotency key другим principal. Service identities получают минимальные short-lived credentials; workload не получает Control Plane credentials.
```
new_string:
```
## 15. Identity и authorization

Кандидат `IdentityProvider` — OIDC adapter со статусом `PROPOSED`; protocol contract определяется [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html). Adapter подтверждает issuer, subject, audience, signature, expiry и claims идентичности `User`, но не принимает domain authorization decisions.

Authorization остаётся application logic Control Plane:

- `User` объединяются в `Group` через `Membership`, без собственной роли на уровне membership;
- `ProjectAccessGrant` связывает `Group` и `Project` и несёт `revisionAllowList` (допустимые revision/ref-паттерны, default — только `master`) и `capabilities` (`RUN_CAMPAIGN`, `MANAGE_OTHERS_CAMPAIGN`, `TRIAGE_FINDINGS`, `MANAGE_ACCESS`);
- создание `Campaign` на `SourceRevision` X проверяет, что у пользователя есть `ProjectAccessGrant` на целевой `Project` с `RUN_CAMPAIGN` и X в `revisionAllowList`; другие операции проверяются по соответствующей capability;
- command, event, storage key, lease и backend operation несут проверенный `projectId`;
- изменение `ProjectAccessGrant`, delegation и security-sensitive action создают immutable audit event через `AuditStore`; сбой append оставляет authoritative state неизменным.

PoC проверяет key rotation, expired/revoked token, issuer/audience confusion, stale membership, concurrent изменение `ProjectAccessGrant`, enforcement `revisionAllowList` (запрет запуска с revision вне allow-list) и повтор idempotency key другим principal. Service identities получают минимальные short-lived credentials; workload не получает Control Plane credentials.
```

- [ ] **Step 5: §16 Workload isolation — заменить целиком коротким разделом**

old_string:
```
## 16. Workload isolation

MVP включает обязательные trust-boundary controls, а не откладывает изоляцию целиком:

- отдельные tenant-scoped namespaces/service accounts и deny-by-default network policy;
- `restartPolicy: Never`, read-only root filesystem где возможно, dropped capabilities, seccomp profile и запрет privilege escalation;
- resource requests/limits, ephemeral storage limits и controlled egress;
- short-lived scoped storage credentials без Control Plane access;
- image digests, admission policy и проверяемое применение `ExecutionPolicy` до запуска;
- отдельные storage prefixes/keys и build cache по Tenant.

Класс `STANDARD` должен быть реализован в MVP и проверен adversarial tests. Кандидаты advanced profiles имеют статус `PROPOSED`: gVisor запускает Kubernetes Pods через `RuntimeClass` ([официальный Kubernetes quick start gVisor](https://gvisor.dev/docs/user_guide/quick_start/kubernetes/)); Kata Containers использует lightweight VMs для усиленной isolation ([репозиторий Kata Containers](https://github.com/kata-containers/kata-containers)).

`STRONG` и `DEDICATED` могут развиваться на следующих этапах, но backend обязан отклонить workload, если запрошенный class фактически не обеспечен. Понижение class без новой авторизованной команды запрещено. CASR/GDB profiles с `ptrace` не смешиваются с обычным fuzzing pool без отдельного ADR и security evidence.
```
new_string:
```
## 16. Workload isolation

Запускаемый fuzzing-код считается доверенным: платформа не применяет отдельный isolation profile, network segmentation или security-ориентированные runtime restrictions поверх стандартного выполнения workload в кластере.

MVP запускает `FuzzJob`/`Task` как обычный Kubernetes Pod: `restartPolicy: Never`, стандартные resource requests/limits и ephemeral storage limits для предсказуемого scheduling, short-lived scoped storage credentials для `ArtifactStore`/`CorpusStore` и отдельные storage prefixes/keys по `Project` для организации данных (не как security-контроль).
```

- [ ] **Step 6: §17 External integrations — tenant/project scope → project scope**

old_string:
```
Outbound record содержит stable operation key, tenant/project scope, Finding version, redacted payload и ссылку на audit decision.
```
new_string:
```
Outbound record содержит stable operation key, project scope, Finding version, redacted payload и ссылку на audit decision.
```

- [ ] **Step 7: §19 MVP roadmap — первый этап**

old_string:
```
- Минимальные trust-boundary controls класса `STANDARD`, tenant authorization и отсутствие Control Plane credentials в workload.
```
new_string:
```
- Авторизация доступа к `Project` через `Group`/`ProjectAccessGrant` и отсутствие Control Plane credentials в workload.
```

- [ ] **Step 8: §19 MVP roadmap — второй этап, убрать advanced isolation**

old_string:
```
- Доказанные fair sharing, tenant quotas и preemption semantics Kueue.
- Reproduce, minimize, regression, fix verification и re-normalization tasks.
- Advanced isolation profiles `STRONG` и `DEDICATED` после security/performance PoC gVisor/Kata.
- Несколько storage providers после parity suite.
```
new_string:
```
- Доказанные fair sharing, project quotas и preemption semantics Kueue.
- Reproduce, minimize, regression, fix verification и re-normalization tasks.
- Несколько storage providers после parity suite.
```

- [ ] **Step 9: §20 Матрица соответствия — таблица "Порты", обновить номера инвариантов**

old_string:
```
| Repositories | §17, invariant 6 | Собственные persistence adapters | `PROPOSED` | Результатов PoC нет | Product/schema открыты; atomicity не доказана |
| `EventPublisher` | §8, §17, invariants 6–7 | Transactional outbox relay | `PROPOSED` | Критерии crash-injection определены; versioned scope и результатов нет | Transport/order/dedup store открыты |
| `BuildExecutor` | §9, §17, invariant 12 | OSS-Fuzz + BuildKit adapter | `PROPOSED` | Критерии fingerprint/cache-hit/provenance определены; versioned scope и результатов нет | Cache isolation, input/output manifests и cancellation не доказаны |
| `ExecutionBackend` | §10, §17, invariants 4–5 | Kubernetes adapter | `PROPOSED` | Attempt mapping, create-timeout lookup и fencing scenarios определены; versioned scope и результатов нет | Duplicate/replacement behavior не проверен |
| `ResourceAdmission` | §11, §17, invariant 9 | Kueue adapter | `PROPOSED` | Long-running/preemption/no-attempt revoke scenario определён; versioned scope и результатов нет | Lease expiry и quota mapping не доказаны |
| `CorpusStore` | §12, §17, invariants 10–11, 13 | Собственный interface; Go CDK candidate | `PROPOSED` | Delta gate и finite CAS retry criteria определены; versioned scope и результатов нет | Provider CAS/parity открыты |
| `ArtifactStore` | §9, §17, invariant 10 | Собственный interface; Go CDK candidate | `PROPOSED` | Результатов PoC нет | Atomic visibility/large objects не проверены |
| `CoverageAnalyzer` | §13, §17, invariants 14–15 | Собственный normalizer | `PROPOSED` | Compatibility, telemetry-gap и active-time scenarios определены; versioned scope и результатов нет | Feature stability и interval dedup не доказаны |
| `FindingNormalizer` | §14, §17, invariant 16 | CASR adapter | `PROPOSED` | Golden/security criteria определены; versioned scope и результатов нет | Format drift и privileged modes не проверены |
| `FindingDeduplicator` | §14, §17, invariant 16 | Versioned platform policy + CASR signals | `PROPOSED` | Replay criteria определены; versioned scope и результатов нет | Merge/split accuracy не измерена |
| `TaskExecutor` | §7, §10, §17, invariant 4 | Kubernetes Job adapter | `PROPOSED` | Bounded retry, quiescence и rejection scenarios определены; versioned scope и результатов нет | Hidden backend retries не исключены |
| `IdentityProvider` | §4, §15, §17, invariant 2 | OIDC adapter | `PROPOSED` | Результатов PoC нет | Provider, claims и revocation открыты |
```
new_string:
```
| Repositories | §17, invariant 5 | Собственные persistence adapters | `PROPOSED` | Результатов PoC нет | Product/schema открыты; atomicity не доказана |
| `EventPublisher` | §8, §17, invariants 5–6 | Transactional outbox relay | `PROPOSED` | Критерии crash-injection определены; versioned scope и результатов нет | Transport/order/dedup store открыты |
| `BuildExecutor` | §9, §17, invariant 11 | OSS-Fuzz + BuildKit adapter | `PROPOSED` | Критерии fingerprint/cache-hit/provenance определены; versioned scope и результатов нет | Cache isolation, input/output manifests и cancellation не доказаны |
| `ExecutionBackend` | §10, §17, invariants 3–4 | Kubernetes adapter | `PROPOSED` | Attempt mapping, create-timeout lookup и fencing scenarios определены; versioned scope и результатов нет | Duplicate/replacement behavior не проверен |
| `ResourceAdmission` | §11, §17, invariant 8 | Kueue adapter | `PROPOSED` | Long-running/preemption/no-attempt revoke scenario определён; versioned scope и результатов нет | Lease expiry и quota mapping не доказаны |
| `CorpusStore` | §12, §17, invariants 9–10, 12 | Собственный interface; Go CDK candidate | `PROPOSED` | Delta gate и finite CAS retry criteria определены; versioned scope и результатов нет | Provider CAS/parity открыты |
| `ArtifactStore` | §9, §17, invariant 9 | Собственный interface; Go CDK candidate | `PROPOSED` | Результатов PoC нет | Atomic visibility/large objects не проверены |
| `CoverageAnalyzer` | §13, §17, invariants 13–14 | Собственный normalizer | `PROPOSED` | Compatibility, telemetry-gap и active-time scenarios определены; versioned scope и результатов нет | Feature stability и interval dedup не доказаны |
| `FindingNormalizer` | §14, §17, invariant 15 | CASR adapter | `PROPOSED` | Golden/security criteria определены; versioned scope и результатов нет | Format drift не проверен |
| `FindingDeduplicator` | §14, §17, invariant 15 | Versioned platform policy + CASR signals | `PROPOSED` | Replay criteria определены; versioned scope и результатов нет | Merge/split accuracy не измерена |
| `TaskExecutor` | §7, §10, §17, invariant 3 | Kubernetes Job adapter | `PROPOSED` | Bounded retry, quiescence и rejection scenarios определены; versioned scope и результатов нет | Hidden backend retries не исключены |
| `IdentityProvider` | §4, §15, §17, invariant 2 | OIDC adapter | `PROPOSED` | Результатов PoC нет | Provider, claims и revocation открыты |
```

- [ ] **Step 10: §20 Матрица соответствия — таблица "Инварианты", полная замена**

old_string:
```
| Инвариант | Ссылка на архитектуру | Реализационный механизм | Статус | Свидетельства проверки | Известные пробелы |
|---|---|---|---|---|---|
| 1. Authoritative state только в Control Plane | §6, §19.1 | Repository + application reconciliation | `PROPOSED` | Результатов PoC нет | Persistence не выбрана |
| 2. Tenant-scoped ownership и access | §4, §19.2 | Tenant keys, authorization, storage policy | `PROPOSED` | Результатов adversarial tests нет | Identity provider открыт |
| 3. Ownership chain не пересекает Tenant | §5, §19.3 | Aggregate validation и foreign-key policy | `PROPOSED` | Результатов tests нет | Schema открыта |
| 4. Один owner на attempt; retry создаёт новый | §7, §10, §19.4 | Pod UID mapping и immutable history | `PROPOSED` | Scenario определён; versioned scope и результатов нет | Job replacement не проверен |
| 5. `BackendResourceRef` opaque | §10, §19.5 | Adapter value object | `PROPOSED` | Результатов boundary tests нет | Serialization открыта |
| 6. Desired state и event атомарны | §6, §8, §19.6 | Transactional outbox | `PROPOSED` | Crash matrix определена; versioned scope и результатов нет | Database открыта |
| 7. Idempotent consumers/reconcilers/adapters | §8, §19.7 | Operation keys и dedup records | `PROPOSED` | Duplicate scenarios определены; versioned scope и результатов нет | Retention window открыто |
| 8. Stale update не откатывает state | §6, §19.8 | Version/generation compare-and-set | `PROPOSED` | Результатов race tests нет | Persistence semantics открыты |
| 9. Запуск только с совместимым lease | §11, §19.9 | Admission gate перед backend start | `PROPOSED` | Kueue scenario определён; versioned scope и результатов нет | Expiry race не проверена |
| 10. Artifact/snapshot immutable и verified | §9, §12, §19.10 | Digest, manifest-ready transaction | `PROPOSED` | Immutable publish scenario определён; versioned scope и результатов нет | Provider semantics не проверены |
| 11. Canonical snapshot меняется только CAS | §12, §19.11 | Expected identity/version CAS | `PROPOSED` | Conflict scenario определён; versioned scope и результатов нет | CAS location открыта |
| 12. Consumers используют artifact; исполняемый build attempt — inputs | §9, §19.12 | Раздельные consuming/build manifests; verified cache hit без новой attempt | `PROPOSED` | Build provenance/cache-hit/input-output criteria определены; versioned scope и результатов нет | Runner compatibility не доказана |
| 13. Resume только с опубликованного compatible snapshot | §12, §19.13 | `corpusCompatibilityKey` и manifest readiness | `PROPOSED` | Resume scenarios определены; versioned scope и результатов нет | Conversion task не спроектирована |
| 14. `noCoverageGrowthFor` считает healthy active time на job | §13, §19.14 | Coverage ledger per `FuzzJob` + telemetry health gate | `PROPOSED` | Time-window/gap scenario определён; versioned scope и результатов нет | Lost telemetry policy не проверена |
| 15. Compatibility отделена от artifact provenance | §13, §19.15 | `artifactDigest` + versioned `coverageCompatibilityKey` | `PROPOSED` | Compatible/incompatible artifact matrix определена; versioned scope и результатов нет | Feature normalizer открыт |
| 16. Finding history immutable/versioned | §14, §19.16 | Occurrence store, versioned reports/rules | `PROPOSED` | CASR replay criteria определены; versioned scope и результатов нет | Dedup quality не измерена |
| 17. Потеря attempt не означает `Campaign FAILED` | §7, §13, §19.17 | Campaign aggregator application logic | `PROPOSED` | Результатов failure tests нет | Recovery thresholds открыты |
| 18. Stop после checkpoint deadline обязателен | §7, §11, §19.18 | Timed checkpoint, backend stop и fencing до terminal/release | `PROPOSED` | Preemption/StopPolicy/fencing scenarios определены; versioned scope и результатов нет | Backend timing не измерен |
| 19. Workload не получает Control Plane credentials | §15, §19.19 | Scoped workload identity и deny policy | `PROPOSED` | Результатов security tests нет | Runtime/isolation candidate открыт |
```
new_string:
```
| Инвариант | Ссылка на архитектуру | Реализационный механизм | Статус | Свидетельства проверки | Известные пробелы |
|---|---|---|---|---|---|
| 1. Authoritative state только в Control Plane | §6, §19.1 | Repository + application reconciliation | `PROPOSED` | Результатов PoC нет | Persistence не выбрана |
| 2. Доступ к Project авторизуется через ProjectAccessGrant | §4, §19.2 | `Group`/`ProjectAccessGrant`, IdentityProvider | `PROPOSED` | Результатов adversarial tests нет | Identity provider открыт |
| 3. Один owner на attempt; retry создаёт новый | §7, §10, §19.3 | Pod UID mapping и immutable history | `PROPOSED` | Scenario определён; versioned scope и результатов нет | Job replacement не проверен |
| 4. `BackendResourceRef` opaque | §10, §19.4 | Adapter value object | `PROPOSED` | Результатов boundary tests нет | Serialization открыта |
| 5. Desired state и event атомарны | §6, §8, §19.5 | Transactional outbox | `PROPOSED` | Crash matrix определена; versioned scope и результатов нет | Database открыта |
| 6. Idempotent consumers/reconcilers/adapters | §8, §19.6 | Operation keys и dedup records | `PROPOSED` | Duplicate scenarios определены; versioned scope и результатов нет | Retention window открыто |
| 7. Stale update не откатывает state | §6, §19.7 | Version/generation compare-and-set | `PROPOSED` | Результатов race tests нет | Persistence semantics открыты |
| 8. Запуск только с совместимым lease | §11, §19.8 | Admission gate перед backend start | `PROPOSED` | Kueue scenario определён; versioned scope и результатов нет | Expiry race не проверена |
| 9. Artifact/snapshot immutable и verified | §9, §12, §19.9 | Digest, manifest-ready transaction | `PROPOSED` | Immutable publish scenario определён; versioned scope и результатов нет | Provider semantics не проверены |
| 10. Canonical snapshot меняется только CAS | §12, §19.10 | Expected identity/version CAS | `PROPOSED` | Conflict scenario определён; versioned scope и результатов нет | CAS location открыта |
| 11. Consumers используют artifact; исполняемый build attempt — inputs | §9, §19.11 | Раздельные consuming/build manifests; verified cache hit без новой attempt | `PROPOSED` | Build provenance/cache-hit/input-output criteria определены; versioned scope и результатов нет | Runner compatibility не доказана |
| 12. Resume только с опубликованного compatible snapshot | §12, §19.12 | `corpusCompatibilityKey` и manifest readiness | `PROPOSED` | Resume scenarios определены; versioned scope и результатов нет | Conversion task не спроектирована |
| 13. `noCoverageGrowthFor` считает healthy active time на job | §13, §19.13 | Coverage ledger per `FuzzJob` + telemetry health gate | `PROPOSED` | Time-window/gap scenario определён; versioned scope и результатов нет | Lost telemetry policy не проверена |
| 14. Compatibility отделена от artifact provenance | §13, §19.14 | `artifactDigest` + versioned `coverageCompatibilityKey` | `PROPOSED` | Compatible/incompatible artifact matrix определена; versioned scope и результатов нет | Feature normalizer открыт |
| 15. Finding history immutable/versioned | §14, §19.15 | Occurrence store, versioned reports/rules | `PROPOSED` | CASR replay criteria определены; versioned scope и результатов нет | Dedup quality не измерена |
| 16. Потеря attempt не означает `Campaign FAILED` | §7, §13, §19.16 | Campaign aggregator application logic | `PROPOSED` | Результатов failure tests нет | Recovery thresholds открыты |
| 17. Stop после checkpoint deadline обязателен | §7, §11, §19.17 | Timed checkpoint, backend stop и fencing до terminal/release | `PROPOSED` | Preemption/StopPolicy/fencing scenarios определены; versioned scope и результатов нет | Backend timing не измерен |
| 18. Workload не получает Control Plane credentials | §15, §19.18 | Scoped workload identity и deny policy | `PROPOSED` | Результатов security tests нет | — |
```

- [ ] **Step 11: §21 PoC-реестр — убрать `POC-ISOLATION-001`, tenant policy → project policy**

old_string:
```
| `POC-BLOB-001` | Storage provider parity | Go CDK `blob` и native SDK alternatives | `PROPOSED` | Conditional writes, checksums, streaming, errors и authorization одинаково удовлетворяют contracts | Кандидаты provider, versioned scope и results path отсутствуют |
| `POC-ISOLATION-001` | Advanced isolation | gVisor/Kata | `PROPOSED` | Compatibility, escape surface, performance, checkpoint и operations измерены для `STRONG`/`DEDICATED` | Этап после MVP; versioned scope и results path отсутствуют |
| `POC-RESPROFILE-001` | Repo resource profile → admission clamp | Закрытый набор resource-профилей target'а + клэмп в `Resource Admission` | `PROPOSED` | Профиль из репозитория детерминированно маппится в normalized resource request; `Resource Admission` клэмпит или отклоняет профиль вне tenant policy до создания Kueue `Workload`; отсутствующий профиль получает safe default | Versioned scope, набор tiers и results path отсутствуют |

Открытыми остаются persistence product, audit storage/WORM mechanism, event transport, blob providers, OIDC provider, exact Kubernetes/Kueue versions, sandbox mapping, FindingSink providers и схема resource-профилей target'ов.
```
new_string:
```
| `POC-BLOB-001` | Storage provider parity | Go CDK `blob` и native SDK alternatives | `PROPOSED` | Conditional writes, checksums, streaming, errors и authorization одинаково удовлетворяют contracts | Кандидаты provider, versioned scope и results path отсутствуют |
| `POC-RESPROFILE-001` | Repo resource profile → admission clamp | Закрытый набор resource-профилей target'а + клэмп в `Resource Admission` | `PROPOSED` | Профиль из репозитория детерминированно маппится в normalized resource request; `Resource Admission` клэмпит или отклоняет профиль вне project policy до создания Kueue `Workload`; отсутствующий профиль получает safe default | Versioned scope, набор tiers и results path отсутствуют |

Открытыми остаются persistence product, audit storage/WORM mechanism, event transport, blob providers, OIDC provider, exact Kubernetes/Kueue versions, FindingSink providers и схема resource-профилей target'ов.
```

- [ ] **Step 12: §22 Референсы — убрать gVisor/Kata**

old_string:
```
- [gVisor Kubernetes Quick Start](https://gvisor.dev/docs/user_guide/quick_start/kubernetes/) — Kubernetes `RuntimeClass` integration candidate.
- [Kata Containers](https://github.com/kata-containers/kata-containers) — lightweight VM isolation candidate.
- [OpenID Connect Core]
```
new_string:
```
- [OpenID Connect Core]
```

(Обратить внимание: это частичное совпадение начала строки — если `old_string` неуникален, использовать более широкий контекст соседних строк списка референсов при применении.)

- [ ] **Step 13: Verify**

Run: `grep -in "tenant" Implementation.md`
Expected: no output.

Run: `grep -in "gvisor\|kata\|executionpolicy\|STANDARD\|STRONG\|DEDICATED" Implementation.md`
Expected: no output.

Run: `grep -n "§19\." Implementation.md`
Expected: номера после `§19.` совпадают с новым списком 1–18 из `Architecture.md` (сверить вручную построчно с таблицей "Инварианты").

---

### Task 3: `README.md` — оставшиеся правки (диаграммы уже добавлены ранее в этой сессии)

**Files:**
- Modify: `README.md`

- [ ] **Step 1: FAQ №2 — убрать multi-tenant изоляцию из списка причин**

old_string:
```
ClusterFuzz исторически построен под операционную модель и инфраструктурные допущения Google и не даёт из коробки multi-tenant изоляцию, управление ресурсами через квоты/lease и audit trail поверх произвольного Kubernetes-кластера — а это то, ради чего затевается собственный Control Plane.
```
new_string:
```
ClusterFuzz исторически построен под операционную модель и инфраструктурные допущения Google и не даёт из коробки управление ресурсами через квоты/lease и audit trail поверх произвольного Kubernetes-кластера — а это то, ради чего затевается собственный Control Plane.
```

- [ ] **Step 2: FAQ №4 — tenant policy → project policy**

old_string:
```
а `Resource Admission` клэмпит его по tenant policy до отправки в Kueue
```
new_string:
```
а `Resource Admission` клэмпит его по project policy до отправки в Kueue
```

- [ ] **Step 3: Таблица «Предлагаемая реализация» — Выделение ресурсов/Artifacts/Identity/Изоляция**

old_string:
```
| Выделение ресурсов | `ResourceLease`, admission gate, отображение tenant/project policy и собственный `ResourceAdmission` adapter | Kueue как кандидат для queues, quotas, fair sharing и preemption |
| Физическое исполнение | `ExecutionBackend`/`TaskExecutor` adapters, operation keys, mapping `ExecutionAttempt ↔ Pod UID`, fencing и сбор результатов | Kubernetes Pod/Job и scheduler; controller-runtime только внутри adapter |
| Сборка | Lifecycle `Build`, fingerprint, provenance, cache policy и `BuildExecutor` adapter | OSS-Fuzz и BuildKit как кандидаты внутри adapter |
| Artifacts и corpus | Собственные `ArtifactStore`/`CorpusStore` interfaces, tenant authorization, immutable manifests и canonical CAS | Go CDK `blob` или native provider SDK после parity PoC |
| Findings | Immutable occurrence/report history, versioned dedup policy и adapters | CASR как кандидат для parsing и normalization |
| Coverage | Coverage compatibility, epochs, active-time accounting и собственный normalizer | Fuzzer-specific tools только за adapter boundary |
| Identity и audit | Tenant/project policy, audit contract и append-only adapter | OIDC provider и WORM/persistence product ещё не выбраны |
| Изоляция | Выбор обязательного isolation profile и проверка capabilities до запуска | Kubernetes controls для MVP; gVisor/Kata — кандидаты последующих этапов |
```
new_string:
```
| Выделение ресурсов | `ResourceLease`, admission gate, отображение project policy и собственный `ResourceAdmission` adapter | Kueue как кандидат для queues, quotas, fair sharing и preemption |
| Физическое исполнение | `ExecutionBackend`/`TaskExecutor` adapters, operation keys, mapping `ExecutionAttempt ↔ Pod UID`, fencing и сбор результатов | Kubernetes Pod/Job и scheduler; controller-runtime только внутри adapter |
| Сборка | Lifecycle `Build`, fingerprint, provenance, cache policy и `BuildExecutor` adapter | OSS-Fuzz и BuildKit как кандидаты внутри adapter |
| Artifacts и corpus | Собственные `ArtifactStore`/`CorpusStore` interfaces, project authorization, immutable manifests и canonical CAS | Go CDK `blob` или native provider SDK после parity PoC |
| Findings | Immutable occurrence/report history, versioned dedup policy и adapters | CASR как кандидат для parsing и normalization |
| Coverage | Coverage compatibility, epochs, active-time accounting и собственный normalizer | Fuzzer-specific tools только за adapter boundary |
| Identity и audit | Project policy, audit contract и append-only adapter | OIDC provider и WORM/persistence product ещё не выбраны |
```

- [ ] **Step 4: Mermaid sequence-диаграмма (Уровень 3) — очередь tenant → project**

old_string:
```
    P->>Q: Зарегистрировать Workload<br/>на N ресурсов в очереди tenant
```
new_string:
```
    P->>Q: Зарегистрировать Workload<br/>на N ресурсов в очереди project
```

- [ ] **Step 5: RBAC-абзац — namespace на tenant/project → Project, без изоляционной аргументации**

old_string:
```
- Этому адаптеру нужен **service account с ограниченным RBAC**: право создавать и наблюдать Pod, Job и Kueue `Workload` в выделенных namespace (обычно свой namespace на tenant/project — это и есть требование изоляции).
```
new_string:
```
- Этому адаптеру нужен **service account с ограниченным RBAC**: право создавать и наблюдать Pod, Job и Kueue `Workload` в выделенных namespace (обычно свой namespace на Project — для операционного разделения ресурсов, а не security-изоляции).
```

- [ ] **Step 6: Verify**

Run: `grep -in "tenant" README.md`
Expected: no output.

Run: `grep -in "gvisor\|kata" README.md`
Expected: no output.

---

### Task 4: `docs/open-questions-devops-security.md` — убрать неактуальные пункты

**Files:**
- Modify: `docs/open-questions-devops-security.md`

- [ ] **Step 1: Заголовок и убрать "Онбординг Tenant"**

old_string:
```
# Открытые вопросы: инфраструктура, supply chain, изоляция

Ненормативный документ. Не изменяет `Architecture.md` и не вводит domain-сущности. Фиксирует вопросы DevOps/Security, не покрытые текущей документацией, для последующей проработки в ADR.

## 1. Инфраструктура кластера (требует проработки)

- **Онбординг Tenant.** Не описано, кто и как создаёт tenant-scoped namespace, service account и deny-by-default NetworkPolicy при появлении нового Tenant/Project — автоматизированный шаг платформы или ручная ops-процедура. Нужен ADR: либо platform-managed provisioning (отдельный adapter), либо явный ручной runbook.
- **Cluster autoscaling vs Kueue scheduling gates.**
```
new_string:
```
# Открытые вопросы: инфраструктура и supply chain

Ненормативный документ. Не изменяет `Architecture.md` и не вводит domain-сущности. Фиксирует вопросы DevOps/Security, не покрытые текущей документацией, для последующей проработки в ADR.

## 1. Инфраструктура кластера (требует проработки)

- **Cluster autoscaling vs Kueue scheduling gates.**
```

- [ ] **Step 2: Убрать раздел "Изоляция и privileged-режимы" целиком**

old_string:
```
### Изоляция и privileged-режимы

MVP не пытается поддержать `STRONG`/`DEDICATED` или privileged CASR/GDB profile:

- MVP ограничивается классом `STANDARD` (уже так решено в Implementation.md §16) — зафиксировать это как границу scope, а не временное упущение;
- CASR/GDB-режим с `SYS_PTRACE`/`seccomp=unconfined` полностью исключается из MVP fuzzing pool; без частичной изоляции для него сейчас — либо не поддерживать вовсе, либо выносить в отдельный полностью изолированный pool после отдельного security review, вне критического пути MVP;
- gVisor/Kata не оцениваются в рамках MVP; оценка полностью откладывается на второй этап (согласуется с Implementation.md §19).
```
new_string: (пусто — раздел удаляется)

- [ ] **Step 3: Verify**

Run: `grep -in "tenant\|isolation\|изоляц" docs/open-questions-devops-security.md`
Expected: no output (кроме заголовка, если случайно остался — проверить вручную).

---

### Task 5: Cross-file consistency review

**Files:** все четыре файла выше (только чтение/grep, без правок)

- [ ] **Step 1: Общий grep по репозиторию**

Run: `grep -rn "Tenant\|tenant" Architecture.md Implementation.md README.md docs/open-questions-devops-security.md`
Expected: no output.

- [ ] **Step 2: Проверка нумерации инвариантов между файлами**

Открыть `Architecture.md` §19 (18 пунктов) и `Implementation.md` §20 таблицу "Инварианты" (тоже 18 строк) — построчно сверить, что текст и номер совпадают 1:1.

- [ ] **Step 3: Проверка терминологии сущностей**

Run: `grep -n "Repository\b" Architecture.md Implementation.md README.md` (кроме слова «репозиторий» в общем смысле кода/git — не доменной сущности)
Expected: нет использования `Repository` как названия домен-сущности (только как обычное слово в контексте git, если вообще есть).

Run: `grep -c "ProjectAccessGrant" Architecture.md Implementation.md README.md`
Expected: >0 в каждом файле, использование согласовано (одно и то же название полей `revisionAllowList`/`capabilities` везде).

- [ ] **Step 4: Рендер-проверка Mermaid**

Прочитать оба новых mermaid-блока в README (доменная иерархия и авторизация) — визуально проверить синтаксис (совпадающие скобки subgraph/end, отсутствие незакрытых кавычек в labels).

- [ ] **Step 5: Итоговый отчёт**

Свести результаты Steps 1–4 в короткое сообщение пользователю: что подтверждено, что нашлось (если что-то нашлось — исправить и повторить проверку).
