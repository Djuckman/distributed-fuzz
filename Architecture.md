# Архитектура распределённой fuzzing-платформы

## 1. Статус и область действия

Этот документ является нормативным источником domain terminology, lifecycle, consistency guarantees, trust boundaries, портов и архитектурных инвариантов платформы. Слова «должен», «обязан» и «запрещён» обозначают обязательные требования.

Направление зависимости документации: `Architecture → Implementation`. `Implementation.md` обязан показывать, как выбранные компоненты удовлетворяют контрактам этого документа, но не может переопределять domain model, lifecycle или инварианты. При расхождении приоритет имеет `Architecture.md`.

Архитектура не зависит от выбора базы данных, оркестратора, execution platform, хранилища, очереди, fuzzing engine или иных сторонних технологий. Конкретные продукты и operational mapping относятся к реализации и ADR.

## 2. Цели и ограничения

### Цели

- Управлять распределёнными fuzzing-кампаниями как долгоживущими domain entities.
- Разделять пользовательский intent, авторитетное состояние и физическое исполнение.
- Обеспечивать tenant isolation, воспроизводимость сборок и трассируемость результатов.
- Восстанавливать полезную работу после потери вычислительных ресурсов без потери domain history.
- Поддерживать finite tasks и непрерывный fuzzing через общий execution contract.
- Формализовать scheduling, admission, preemption, corpus, coverage и findings без привязки к технологии.

### Non-goals

- Архитектура не требует микросервисного разбиения; один процесс и несколько процессов допустимы при соблюдении границ модулей.
- Архитектура не выбирает конкретную базу данных, конкретный cluster backend, оркестратор, execution platform или storage.
- Архитектура не предписывает конкретный sandbox, алгоритм fair sharing, формат transport или способ доставки событий.
- Архитектура не гарантирует exactly once delivery и не делает физический ресурс частью domain identity.
- Архитектура не стандартизует внутренний протокол конкретного fuzzing engine.

## 3. Термины и принципы

**Control Plane** — логическая часть системы, которая принимает команды, хранит authoritative domain state и выполняет reconciliation.

**Execution Backend** — адаптер и инфраструктура, которые исполняют запрошенный workload и сообщают наблюдаемое состояние, но не определяют domain lifecycle.

**Desired state** — сохранённый пользовательский или policy intent. **Observed state** — подтверждённое системой фактическое состояние. **Authoritative state** — единственная нормативная запись domain facts в Control Plane.

**Workload** — обобщённая заявка на исполнение `FuzzJob` или конечной `Task`. **Terminal phase** — фаза, из которой нет перехода для той же сущности; повтор исполнения создаёт новую попытку, а не меняет историю terminal-сущности.

Архитектура следует принципам:

1. domain identity не зависит от identity инфраструктурного ресурса;
2. intent сохраняется до вызова внешнего адаптера;
3. внешние вызовы и обработка событий идемпотентны;
4. immutable facts и artifacts не переписываются;
5. tenant context обязателен на каждой границе;
6. восстановление выполняется reconciliation, а не изменением истории;
7. длительные операции асинхронны относительно API.

## 4. Trust boundaries и multi-tenancy

Платформа рассматривает tenants как взаимно недоверенные стороны, а пользовательский код — как недоверенный workload. Основные trust boundaries проходят между клиентом и Control Plane, Control Plane и adapters, workload и Control Plane, workload и storage, а также между workloads разных tenants.

`Tenant` является верхней границей владения и изоляции. `Team` группирует субъектов внутри одного Tenant. `Membership` связывает identity с Team и ролью. Team получает явно заданную роль для Project; принадлежность к Tenant сама по себе не даёт доступ ко всем Project.

Каждый `Project` принадлежит ровно одному `Tenant`. Межtenant-доступ запрещён, включая чтение metadata, artifacts, corpus, findings, telemetry и audit records. Любая команда, событие, ключ хранения, lease и backend operation обязаны нести проверенный tenant context. Делегирование доступа возможно только внутри Tenant и фиксируется audit event.

Control Plane владеет domain entities. Execution Backend владеет лишь своими физическими ресурсами и их telemetry. Artifact и corpus adapters хранят bytes и manifests, но их domain ownership и права доступа определяет Control Plane.

## 5. Domain model

