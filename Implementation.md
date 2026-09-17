# Реализация распределённой fuzzing-платформы

## 1. Статус и назначение

Этот документ является ненормативным отображением утверждённой архитектуры на кандидаты технологий, адаптеры, operational constraints и проверки. Он не выбирает domain model, lifecycle или архитектурные инварианты.

Нормативным источником служит [Architecture.md](Architecture.md). При любом расхождении конфликт разрешается в пользу `Architecture.md`. Технологическое решение становится принятым только после успешной проверки и отдельного ADR по правилам [реестра ADR](docs/adr/README.md).

В репозитории пока нет зафиксированных versioned scopes выполняющихся PoC и нет результатов PoC с evidence location. Поэтому все кандидаты ниже имеют статус `PROPOSED`; статусы `VALIDATING` и `VALIDATED` не назначены. Версии зависимостей, commit и image digests, лицензии, security advisories и условия обновления должны быть зафиксированы до начала PoC и затем закреплены в ADR до production-внедрения.

## 2. Связь с архитектурой

Направление зависимости остаётся односторонним:

```text
Architecture.md
  domain model, lifecycle, invariants, ports
          ↓
Implementation.md
  candidates, adapters, risks, PoC criteria
          ↓
ADR
  подтверждённый выбор и зафиксированные ограничения
```

Control Plane хранит единственное authoritative domain state. Внешние компоненты сообщают observations и исполняют idempotent operations, но не определяют переходы `Campaign`, `FuzzJob`, `Task`, `Build` или `ExecutionAttempt`.

Критические отображения, которые реализация обязана сохранить:

- закрытые таблицы переходов `Campaign`, `FuzzJob`, `Task`, `Build` и `ExecutionAttempt` не расширяются состояниями Kubernetes, Kueue или иной инфраструктуры;
- retry, pause, preemption и admission loss не переиспользуют terminal attempt: остановленная попытка получает `CANCELLED`, а разрешённый retry конечной `Task` возвращает её в `QUEUED` и создаёт новый `ExecutionAttempt`;
- сборка имеет единственную цепочку `Build → build Task → ExecutionAttempt`; `Build` не создаёт параллельную попытку;
- `Campaign` получает `COMPLETED_WITH_ERRORS`, когда полезное fuzzing-исполнение и результаты сохранены, но часть обязательных jobs завершилась невосстановимой ошибкой;
- canonical corpus reference меняется только через compare-and-swap по expected snapshot identity и version;
- `BackendResourceRef` остаётся opaque и не становится domain identity;
- build attempt фиксирует immutable inputs без будущего output digest, а потребляющие attempts используют точный `artifactDigest`;
- `coverageCompatibilityKey` отделён от artifact provenance, и stop window сбрасывается только при несовместимости;
- обязательная запись `AuditStore` сохраняется immutable и fail closed до применения security-sensitive state.

## 3. Статусы технологических решений

Используются четыре значения:

- `PROPOSED` — кандидат без достаточной проверки;
- `VALIDATING` — PoC реально выполняется по зафиксированному в репозитории versioned scope с указанным evidence location;
- `VALIDATED` — критерии проверки выполнены и решение может получить ADR;
- `REJECTED` — кандидат не соответствует контракту.

Статус относится к конкретному кандидату и зафиксированной версии, а не к проекту вообще. Одних критериев или записи в реестре недостаточно для `VALIDATING`: обязательны versioned scope, ответственный запуск и путь для evidence. Переход в `VALIDATED` требует воспроизводимых результатов PoC в репозитории, проверки известных gaps и последующего ADR. Наличие официальной документации или успешное применение проекта в другой системе не считается evidence совместимости с этой архитектурой.

### Лицензии, версии и фиксация зависимостей

Таблица не выбирает версии и не заменяет юридическую или security-оценку. В текущем репозитории pin отсутствует для каждого кандидата; запись «не выбран» означает, что нельзя начинать versioned PoC, пока не зафиксированы tag/commit, image digest и совместимые зависимости.