| Сущность | Назначение | Domain owner |
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
| `BuildArtifact` | Immutable content-addressed результат Build | `Build` |
| `Task` | Конечная работа: build, reproduce, minimize, normalization, coverage, corpus merge/prune, regression analysis или fix verification | Сущность, которая запросила работу; ссылка обязательна |
| `ExecutionAttempt` | Одна историческая попытка физического выполнения FuzzJob или Task | Соответствующий `FuzzJob` или `Task` |
| `ResourceLease` | Ограниченное по времени admission-разрешение на ресурсы | Resource Admission от имени `Tenant` |
| `Corpus` | Именованная логическая линия входных данных | `Project` и `FuzzTarget` |
| `CorpusSnapshot` | Immutable manifest конкретного состояния Corpus | `Corpus` |
| `FindingOccurrence` | Immutable-факт обнаружения с raw artifacts | `FuzzJob` и породивший `ExecutionAttempt` |
| `CrashReport` | Версионированный результат нормализации occurrence | `FindingOccurrence` |
| `Finding` | Дедуплицированная пользовательская проблема и её triage state | `Project` |

Обязательные отношения владения и происхождения:

```text
Tenant → Project → Campaign → FuzzJob → ExecutionAttempt
Task → ExecutionAttempt
FindingOccurrence → CrashReport → Finding
```

`Campaign` фиксирует выбранные targets, `SourceRevision`, build parameters, resource policy и stop policy. `FuzzJob` не привязан к worker или размещению. `ExecutionAttempt` является попыткой выполнения, а не зеркалом backend resource. Физический идентификатор хранится только как opaque `BackendResourceRef`.

`BuildRecipe` — immutable value object со всеми значимыми параметрами сборки и ссылкой на версию build environment. Он не является самостоятельно управляемым aggregate.

## 6. Авторитетное, desired и observed state

Единственный authoritative domain state принадлежит Control Plane. Execution Backend сообщает observations и может потерять или пересоздать физический ресурс, но не становится source of truth для `Campaign`, `FuzzJob`, `Task` или `ExecutionAttempt`.

Управляемые aggregates обязаны содержать:

- монотонный `generation`, увеличиваемый при изменении desired state;
- `observedGeneration`, до которого reconciler подтвердил обработку;
- version token для optimistic concurrency;
- structured conditions с `reason`, `message` и `lastTransitionAt`;
- tenant и owner references, проверяемые при каждом изменении.

Запись observed state принимается только при совпадении version token и применимой `generation`. Устаревшее наблюдение не откатывает более новое состояние. Конфликт optimistic concurrency заставляет обработчик перечитать aggregate и повторно оценить intent.

Изменение desired state и запись соответствующего durable domain event выполняются атомарно. Если атомарная операция не состоялась, ни новое состояние, ни событие не считаются опубликованными. Способ обеспечения атомарности является деталью реализации.

## 7. Lifecycle и state machines

Desired state `Campaign` принимает только `RUNNING`, `PAUSED` или `STOPPED`. Observed phase отдельно отражает прогресс применения intent.

### Campaign

| Current phase | Trigger | Next phase | Обязательный side effect |
|---|---|---|---|
| `CREATED` | desired=`RUNNING`, команда принята | `BUILDING` | Зафиксировать требуемые Build/Task и событие reconciliation |
| `BUILDING` | обязательные artifacts готовы | `QUEUED` | Создать или активировать FuzzJob и admission-запросы |
| `BUILDING` | часть recoverable build work неуспешна | `DEGRADED` | Сохранить conditions и запланировать допустимые retries |
| `QUEUED` | обязательный FuzzJob начал полезное исполнение | `RUNNING` | Обновить `observedGeneration` и время активного выполнения |
| `RUNNING` или `DEGRADED` | desired=`PAUSED` | `PAUSING` | Запросить checkpoint всех активных workloads с deadline |
| `PAUSING` | workloads остановлены или deadline истёк | `PAUSED` | Освободить leases и сохранить ссылки на последние опубликованные snapshots |
| `PAUSED` | desired=`RUNNING` | `QUEUED` | Запросить новые leases и указать последние успешные snapshots |
| Любая non-terminal | desired=`STOPPED` | `STOPPING` | Отменить pending work и остановить активные workloads |
| `STOPPING` | активных workloads больше нет | `STOPPED` | Освободить leases и опубликовать terminal event |
| `RUNNING` | все обязательные jobs успешно достигли terminal condition | `COMPLETED` | Зафиксировать итог, snapshots и terminal event |
| `RUNNING` или `DEGRADED` | полезное исполнение было, но есть невосстановимые ошибки обязательных jobs | `COMPLETED_WITH_ERRORS` | Зафиксировать успешные результаты и перечень ошибок |
| `BUILDING`, `QUEUED` или `DEGRADED` | полезное исполнение невозможно и recovery исчерпан | `FAILED` | Зафиксировать причины и отменить оставшуюся работу |
| `RUNNING` | временная потеря части capacity или попытки | `DEGRADED` | Начать recovery, не теряя intent Campaign |
| `DEGRADED` | требуемая capacity восстановлена | `RUNNING` | Очистить временную condition после подтверждения |

### FuzzJob

| Current phase | Trigger | Next phase | Обязательный side effect |
|---|---|---|---|
| `CREATED` | требуемый Build ещё не готов | `WAITING_FOR_BUILD` | Связать job с точным Build |
| `CREATED` или `WAITING_FOR_BUILD` | BuildArtifact готов | `QUEUED` | Создать admission-запрос |
| `QUEUED` | ResourceLease выдан | `STARTING` | Создать новый ExecutionAttempt |
| `STARTING` | попытка сообщает начало полезного fuzzing | `RUNNING` | Начать учёт active time текущей coverage epoch |
| `RUNNING` | попытка потеряна, intent остаётся `RUNNING` | `RECOVERING` | Завершить попытку как `LOST`, освободить lease и запросить replacement |
| `RECOVERING` | новый lease выдан | `STARTING` | Создать новый ExecutionAttempt с последним snapshot |
| `RUNNING` | Campaign переходит к pause | `PAUSING` | Запросить checkpoint до общего deadline |
| `PAUSING` | workload остановлен или deadline истёк | `PAUSED` | Исключить период pause из active time и освободить lease |
| `PAUSED` | Campaign возобновлена | `QUEUED` | Указать последний успешно опубликованный snapshot |
| `RUNNING` | terminal condition StopPolicy выполнен | `COMPLETED` | Опубликовать финальный snapshot и итог coverage epoch |
| Любая non-terminal | остановка или отмена Campaign | `CANCELLED` | Остановить попытку и освободить lease |
| Любая non-terminal | невосстановимая ошибка или исчерпан recovery policy | `FAILED` | Зафиксировать condition и сообщить агрегатору Campaign |

### ExecutionAttempt

| Current phase | Trigger | Next phase | Обязательный side effect |
|---|---|---|---|
| `PENDING` | lease и входы подтверждены | `STARTING` | Идемпотентно запросить запуск у Execution Backend |
| `STARTING` | backend подтвердил готовность workload | `RUNNING` | Сохранить opaque `BackendResourceRef` и start time |
| `PENDING` или `STARTING` | запуск отменён до готовности | `CANCELLED` | Освободить lease и записать terminal event |
| `RUNNING` | workload штатно завершил конечную работу | `SUCCEEDED` | Сохранить outputs и освободить lease |
| `RUNNING` | workload сообщил ошибку | `FAILED` | Сохранить diagnostics и освободить lease |
| `STARTING` или `RUNNING` | backend resource потерян или observation протухло | `LOST` | Зафиксировать последний snapshot reference и освободить lease |
| `RUNNING` | stop, pause или preemption требует остановки | `STOPPING` | Запросить best-effort checkpoint с deadline |
| `STOPPING` | остановка после успешного checkpoint | `SUCCEEDED` | Опубликовать snapshot до terminal event и освободить lease |
| `STOPPING` | workload остановлен без требуемого результата | `CANCELLED` | Сохранить причину и последний ранее опубликованный snapshot |

`SUCCEEDED`, `FAILED`, `LOST` и `CANCELLED` — terminal phases `ExecutionAttempt`. Retry всегда создаёт новый `ExecutionAttempt`.

### Build