| Кандидат или роль | Официальный источник лицензии/IPR | Официальный источник версий или совместимости | Состояние фиксации | Обязательное условие перед ADR |
|---|---|---|---|---|
| Kubernetes и controller-runtime | [Kubernetes LICENSE](https://github.com/kubernetes/kubernetes/blob/master/LICENSE), [controller-runtime LICENSE](https://github.com/kubernetes-sigs/controller-runtime/blob/main/LICENSE) | [Kubernetes version skew policy](https://kubernetes.io/releases/version-skew-policy/), [controller-runtime compatibility](https://github.com/kubernetes-sigs/controller-runtime#versioning-maintenance-and-compatibility) | Версии cluster API, client libraries и images не выбраны | License review, security review и проверка всей compatibility matrix |
| Kueue | [LICENSE](https://github.com/kubernetes-sigs/kueue/blob/main/LICENSE) | [установка](https://kueue.sigs.k8s.io/docs/getting-started/installation/), [выпуски](https://github.com/kubernetes-sigs/kueue/releases) | Версия controller и image digest не выбраны | License review, security review и проверка с выбранной версией Kubernetes |
| OSS-Fuzz runner assets | [LICENSE](https://github.com/google/oss-fuzz/blob/master/LICENSE) | [официальный репозиторий для commit pin](https://github.com/google/oss-fuzz), [официальное руководство](https://google.github.io/oss-fuzz/getting-started/new-project-guide/) | Commit и digests builder/runner images не выбраны | License review, security review и compatibility suite на всех pins |
| BuildKit | [LICENSE](https://github.com/moby/buildkit/blob/master/LICENSE) | [политика выпусков](https://github.com/moby/buildkit/blob/master/PROJECT.md), [выпуски](https://github.com/moby/buildkit/releases) | Release, daemon/client images и dependencies не выбраны | License review, security review и проверка API/runtime compatibility |
| Go CDK | [LICENSE](https://github.com/google/go-cloud/blob/master/LICENSE) | [выпуски](https://github.com/google/go-cloud/releases), [метаданные модуля](https://github.com/google/go-cloud/blob/master/go.mod) | Module version и providers не выбраны | License review каждого driver/provider, security review и parity suite |
| CASR | [LICENSE](https://github.com/ispras/casr/blob/master/LICENSE) | [выпуски](https://github.com/ispras/casr/releases) | Release/container digest и parser profiles не выбраны | License review, security review privileged profiles и golden suite |
| gVisor | [LICENSE](https://github.com/google/gvisor/blob/master/LICENSE) | [официальная установка](https://gvisor.dev/docs/user_guide/install/), [выпуски](https://github.com/google/gvisor/releases) | Release и runtime image digest не выбраны | License review, security review и compatibility/performance PoC |
| Kata Containers | [LICENSE](https://github.com/kata-containers/kata-containers/blob/main/LICENSE) | [модель выпусков](https://github.com/kata-containers/kata-containers/blob/main/docs/Release-Process.md), [поддерживаемые версии](https://github.com/kata-containers/kata-containers/security) | Release, runtime и guest image digests не выбраны | License review включая bundled dependencies, security review и compatibility/performance PoC |
| ClusterFuzz, ClusterFuzzLite и FuzzBench как источники и доноры | [ClusterFuzz LICENSE](https://github.com/google/clusterfuzz/blob/master/LICENSE), [ClusterFuzzLite LICENSE](https://github.com/google/clusterfuzzlite/blob/main/LICENSE), [FuzzBench LICENSE](https://github.com/google/fuzzbench/blob/master/LICENSE) | [ClusterFuzz releases](https://github.com/google/clusterfuzz/releases), официальные репозитории для commit pin: [ClusterFuzzLite](https://github.com/google/clusterfuzzlite), [FuzzBench](https://github.com/google/fuzzbench) | Revisions не выбраны; импорт кода не утверждён | License/security review обязателен до заимствования кода или ADR роли |
| OpenID Connect Core | [политика IPR OpenID Foundation](https://openid.net/wp-content/uploads/2024/10/OIDF-Policy_IPR-Policy_Final_2024-10-19.pdf) | [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html) | Adapter и provider не выбраны | License/IPR и security review выбранной реализации и provider |

## 4. Сводная карта портов

| Архитектурный порт | Кандидат реализации | Статус | Обоснование | Основные риски | Обязательная проверка |
|---|---|---|---|---|---|
| Repositories | Собственные adapters; persistence technology открыта | `PROPOSED` | Domain interfaces не должны зависеть от схемы конкретной БД | Нет доказательств atomic aggregate/event update, optimistic concurrency и tenant isolation | PoC атомарности, conflict handling, migrations и восстановления |
| `EventPublisher` | Transactional outbox relay; transport открыт | `PROPOSED` | Сохраняет событие вместе с intent и допускает at least once delivery | Дубликаты, нарушение aggregate order, потеря wake-up | Crash-injection PoC outbox и consumer deduplication |
| `BuildExecutor` | OSS-Fuzz conventions поверх BuildKit adapter | `PROPOSED` | Даёт исходную модель build environment и изолируемый build engine | Нестабильный helper interface, cache leakage, недетерминированный fingerprint | Совместимость runner, provenance, cache isolation, cancel/timeout |
| `ExecutionBackend` | Собственный Kubernetes adapter | `PROPOSED` | Kubernetes предоставляет физические Pod/Job resources, но не domain lifecycle | Скрытые retries, duplicate starts, stale watches, UID reuse assumptions | Attempt/Pod UID mapping, loss/retry, idempotent stop/checkpoint |
| `ResourceAdmission` | Kueue adapter | `PROPOSED` | Кандидат на quotas, priority, fair sharing и preemption | Kueue semantics не равны `ResourceLease`; удаление Pod при preemption | Long-running Pod, lease, tenant quotas, fair sharing и deletion PoC |
| `CorpusStore` | Собственный интерфейс; Go CDK `blob` как возможный byte adapter | `PROPOSED` | Domain CAS и compatibility rules нельзя делегировать generic blob API | Разные conditional writes и consistency у providers | CAS, checksums, streaming, provider parity и immutable publish |
| `ArtifactStore` | Собственный интерфейс; Go CDK `blob` как возможный byte adapter | `PROPOSED` | Нужны content addressing, integrity и tenant authorization | Partial visibility, большие объекты, cross-tenant dedup leakage | Atomic visibility, restore, checksums, streaming и authorization |
| `CoverageAnalyzer` | Собственный normalizer; OSS-Fuzz/FuzzBench как reference | `PROPOSED` | Stop policy требует backend-independent coverage epoch | Несовместимые instrumentation и двойной учёт telemetry | Совместимые разные artifacts, несовместимые keys и active-time accounting |
| `FindingNormalizer` | CASR за parser/normalizer boundary | `PROPOSED` | CASR умеет разбирать crash reports, но domain `CrashReport` принадлежит платформе | Format drift, nondeterminism, `ptrace` и sandbox weakening | Golden corpus, versioning, deterministic output и security review |
| `FindingDeduplicator` | Собственная versioned policy; CASR как один из signals | `PROPOSED` | Merge/split должен быть объясним и сохранять rule version | Нестабильные keys и ошибочный merge разных defects | Replay, merge/split evidence и `REOPENED` scenarios |
| `TaskExecutor` | Kubernetes Job adapter | `PROPOSED` | Job подходит как envelope конечной работы | Replacement Pods и backend retries могут скрыть новые attempts | `backoffLimit`, `podFailurePolicy`, Pod UID и idempotency PoC |
| `IdentityProvider` | OIDC adapter | `PROPOSED` | Authentication отделяется от domain authorization | Token validation, revocation и membership staleness | Issuer/audience/key rotation и tenant-membership tests |
| `FindingSink` | Собственные provider-specific adapters | `PROPOSED` | Внешний tracker не получает authoritative ownership | Duplicate issues, disclosure и credential scope | Idempotency, redaction, retry и reconciliation tests |
| `AuditStore` | Собственный append-only adapter; persistence technology открыта | `PROPOSED` | Обязательная audit-запись должна быть immutable и fail closed относительно security-sensitive state | Неатомарная запись, tampering, преждевременная retention deletion, cross-tenant read | Fault-injection PoC atomic/fail-closed append, integrity, retention и access control |

## 5. Persistence и события

Persistence technology намеренно не выбрана. Ни SQL product, ни document store, ни message broker не являются частью архитектурного контракта. Кандидат оценивается по свойствам:

- атомарно сохраняет изменение aggregate и соответствующий durable domain event;
- применяет optimistic concurrency по version token и не принимает stale `generation`/`observedGeneration`;
- обеспечивает tenant-scoped keys, queries, transactions, backups и audit access;
- сохраняет immutable facts и terminal history;
- позволяет восстанавливать outbox relay после сбоя без потери события;
- поддерживает aggregate identity и sequence для at least once delivery;
- допускает idempotent consumers с durable deduplication state;
- обеспечивает проверяемые backup, restore, schema migration и retention procedures.

Базовый кандидат публикации — transactional outbox, но конкретные schema, polling/change-feed mechanism, broker и delivery transport остаются открытыми. Exactly once не заявляется. PoC должен внедрять сбои до commit, после commit до publish, после publish до acknowledgement и во время повторной обработки. Успех означает либо отсутствие и state, и event, либо наличие обоих; после восстановления допускаются дубликаты, но не потеря и не нарушение порядка внутри aggregate.

Canonical corpus reference также требует optimistic transaction или эквивалентный compare-and-swap. Успешная загрузка bytes без условного обновления identity и version не удовлетворяет контракту `CorpusStore`.

### AuditStore

Кандидат `AuditStore` — собственный append-only adapter со статусом `PROPOSED`; конкретная persistence/WORM technology не выбрана. Security-sensitive transition допускается только если immutable audit record с operation key, actor, tenant scope, policy decision, target, result и timestamp сохранён в той же атомарной единице с authoritative state либо как доказуемая fail-closed предпосылка. Append повторяется идемпотентно, но существующая запись не перезаписывается.

PoC должен внедрять сбой до append, после append до state commit и во время retry; ни один сценарий не должен оставлять применённый security-sensitive state без обязательной записи. Отдельно проверяются tamper detection, запрет premature deletion по retention policy, tenant-scoped read/export, denied cross-tenant access, backup/restore integrity и недоступность audit backend. Scope, pins и evidence path пока отсутствуют, поэтому критерии не повышают статус выше `PROPOSED`.

## 6. Reconciliation process

Domain reconciliation реализуется как собственная application logic и не зависит от controller-runtime. Reconciler читает authoritative desired state и последнее применимое observed state, вычисляет действие по закрытой таблице переходов, сохраняет intent/event и только затем вызывает порт с устойчивым operation key.

[controller-runtime](https://github.com/kubernetes-sigs/controller-runtime) является набором Go libraries для Kubernetes controllers. Его допустимая граница — только Kubernetes adapter: watches, cache, leader election и lifecycle Kubernetes controllers. Domain packages не импортируют Kubernetes API types, controller-runtime или его reconciliation result.

Каждый application reconciler обязан:

1. отклонять stale observations и updates terminal entities;
2. безопасно повторять operation после timeout без предположения о результате предыдущего вызова;
3. обновлять `observedGeneration` только после подтверждённого применения intent;
4. сохранять structured condition для transient, permanent, conflict, stale и policy rejection;
5. ограничивать retries policy и не создавать нелегальный lifecycle transition.

Особенно важно, что потеря отдельного backend resource завершает текущий `ExecutionAttempt` как `LOST`, но не переводит автоматически `Campaign` в `FAILED`. StopPolicy, pause, preemption и admission loss завершают attempt как `CANCELLED`. После preemption/admission loss владеющий `FuzzJob` переходит в `RECOVERING`; конечная `Task` при разрешённом retry возвращается в `QUEUED`; новый запуск всегда получает новый attempt identity. `FuzzJob` становится `COMPLETED` по StopPolicy только после checkpoint/stop handshake, terminal attempt и освобождения lease.

## 7. ExecutionBackend

Кандидат — собственный Kubernetes adapter со статусом `PROPOSED`. Отображение для первой реализации:

| Понятие предметной области | Ресурс Kubernetes | Правило идентичности |
|---|---|---|
| `ExecutionAttempt` | Pod | Один Pod UID соответствует ровно одной попытке |
| Непрерывный `FuzzJob` | Long-running Pod | Replacement Pod всегда создаёт новый `ExecutionAttempt` |
| Конечная `Task` | Kubernetes Job | Job является envelope; каждый созданный Pod UID отображается на отдельный `ExecutionAttempt` |
| `BackendResourceRef` | Opaque adapter value | Содержит необходимые adapter identifiers, но domain не разбирает их структуру |

Pod name недостаточен для identity: adapter сохраняет UID и отклоняет observations другого UID. Container restart внутри того же Pod остаётся той же попыткой. Replacement Pod, даже созданный тем же Job, является новой физической попыткой и должен появиться в Control Plane до принятия его outputs.

Официальная документация Kubernetes предупреждает, что Job controller создаёт replacement Pod после failure и что даже при `parallelism=1`, `completions=1` и `restartPolicy=Never` программа иногда может стартовать дважды. Она также описывает default `backoffLimit=6` и `podFailurePolicy` ([Kubernetes Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)). Поэтому backend retries должны быть отключены либо полностью наблюдаемы:

- предпочтительный baseline — `restartPolicy: Never`, `backoffLimit: 0`, один active Pod и domain-managed retry;
- если backend retry разрешён ADR, adapter обязан наблюдать каждый Pod UID, не допускать replacement к исполнению до перехода `Task` через `QUEUED` и выдачи нового lease, создать отдельный `ExecutionAttempt` и исключить concurrent ownership одних outputs; если такую admission gate доказать нельзя, скрытый retry отключается;
- `podFailurePolicy` используется для классификации backend result, но не расширяет domain state machine;
- operation key, labels и owner references обеспечивают idempotent create/observe/stop; duplicate create не должен порождать неучтённый workload;
- удаление, watch gap или истёкшая observation переводят попытку в `LOST`/`CANCELLED` только по правилам `Architecture.md`, а не по одному строковому reason Kubernetes.

PoC обязан проверить API timeout после фактического create, watch restart, duplicate event, node loss, graceful termination deadline, checkpoint capability и сбор outputs до `SUCCEEDED`.

## 8. ResourceAdmission

Kueue имеет статус `PROPOSED`. Его официальная модель включает `ClusterQueue`, tenant-oriented `LocalQueue`, quota reservation, fair sharing и preemption ([Kueue Concepts](https://kueue.sigs.k8s.io/docs/concepts/)). Эти объекты не включаются в domain model: adapter отображает их observations в собственный `ResourceLease`.

Обязательное отображение:

- `ResourceLease` сохраняет workload, `tenantId`, optional `projectId`, normalized resource request, priority, policy decision reference, `issuedAt`, `expiresAt` и lease identity;
- Kueue admission не считается lease, пока Control Plane не сохранил все эти поля и не подтвердил их совместимость;
- revoke, expiry или preemption прекращают право на запуск, инициируют checkpoint/stop и сохраняют attempt history;
- quotas, cohorts и queues конфигурируются adapter-specific policy и не становятся domain identifiers.

Интеграция Plain Pods требует отдельного PoC. Официальная страница указывает, что admission снимает scheduling gate, а при preemption управляемый Pod завершается и удаляется; для Pod groups replacement остаётся обязанностью внешнего controller ([Run Plain Pods](https://kueue.sigs.k8s.io/docs/tasks/run/plain_pods/)). Требуется проверить:

- admission long-running Pod и устойчивость reservation в течение fuzzing interval;
- preemption/deletion semantics, checkpoint deadline и отображение текущего attempt в `CANCELLED` с причиной `PREEMPTED`;
- повторный admission только через новый `ResourceLease` и новый `ExecutionAttempt`;
- fair sharing между tenants, project quotas, priority и запрет cross-tenant borrowing без явной policy;
- lease expiry/renewal при потере Kueue observations;
- отсутствие скрытого restart, который обходит Control Plane.

## 9. BuildExecutor и BuildArtifact

OSS-Fuzz и BuildKit имеют статус `PROPOSED`. Они являются кандидатами внутри `BuildExecutor`, а не владельцами `Build` lifecycle.

Нормативное разделение сохраняется:

```text
SourceRevision + BuildRecipe
            ↓
          Build                 процесс и lifecycle
            ↓
       build Task
            ↓
     ExecutionAttempt           единственная физическая попытка
            ↓
      BuildArtifact             immutable content-addressed результат
```

OSS-Fuzz определяет project metadata, build environment, `$SRC`, `$WORK`, `$OUT`, sanitizer/engine combinations и локальные команды build/run/coverage ([официальное руководство OSS-Fuzz](https://google.github.io/oss-fuzz/getting-started/new-project-guide/)). Это полезная compatibility target, но не API платформы.

BuildKit позиционируется как toolkit для воспроизводимого преобразования source в artifacts и предоставляет caching, cache import/export, разные outputs и rootless mode ([репозиторий BuildKit](https://github.com/moby/buildkit)). Наличие возможностей не доказывает tenant isolation или детерминизм; это предмет PoC.

Fingerprint должен включать immutable `SourceRevision`, все значимые поля `BuildRecipe`, target, architecture, instrumentation, sanitizer, fuzzing engine, build environment image digest и версию runner contract. Cache по умолчанию tenant-scoped. `BuildArtifact` публикуется только после проверки digest и manifest availability; provenance связывает Tenant, Project, revision, recipe, Build, build Task и creator attempt.

Build Task и её `ExecutionAttempt` до публикации результата получают только immutable `SourceRevision`, `BuildRecipe`, pinned build-environment reference и остальные input references. Будущий `artifactDigest` не входит в их input manifest: `BuildExecutor` возвращает его как output после integrity check. Точный `artifactDigest` обязателен до запуска только для потребляющих `FuzzJob`/Task и их attempts.

Критерии PoC:

- одинаковые значимые inputs дают одинаковый deterministic fingerprint, а изменение каждого значимого поля меняет его;
- provenance однозначно восстанавливает полную цепочку до `SourceRevision` и `BuildRecipe`;
- cache и artifacts разных tenants изолированы, включая error/log paths;
- artifact можно восстановить по digest после очистки локального worker state;
- cancel и timeout останавливают build attempt, не публикуют partial artifact и сохраняют `CANCELLED`/`FAILED` по архитектурной причине;
- повтор после transient failure создаёт новый `ExecutionAttempt`, но не второй attempt от имени `Build`;
- corrupted output и несовпадающий digest блокируют `SUCCEEDED`;
- input manifest build attempt не содержит заранее известного output digest, а опубликованный digest появляется только в terminal result и provenance.

## 10. Generic Fuzz Runner

Платформа определяет собственный versioned runner contract. Файл [`infra/helper.py`](https://github.com/google/oss-fuzz/blob/master/infra/helper.py) остаётся reference implementation и command-line tool OSS-Fuzz; он не объявляется стабильным SDK платформы.

Runner contract должен фиксировать:

- версию protocol и machine-readable input/output manifests;
- команды `build`, `run`, `reproduce`, `minimize`, `coverage` и capability discovery;
- для consuming-команд — точный immutable `artifactDigest`; для `build` — immutable SourceRevision/BuildRecipe/build-environment/input references без будущего output digest;
- target, engine, sanitizer, resource limits, `coverageCompatibilityKey` и corpus snapshot inputs;
- typed output references для logs, findings, coverage и нового private snapshot;
- heartbeat, graceful stop, checkpoint deadline и terminal result;
- запрет прямого доступа к Control Plane credentials.

Для каждой поддерживаемой комбинации pin-ятся OSS-Fuzz commit, runner image digest, base-builder/base-runner image digests и toolchain versions. Compatibility suite запускает golden projects и проверяет build, run, crash reproduction, corpus import/export, coverage и cancellation. Обновление pin не допускается без повторного suite и сравнения manifests.

## 11. CorpusStore и ArtifactStore

`CorpusStore` и `ArtifactStore` остаются отдельными собственными interfaces. Generic blob library может переносить bytes, но не определяет domain authorization, lifecycle, canonical CAS или compatibility.

Go CDK предоставляет общий `blob` API и provider drivers, при этом сам проект называет APIs alpha ([репозиторий Go CDK](https://github.com/google/go-cloud)). Поэтому Go CDK `blob` имеет статус `PROPOSED`, а не является обязательным выбором.

Общие требования к adapters:

- потоковая запись и чтение больших объектов без загрузки всего содержимого в память;
- checksum во время transfer и повторная проверка перед publication;
- content-addressed immutable keys и запрет overwrite с иным содержимым;
- tenant authorization до lookup, чтобы знание digest не давало право чтения;
- partial uploads невидимы domain state и удаляются recovery/retention process;
- одинаковые semantics ошибок, retry и integrity для каждого поддержанного provider.

Дополнительный contract `CorpusStore`:

1. private campaign snapshot сначала полностью публикует objects и immutable manifest;
2. manifest содержит digest, checksums, parent, `corpusCompatibilityKey`, Tenant, Project, Corpus и creator attempt;
3. merge/prune читает immutable inputs и фиксирует expected canonical identity/version;
4. canonical reference меняется одним compare-and-swap;
5. при conflict ссылка не меняется: `CorpusMergeTask` перечитывает новый canonical snapshot, повторно вычисляет merge и повторяет publication в пределах retry policy;
6. предыдущий canonical snapshot остаётся доступным для rollback/retention.

PoC сравнивает conditional writes, checksums, large-object streaming, list/read-after-write behavior, cancellation и provider parity. Если выбранный blob provider не даёт требуемый CAS, canonical pointer хранится в persistence adapter с атомарной связью на уже проверенный immutable manifest; это фиксируется ADR.

## 12. Findings и triage

Pipeline остаётся `FindingOccurrence → CrashReport → Finding`. Raw occurrence immutable; повторная нормализация создаёт новый versioned `CrashReport`; deduplication decision хранит rule version и evidence.

CASR имеет статус `PROPOSED`. Официальный проект предоставляет parsers для sanitizer/GDB outputs, JSON crash reports, stacktrace triage, deduplication и clustering ([репозиторий CASR](https://github.com/ispras/casr)). Интеграция проходит через собственную границу:

```text
immutable FindingOccurrence
  → CASR parser adapter
  → platform-normalized CrashReport schema
  → versioned FindingDeduplicator policy
  → Finding
```

CASR format и cluster identity не становятся domain schema или постоянным `Finding` key. Adapter сохраняет `normalizerName`, `normalizerVersion`, input artifact digests и diagnostics. Golden tests проверяют deterministic normalization, ASAN/MSAN/UBSAN и malformed/truncated reports, а replay старых occurrences — совместимость обновления.

Отдельный security review обязателен для режимов, требующих GDB/core analysis. Документация CASR указывает, что Docker-сценарий с GDB использует `SYS_PTRACE` и `seccomp=unconfined` ([CASR usage](https://github.com/ispras/casr#usage)). Такой workload не получает автоматического исключения из `ExecutionPolicy`: он запускается в отдельном isolation profile после оценки threat model либо соответствующая capability отклоняется.

Dedup PoC обязан показать объяснимые merge/split decisions, versioned rule, отсутствие переписывания occurrence/report history и переход совпавшего нового occurrence из `FIXED` Finding в `REOPENED` с audit event.

## 13. Coverage и telemetry

`CoverageAnalyzer` нормализует backend outputs в стабильное множество features/edges внутри coverage epoch. OSS-Fuzz поддерживает отдельные coverage builds и команду coverage в официальном guide ([OSS-Fuzz local testing](https://google.github.io/oss-fuzz/getting-started/new-project-guide/#testing-locally)); FuzzBench рассматривается только как reference для analysis workflows ([репозиторий FuzzBench](https://github.com/google/fuzzbench)).

Coverage record хранит `artifactDigest` для provenance и отдельный `coverageCompatibilityKey` для semantic feature space, определённого target, instrumentation, feature schema и normalizer contract. Разные `BuildArtifact` могут разделять key и продолжать одну epoch. Только несовместимый key начинает новую epoch и сбрасывает её baseline и окно стагнации; одна лишь смена `artifactDigest` окно `StopPolicy.noCoverageGrowthFor` не сбрасывает. Окно оценивается отдельно для каждого `FuzzJob` только по подтверждённому active time в `RUNNING`. Queue, build, pause, preemption, checkpoint, recovery и admission wait исключаются.

Telemetry ingestion обязана дедуплицировать intervals по attempt и sequence, отклонять stale epoch и не считать один interval дважды. Normalized telemetry влияет на domain policies; infrastructure telemetry только объясняет состояние до применения явного reconciliation rule.

Metrics labels имеют ограниченную cardinality. Raw input, stack trace, artifact digest, Pod UID и пользовательский текст сохраняются в logs/traces с tenant access и retention, а не в labels. Security-sensitive operation блокируется, если обязательный audit event нельзя надёжно сохранить.

PoC coverage epoch совместимости обязан проверить два разных `artifactDigest` с одинаковым `coverageCompatibilityKey` без сброса stop window, разные artifacts с несовместимыми keys с новой epoch и сбросом окна, а также неизменный artifact с изменившимся несовместимым key. Дополнительно проверяются duplicate/lost intervals, pause/resume и replacement attempt.

## 14. TaskExecutor

`TaskExecutor` маршрутизирует build, reproduce, minimize, normalization, coverage, corpus merge/prune, regression analysis и fix verification через общий execution contract. Kubernetes Job adapter имеет статус `PROPOSED`, но Job status не заменяет `Task` lifecycle.

Для каждой `Task` создаётся idempotent Job envelope. Каждый физический Pod UID отображается на отдельный `ExecutionAttempt`. Рекомендуемый baseline — `restartPolicy: Never`, `backoffLimit: 0` и domain-managed retries. Если policy разрешает replacement Pods, adapter обязан удерживать replacement до перехода `Task` через `QUEUED`, выдачи нового lease и регистрации нового attempt; простого обнаружения уже запущенного Pod недостаточно.

`podFailurePolicy` может классифицировать exit codes и Pod conditions, однако `Ignore`, `Count` или `FailJob` не определяют domain transition напрямую. Adapter переводит observation в transient/permanent/stale/policy result; application logic применяет закрытую таблицу:

- recoverable `FAILED` или `LOST` attempt: `Task RUNNING → QUEUED`, новый lease, новый attempt;
- pause/preemption/admission loss: attempt `CANCELLED`, `Task RUNNING → QUEUED` только если retry разрешён и pause intent больше не блокирует admission;
- permanent admission или policy rejection до запуска: `Task QUEUED → FAILED`, terminal event и запрет дальнейших retries;
- owner cancellation: `Task → CANCELLED`, дальнейшие retries запрещены;
- исчерпание finite retry policy: `Task → FAILED`.

Operation key связывает command, Task, lease и expected attempt. Typed outputs публикуются идемпотентно и принимаются только от текущего допустимого attempt. PoC включает duplicate create, backend replacement, `backoffLimit`, `podFailurePolicy`, timeout, active deadline, cancellation, late output и повторную доставку terminal event.

## 15. Identity и authorization

Кандидат `IdentityProvider` — OIDC adapter со статусом `PROPOSED`; protocol contract определяется [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html). Adapter подтверждает issuer, subject, audience, signature, expiry и memberships/claims, но не принимает domain authorization decisions.

Authorization остаётся application logic Control Plane:

- каждый `Project` принадлежит одному `Tenant`;
- `Membership` связывает identity с Team и versioned role;
- Team получает явную роль на Project, а membership в Tenant не даёт автоматический доступ ко всем Projects;
- command, event, storage key, lease и backend operation несут проверенный tenant context;
- изменение роли, delegation и security-sensitive action создают immutable audit event через `AuditStore`; сбой append оставляет authoritative state неизменным.

PoC проверяет key rotation, expired/revoked token, issuer/audience confusion, stale membership, concurrent role update, cross-tenant identifiers и повтор idempotency key другим principal. Service identities получают минимальные short-lived credentials; workload не получает Control Plane credentials.

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

## 17. External integrations

Внешняя интеграция реализует `FindingSink` и не получает authoritative ownership. Outbound record содержит stable operation key, tenant/project scope, Finding version, redacted payload и ссылку на audit decision.

Adapter обязан:

- идемпотентно создавать или обновлять внешнюю запись;
- хранить opaque external reference отдельно от Finding identity;
- повторять transient errors с finite backoff и сохранять permanent rejection;
- не передавать raw crash input, secrets или embargoed details без явной policy;
- периодически reconciliate drift, не импортируя внешний status как безусловный domain transition;
- использовать provider-specific short-lived credentials.

[ClusterFuzz](https://github.com/google/clusterfuzz) служит reference для issue-tracker workflows, но его integration model не принимается без проверки. Для каждого provider нужны contract tests duplicate delivery, rate limits, deleted/renamed external issue, credential rotation и disclosure policy.

## 18. Компоненты, не используемые как ядро платформы

Следующие проекты полезны как reference или доноры ограниченных adapters, но не заменяют Control Plane:

| Проект | Допустимое использование | Почему не ядро платформы | Статус роли |
|---|---|---|---|
| [ClusterFuzz](https://github.com/google/clusterfuzz) | Reference для triage, reproduce, minimize, regression и issue workflows | Несёт собственные domain и operational assumptions | `PROPOSED` |
| [ClusterFuzzLite](https://github.com/google/clusterfuzzlite) | Reference для CI workflows, periodic coverage и corpus operations | CI-oriented lifecycle не является lifecycle платформы | `PROPOSED` |
| [FuzzBench](https://github.com/google/fuzzbench) | Reference для coverage analysis и benchmarking | Benchmark service не реализует authoritative campaign control | `PROPOSED` |
| OSS-Fuzz `infra/helper.py` | Compatibility fixture и источник golden behavior | Не считается стабильным SDK платформы | `PROPOSED` |
| controller-runtime | Watches/cache/leader election внутри Kubernetes adapter | Не является domain reconciliation framework | `PROPOSED` |

Статус здесь относится к ограниченной роли. Импорт целого проекта как ядра без нового анализа считается `REJECTED`, потому что передал бы domain lifecycle внешнему компоненту.

## 19. MVP и этапы развития

### Первый этап: MVP

- Собственные domain entities, закрытые state machines, commands, idempotency и audit.
- Repositories и `EventPublisher` с открытым выбором persistence technology и PoC atomicity/outbox.
- `AuditStore` с fail-closed append, integrity, retention и tenant-scoped access PoC.
- Kubernetes `ExecutionBackend` для long-running Pod и `TaskExecutor` для Job с явным attempt mapping.
- Базовый `ResourceAdmission`; Kueue включается только после прохождения целевого PoC.
- Единственная build chain `Build → build Task → ExecutionAttempt`, OSS-Fuzz-compatible runner и immutable `BuildArtifact`.
- Собственные `CorpusStore`/`ArtifactStore`, private snapshots и CAS canonical publish.
- `FindingOccurrence → CrashReport → Finding`, начальная CASR integration за boundary.
- Coverage epoch и `noCoverageGrowthFor` по active time.
- Минимальные trust-boundary controls класса `STANDARD`, tenant authorization и отсутствие Control Plane credentials в workload.
- Metrics, structured logs, traces и обязательные audit events.

### Второй этап: устойчивость и расширение

- Доказанные fair sharing, tenant quotas и preemption semantics Kueue.
- Reproduce, minimize, regression, fix verification и re-normalization tasks.
- Advanced isolation profiles `STRONG` и `DEDICATED` после security/performance PoC gVisor/Kata.
- Несколько storage providers после parity suite.
- FindingSink adapters и disclosure workflows.
- Automated corpus merge/prune policies, rollback и retention.

### Третий этап: оптимизация

- Multi-cluster backends и placement policies без изменения domain identity.
- Оптимизация cache и optional explicitly-authorized sharing после leakage analysis.
- Расширенные coverage analytics и сравнительные исследования engines.
- Capacity forecasting и cost policies, не влияющие на correctness contracts.

Переход между этапами определяется evidence и ADR, а не календарём. Advanced feature не может ослаблять обязательный MVP invariant.

## 20. Матрица соответствия

### Порты

| Порт | Ссылка на архитектуру | Кандидат | Статус | Свидетельства проверки | Известные пробелы |
|---|---|---|---|---|---|
| Repositories | §17, invariant 6 | Собственные persistence adapters | `PROPOSED` | Результатов PoC нет | Product/schema открыты; atomicity не доказана |
| `EventPublisher` | §8, §17, invariants 6–7 | Transactional outbox relay | `PROPOSED` | Критерии crash-injection определены; versioned scope и результатов нет | Transport/order/dedup store открыты |
| `BuildExecutor` | §9, §17, invariant 12 | OSS-Fuzz + BuildKit adapter | `PROPOSED` | Критерии fingerprint/provenance определены; versioned scope и результатов нет | Cache isolation, input/output manifests и cancellation не доказаны |
| `ExecutionBackend` | §10, §17, invariants 4–5 | Kubernetes adapter | `PROPOSED` | Attempt mapping scenario определён; versioned scope и результатов нет | Duplicate/replacement behavior не проверен |
| `ResourceAdmission` | §11, §17, invariant 9 | Kueue adapter | `PROPOSED` | Long-running/preemption scenario определён; versioned scope и результатов нет | Lease expiry и quota mapping не доказаны |
| `CorpusStore` | §12, §17, invariants 10–11, 13 | Собственный interface; Go CDK candidate | `PROPOSED` | Результатов PoC нет | Provider CAS/parity открыты |
| `ArtifactStore` | §9, §17, invariant 10 | Собственный interface; Go CDK candidate | `PROPOSED` | Результатов PoC нет | Atomic visibility/large objects не проверены |
| `CoverageAnalyzer` | §13, §17, invariants 14–15 | Собственный normalizer | `PROPOSED` | Совместимые/несовместимые artifact scenarios определены; versioned scope и результатов нет | Feature stability и interval dedup не доказаны |
| `FindingNormalizer` | §14, §17, invariant 16 | CASR adapter | `PROPOSED` | Golden/security criteria определены; versioned scope и результатов нет | Format drift и privileged modes не проверены |
| `FindingDeduplicator` | §14, §17, invariant 16 | Versioned platform policy + CASR signals | `PROPOSED` | Replay criteria определены; versioned scope и результатов нет | Merge/split accuracy не измерена |
| `TaskExecutor` | §7, §10, §17, invariant 4 | Kubernetes Job adapter | `PROPOSED` | Retry/rejection scenarios определены; versioned scope и результатов нет | Hidden backend retries не исключены |
| `IdentityProvider` | §4, §15, §17, invariant 2 | OIDC adapter | `PROPOSED` | Результатов PoC нет | Provider, claims и revocation открыты |
| `FindingSink` | §17 | Provider-specific adapters | `PROPOSED` | Результатов PoC нет | Providers и disclosure contracts открыты |
| `AuditStore` | §15–§17 | Собственный append-only adapter | `PROPOSED` | Fault-injection, retention и access criteria определены; versioned scope и результатов нет | Persistence/WORM technology и atomic boundary открыты |

### Инварианты

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
| 12. Consumers используют artifact; build attempt — inputs | §9, §19.12 | Раздельные consuming/build manifests; output digest только в terminal result | `PROPOSED` | Build provenance/input-output criteria определены; versioned scope и результатов нет | Runner compatibility не доказана |
| 13. Resume только с опубликованного compatible snapshot | §12, §19.13 | `corpusCompatibilityKey` и manifest readiness | `PROPOSED` | Resume scenarios определены; versioned scope и результатов нет | Conversion task не спроектирована |
| 14. `noCoverageGrowthFor` считает active time на job | §13, §19.14 | Coverage ledger per `FuzzJob` | `PROPOSED` | Time-window scenario определён; versioned scope и результатов нет | Lost telemetry policy не проверена |
| 15. Compatibility отделена от artifact provenance | §13, §19.15 | `artifactDigest` + versioned `coverageCompatibilityKey` | `PROPOSED` | Compatible/incompatible artifact matrix определена; versioned scope и результатов нет | Feature normalizer открыт |
| 16. Finding history immutable/versioned | §14, §19.16 | Occurrence store, versioned reports/rules | `PROPOSED` | CASR replay criteria определены; versioned scope и результатов нет | Dedup quality не измерена |
| 17. Потеря attempt не означает `Campaign FAILED` | §7, §13, §19.17 | Campaign aggregator application logic | `PROPOSED` | Результатов failure tests нет | Recovery thresholds открыты |
| 18. Stop после checkpoint deadline обязателен | §7, §11, §19.18 | Timed checkpoint then backend stop | `PROPOSED` | Preemption/StopPolicy scenarios определены; versioned scope и результатов нет | Backend timing не измерен |
| 19. Workload не получает Control Plane credentials | §15, §19.19 | Scoped workload identity и deny policy | `PROPOSED` | Результатов security tests нет | Runtime/isolation candidate открыт |

## 21. Реестр PoC и открытых технологических решений

Записи ниже являются только предлагаемыми критериями. Для перевода любой строки в `VALIDATING` нужен отдельный committed versioned scope с pins, environment, ответственным запуском и точным evidence location; таких файлов в репозитории нет.

| Идентификатор | Проверка | Кандидаты | Статус | Критерий завершения | Свидетельства |
|---|---|---|---|---|---|
| `POC-PERSIST-001` | Persistence atomicity/outbox | Persistence adapter + outbox relay | `PROPOSED` | Crash matrix не теряет committed event/state, duplicates идемпотентны, aggregate order сохранён | Versioned scope, pins и results path отсутствуют |
| `POC-AUDIT-001` | Atomic/fail-closed audit recording | `AuditStore` + persistence adapter | `PROPOSED` | Fault injection не оставляет security-sensitive state без immutable record; tampering обнаруживается; retention и tenant-scoped access соблюдаются | Versioned scope, storage candidate, pins и results path отсутствуют |
| `POC-K8S-ATTEMPT-001` | Kubernetes attempt mapping | Pod/Job adapter | `PROPOSED` | Каждый Pod UID имеет один attempt; duplicate start, replacement, watch gap и late output корректны | Versioned scope, pins и results path отсутствуют |
| `POC-KUEUE-001` | Kueue long-running/preemption | Kueue + Plain Pods | `PROPOSED` | Admission, deletion, checkpoint deadline, new lease/attempt, fair sharing и tenant quotas подтверждены | Versioned scope, pins и results path отсутствуют |
| `POC-RUNNER-001` | OSS-Fuzz runner compatibility | Versioned runner + pinned OSS-Fuzz | `PROPOSED` | Golden projects проходят build/run/reproduce/coverage/cancel на pinned digests | Versioned scope, pins и results path отсутствуют |
| `POC-BUILD-001` | Build reproducibility и isolation | OSS-Fuzz + BuildKit | `PROPOSED` | Build attempt содержит immutable inputs без будущего output digest; fingerprint, provenance, cache isolation, restore, cancellation и timeout подтверждены | Versioned scope, pins и results path отсутствуют |
| `POC-CORPUS-001` | Immutable corpus publish | `CorpusStore` + выбранный provider | `PROPOSED` | Partial snapshot невидим; checksum обязателен; CAS conflict запускает recompute; rollback сохранён | Versioned scope, provider, pins и results path отсутствуют |
| `POC-CASR-001` | CASR security boundary | CASR parser/GDB profiles | `PROPOSED` | Golden normalization проходит; privileged paths изолированы либо отклонены; threat model reviewed | Versioned scope, pins и results path отсутствуют |
| `POC-COVERAGE-001` | Coverage epoch compatibility | `CoverageAnalyzer` | `PROPOSED` | Разные artifacts с одинаковым `coverageCompatibilityKey` не сбрасывают stop window; несовместимый key создаёт epoch и сбрасывает окно; intervals не удваиваются | Versioned scope, fixtures и results path отсутствуют |
| `POC-BLOB-001` | Storage provider parity | Go CDK `blob` и native SDK alternatives | `PROPOSED` | Conditional writes, checksums, streaming, errors и authorization одинаково удовлетворяют contracts | Кандидаты provider, versioned scope и results path отсутствуют |
| `POC-ISOLATION-001` | Advanced isolation | gVisor/Kata | `PROPOSED` | Compatibility, escape surface, performance, checkpoint и operations измерены для `STRONG`/`DEDICATED` | Этап после MVP; versioned scope и results path отсутствуют |

Открытыми остаются persistence product, audit storage/WORM mechanism, event transport, blob providers, OIDC provider, exact Kubernetes/Kueue versions, sandbox mapping и FindingSink providers. Любое закрытие решения требует результатов соответствующего PoC, license/security review и ADR; изменение архитектурного контракта вместо этого требует правки `Architecture.md`.

## 22. Репозитории и первичные источники

Все ссылки ведут непосредственно на официальную документацию или репозиторий проекта:

- [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime) — Kubernetes controller libraries, используемые только внутри adapter boundary.
- [Kueue Concepts](https://kueue.sigs.k8s.io/docs/concepts/) — queues, workloads, admission, fair sharing и preemption.
- [Kueue: Run Plain Pods](https://kueue.sigs.k8s.io/docs/tasks/run/plain_pods/) — scheduling gates, Pod admission и deletion semantics.
- [Kubernetes Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/) — retries, `backoffLimit`, `podFailurePolicy` и replacement Pods.
- [OSS-Fuzz: Setting up a new project](https://google.github.io/oss-fuzz/getting-started/new-project-guide/) — build/run/coverage conventions.
- [OSS-Fuzz `infra/helper.py`](https://github.com/google/oss-fuzz/blob/master/infra/helper.py) — reference command implementation, не platform SDK.
- [BuildKit](https://github.com/moby/buildkit) — build engine candidate.
- [Go CDK](https://github.com/google/go-cloud) — `blob` portability candidate.
- [CASR](https://github.com/ispras/casr) — crash parsing, reports и triage candidate.
- [ClusterFuzz](https://github.com/google/clusterfuzz) — donor/reference project.
- [ClusterFuzzLite](https://github.com/google/clusterfuzzlite) — CI workflow reference.
- [FuzzBench](https://github.com/google/fuzzbench) — benchmarking и coverage analysis reference.
- [gVisor Kubernetes Quick Start](https://gvisor.dev/docs/user_guide/quick_start/kubernetes/) — Kubernetes `RuntimeClass` integration candidate.
- [Kata Containers](https://github.com/kata-containers/kata-containers) — lightweight VM isolation candidate.
- [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html) — authentication protocol reference.