| Current phase | Trigger | Next phase | Обязательный side effect |
|---|---|---|---|
| `PENDING` | входы разрешены и Task допущена | `RUNNING` | Создать ExecutionAttempt build-задачи |
| `RUNNING` | artifact опубликован и digest проверен | `SUCCEEDED` | Атомарно связать Build с BuildArtifact и provenance |
| `PENDING` или `RUNNING` | отмена владельцем | `CANCELLED` | Остановить активную попытку и сохранить причину |
| `RUNNING` | невосстановимая ошибка или retries исчерпаны | `FAILED` | Сохранить diagnostics и оповестить зависимые jobs |

### Task

| Current phase | Trigger | Next phase | Обязательный side effect |
|---|---|---|---|
| `PENDING` | зависимости готовы | `QUEUED` | Создать admission-запрос с tenant context |
| `QUEUED` | ResourceLease выдан | `RUNNING` | Создать новый ExecutionAttempt |
| `RUNNING` | обязательные outputs опубликованы | `SUCCEEDED` | Сохранить output references и освободить lease |
| `RUNNING` | попытка потеряна, retry разрешён | `QUEUED` | Завершить попытку как `LOST` и запросить новый lease |
| `PENDING`, `QUEUED` или `RUNNING` | отмена владельцем | `CANCELLED` | Прекратить дальнейшие retries и освободить lease |
| `RUNNING` | невосстановимая ошибка или retries исчерпаны | `FAILED` | Сохранить diagnostics и уведомить владельца Task |

Pause и preemption используют checkpoint best effort до заданного deadline. После deadline workload останавливается независимо от результата checkpoint. Resume использует последний успешно опубликованный `CorpusSnapshot`, а не локальное неподтверждённое состояние остановленной попытки.

## 8. Команды, события и reconciliation

Поток изменения состояния:

```text
API command
  → validate + authorize + idempotency key
  → atomically update desired state and durable domain event
  → reconciler or policy
  → idempotent port call
  → observed state update
```

API подтверждает сохранение команды, но не ждёт завершения долгих операций. Повтор команды с тем же idempotency key и тем же payload возвращает тот же логический результат; повтор ключа с иным payload отклоняется.

Доставка domain events имеет семантику at least once. Consumers и adapters обязаны быть идемпотентными и хранить достаточный deduplication state. Порядок событий гарантируется только в границах одного aggregate и явно выбранной последовательности; consumer обязан отклонять stale update по `generation`, `observedGeneration` и version.

Reconciler вычисляет действия из authoritative desired state и свежего observed state. Он не предполагает, что предыдущий вызов адаптера завершился, и безопасно повторяет запрос с устойчивым operation key. Exactly once не является архитектурной гарантией. Конкретный transport, polling и способ атомарной публикации являются деталями реализации.

## 9. Build и артефакты

Нормативная модель:

```text
SourceRevision + BuildRecipe → Build → BuildArtifact
```

`SourceRevision` разрешается в immutable identity до начала Build. Fingerprint Build включает revision, все значимые поля `BuildRecipe`, target, архитектуру выполнения и совместимость instrumentation. Одинаковый fingerprint может переиспользовать только artifact, доступный тому же Tenant и прошедший integrity check.

`BuildArtifact` является immutable и content-addressed: его identity включает cryptographic digest содержимого. Provenance связывает artifact с Tenant, Project, SourceRevision, BuildRecipe, Build, создавшей Task/ExecutionAttempt и временем публикации. `FuzzJob`, reproduce и analysis tasks ссылаются на точный artifact digest.

Artifacts и build cache разных tenants по умолчанию изолированы. Любое ослабление изоляции требует отдельного явно разрешённого policy с проверяемым отсутствием утечки; базовый контракт не требует такого совместного использования.

Публикация считается успешной только после проверки digest и доступности manifest. Частично записанный artifact не становится видимым через domain state.

## 10. Execution model

`FuzzJob` представляет непрерывный workload, а `Task` — конечный workload. Оба используют `ExecutionAttempt`, но сохраняют независимые lifecycle и retry policies.

Одна попытка соответствует одному созданию физического backend resource. Внутренний restart процесса в рамках того же ресурса остаётся той же попыткой; создание нового ресурса требует новой попытки. Потеря ресурса завершает текущую попытку как `LOST`, но не удаляет её историю и не завершает автоматически владеющий FuzzJob или Task.

Retry всегда создаёт новый `ExecutionAttempt` с новым identity и ссылкой на предыдущую попытку как причину retry. Одновременно активные попытки одного workload допустимы только при явной policy и уникальных ownership scopes для outputs.

`BackendResourceRef` является opaque значением адаптера. Его структура, namespace и lifecycle не проникают в domain semantics, пользовательские identifiers или правила переходов. Control Plane сопоставляет observations с попыткой и игнорирует события для неизвестной или terminal попытки, кроме audit.

Execution Backend обязан поддерживать идемпотентные операции start, observe, checkpoint и stop, а также объявлять capabilities. Отсутствующая capability проверяется до admission; адаптер не имитирует неподдерживаемую семантику.

## 11. Scheduling, admission и preemption

Scheduling состоит из трёх независимых уровней:

```text
Domain Scheduler → Resource Admission → Execution Backend
```

`Domain Scheduler` выбирает, какой workload имеет право претендовать на выполнение, исходя из lifecycle, зависимостей и policy. `Resource Admission` применяет quotas, priority, fair sharing и preemption. `Execution Backend` выбирает физическое размещение в рамках выданного разрешения.

`ResourceLease` обязан включать:

- ссылку на workload (`FuzzJob` или `Task`);
- `tenantId` и при необходимости `projectId`;
- нормализованный resource request;
- priority и policy decision reference;
- `issuedAt`, `expiresAt` и уникальный lease identity.

Запуск без действующего lease запрещён. Lease имеет expiry, продлевается явно и освобождается при terminal attempt, отмене или подтверждённой потере. Истечение lease прекращает право на ресурсы и запускает reconciliation; оно не стирает attempt history.

Preemption является поддерживаемым архитектурным контрактом, даже если policy конкретной реализации отключена. При preemption Resource Admission отзывает lease с deadline, Control Plane запрашивает checkpoint best effort, затем Execution Backend останавливает workload независимо от результата checkpoint. Освобождённый FuzzJob возобновляется с последнего успешно опубликованного snapshot.

## 12. Corpus и snapshots

`Corpus` — именованная логическая линия для пары Project/FuzzTarget и заданного compatibility scope. Он указывает на canonical snapshot, но не является общей mutable директорией.

`CorpusSnapshot` — immutable manifest, который содержит:

- собственный digest и checksums каждого объекта;
- optional parent snapshot;
- compatibility key для target, artifact и instrumentation;
- Tenant, Project, Corpus и creator ExecutionAttempt;
- список content-addressed объектов и metadata публикации.

Каждая Campaign публикует private campaign snapshots. Несколько Campaign не изменяют один общий mutable corpus. Snapshot становится доступен для resume только после полной публикации manifest и проверки checksums.

Policy автоматически создаёт `CorpusMergeTask` по lifecycle event, интервалу или порогу накопленных изменений. Ручной on-demand trigger создаёт ту же Task и проходит те же authorization, admission и audit rules. Merge/prune читает immutable inputs, публикует новый snapshot и атомарно переводит canonical reference; предыдущий canonical snapshot сохраняется согласно rollback и retention policy.

Несовместимый compatibility key запрещает silent merge или resume. Требуется новая corpus line либо явная преобразующая Task, результат которой имеет новый key и provenance.

## 13. Coverage и условия завершения

Основной stop criterion — `StopPolicy.noCoverageGrowthFor`: отсутствие роста coverage в течение `N` активных часов. Окно считается отдельно для каждого `FuzzJob`, а не на уровне Campaign.

Active time растёт только во время подтверждённого полезного fuzzing в фазе `RUNNING`. Queue, build, pause, preemption, checkpoint, recovery и ожидание admission исключаются. Потерянные или дублированные telemetry intervals не должны дважды увеличивать active time.

Coverage growth определяется монотонным изменением нормализованного множества features/edges внутри одной coverage epoch. При несовместимой instrumentation или несовместимом `BuildArtifact` начинается новая coverage epoch; baseline и окно стагнации новой epoch не смешиваются с предыдущими. История старых epochs сохраняется.

`StopPolicy` также может содержать `maxActiveTime` как safety limit и `deadline` как абсолютный предел. Эти условия не заменяют основной criterion и оцениваются независимо.

Campaign получает:

- `COMPLETED`, когда все обязательные FuzzJob достигли terminal condition и обязательные результаты опубликованы без невосстановимых ошибок;
- `COMPLETED_WITH_ERRORS`, когда было полезное fuzzing-исполнение и сохранены полезные результаты, но часть обязательных jobs завершилась невосстановимой ошибкой;
- `FAILED`, когда Campaign не смогла получить полезное fuzzing-исполнение или обязательные результаты в пределах recovery policy.

Временная потеря отдельной попытки не переводит Campaign в `FAILED`. Необязательные jobs и tasks влияют на structured conditions, но не меняют terminal outcome, если policy явно не делает их обязательными.

## 14. Findings и triage

Нормативный pipeline:

```text
FindingOccurrence → CrashReport → Finding
```

`FindingOccurrence` неизменяемо фиксирует момент обнаружения, Tenant, Project, FuzzJob, ExecutionAttempt, artifact digest, input и raw outputs. Повторная обработка не изменяет occurrence.

`FindingNormalizer` создаёт новый `CrashReport` с `normalizerName`, `normalizerVersion`, normalized stack, classification и ссылками на inputs. Смена версии нормализатора создаёт новый report и сохраняет прежние результаты для объяснимости.

`FindingDeduplicator` вычисляет версионированный deduplication key из CrashReport и project policy. Решение merge/split обязано сохранять rule version и evidence. Finding ведёт triage states, включая `OPEN`, `ACKNOWLEDGED`, `FIXED`, `IGNORED` и `REOPENED`.

Новое совместимое occurrence, совпавшее с `FIXED` Finding, переводит его в `REOPENED` и создаёт audit event. Новая версия нормализатора или deduplication rule не переписывает историю; массовая переоценка выполняется отдельной Task.

## 15. Security requirements

Каждая операция аутентифицируется, авторизуется в tenant/project scope и оставляет audit event для security-sensitive изменений. Workload не получает credentials Control Plane и не может выбирать tenant context, lease, isolation class или backend reference.

`ExecutionPolicy` задаёт запрошенный класс изоляции и ограничения capabilities. Поддерживаются классы:

- `STANDARD` — базовая изоляция недоверенного workload;
- `STRONG` — усиленная изоляция для повышенного риска;
- `DEDICATED` — исключительное размещение в выделенном security scope.

Архитектура не выбирает sandbox implementation. Execution Backend обязан либо доказуемо применить требуемый класс, либо отклонить запуск до исполнения. Понижение класса без новой авторизованной команды запрещено.

Secrets передаются workload только в минимально необходимом scope, не сохраняются в artifacts, logs или corpus и имеют ограниченный срок действия. Все storage operations проверяют tenant ownership; content address сам по себе не даёт права чтения.

## 16. Observability

Платформа обязана предоставлять metrics, structured logs, distributed traces и audit events с согласованными correlation identifiers: `tenantId`, `projectId`, aggregate identity, `taskId`/`fuzzJobId`, `executionAttemptId` и operation identity.

Telemetry разделяется на два класса:

1. **Normalized coverage/fuzzing telemetry**: active time, coverage epoch, coverage growth, executions, findings, corpus и fuzzer health. Эти данные проходят нормализацию и участвуют в domain policies.
2. **Infrastructure telemetry**: capacity, placement, process/resource health, adapter latency и backend failures. Эти данные объясняют исполнение, но сами не меняют domain lifecycle без reconciliation rule.

Metrics labels обязаны иметь ограниченную cardinality. Нельзя использовать raw input, stack trace, artifact digest, backend resource identity или произвольный пользовательский текст как неограниченный label. Высококардинальные данные помещаются в структурированные logs или trace attributes с retention и access policy.

Audit events неизменяемо фиксируют actor, tenant scope, command, target, result, policy decision и timestamp. Observability failure не должна незаметно менять domain outcome; потеря обязательного audit record блокирует security-sensitive операцию согласно policy.

## 17. Порты и границы модулей

Domain modules зависят от портов, а adapters реализуют порты. Domain code не импортирует backend types и не интерпретирует opaque references.

| Порт | Нормативный контракт |
|---|---|
| Repositories (`TenantRepository`, `ProjectRepository`, `CampaignRepository`, `WorkloadRepository`, `FindingRepository`) | Загружать и атомарно сохранять tenant-scoped aggregates с optimistic concurrency; поддерживать изменение desired state вместе с durable event |
| `EventPublisher` | Публиковать сохранённые domain events с семантикой at least once, сохраняя aggregate identity и sequence |
| `BuildExecutor` | Запускать, наблюдать и отменять Build по immutable inputs; возвращать provenance и artifact digest |
| `ExecutionBackend` | Идемпотентно start/observe/checkpoint/stop workload; возвращать capabilities и opaque `BackendResourceRef` |
| `ResourceAdmission` | Выдавать, продлевать, отзывать и освобождать `ResourceLease` по quotas, priority и policy |
| `CorpusStore` | Публиковать и читать immutable snapshots, проверять checksums и атомарно менять canonical reference |
| `ArtifactStore` | Публиковать и читать content-addressed artifacts с integrity и tenant authorization |
| `CoverageAnalyzer` | Нормализовать coverage в совместимой epoch и определять growth без backend-specific semantics |
| `FindingNormalizer` | Создавать версионированный CrashReport из immutable occurrence, не изменяя source fact |
| `FindingDeduplicator` | Принимать версионированное и объяснимое merge/split решение для Finding |
| `TaskExecutor` | Маршрутизировать конечные Task через общий execution contract и возвращать typed output references |
| `IdentityProvider` | Подтверждать identity и memberships; authorization остаётся обязанностью Control Plane |
| `FindingSink` | Идемпотентно экспортировать Finding во внешнюю систему без передачи ей authoritative ownership |

Входные порты API отвечают за validation, authentication, authorization и idempotency key. Policy modules принимают domain facts и возвращают решения без прямых infrastructure calls. Adapter-specific configuration находится за composition boundary.

## 18. Failure semantics

Ошибки классифицируются как transient, permanent, conflict, stale или policy rejection. Retry допускается только для transient failure и всегда ограничивается policy; permanent failure сохраняется как structured condition. Ни одна ошибка адаптера не даёт права переписать terminal history.

| Сбой | Обнаружение | Обязательная реакция | Recovery/результат |
|---|---|---|---|
| Потеря backend resource | Not-found, expired observation или подтверждённая потеря | Завершить текущий ExecutionAttempt как `LOST`, освободить lease, сохранить diagnostics | При сохраняющемся intent запросить новый lease и создать новый ExecutionAttempt с последним snapshot |
| Checkpoint failure | Ошибка публикации или deadline | Записать condition; после deadline остановить workload независимо от результата | Resume использует последний успешно опубликованный snapshot; локальный partial checkpoint игнорируется |
| Duplicate event | Уже обработанный event identity или sequence | Вернуть сохранённый логический результат без повторного side effect | Продолжить обработку последующих событий; факт duplicate доступен в telemetry |
| Stale update | Старая generation, version или terminal attempt | Отклонить изменение authoritative state | Перечитать aggregate; новое observed state принимается только после reconciliation |
| Admission loss | Lease отозван, истёк или не может быть продлён | Прекратить право на исполнение, инициировать checkpoint/stop и освободить запись lease | Workload возвращается в queue или завершается по policy; новый запуск требует нового lease и attempt |
| Недоступность artifact storage | Timeout или failed integrity/readiness check | Не объявлять Build/Task успешной и не публиковать partial reference | Повторять в пределах policy; при исчерпании завершить работу ошибкой, не повреждая прежний artifact |

## 19. Архитектурные инварианты

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
11. FuzzJob и ExecutionAttempt всегда ссылаются на точный BuildArtifact; provenance ведёт к SourceRevision и BuildRecipe.
12. Resume использует только последний успешно опубликованный совместимый CorpusSnapshot.
13. Окно `noCoverageGrowthFor` считается по active time отдельно для каждого FuzzJob и не включает queue, build, pause, preemption или recovery.
14. Несовместимая instrumentation начинает новую coverage epoch и не смешивает baseline с предыдущей epoch.
15. FindingOccurrence неизменяем; CrashReport и deduplication decisions версионированы; повтор исправленной проблемы может создать `REOPENED`.
16. Временная потеря одной попытки не переводит Campaign в `FAILED`; terminal outcome вычисляется по полезному исполнению и обязательным jobs.
17. Pause и preemption останавливают workload после deadline независимо от успеха checkpoint.
18. Ни один workload не получает credentials Control Plane и не может самостоятельно ослабить `ExecutionPolicy`.
