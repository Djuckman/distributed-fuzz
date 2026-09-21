# Архитектура распределённой fuzzing-платформы

## 1. Статус и область действия

Этот документ является нормативным источником domain terminology, lifecycle, consistency guarantees, trust boundaries, портов и архитектурных инвариантов платформы. Слова «должен», «обязан» и «запрещён» обозначают обязательные требования.

Направление зависимости документации: `Architecture → Implementation`. `Implementation.md` обязан показывать, как выбранные компоненты удовлетворяют контрактам этого документа, но не может переопределять domain model, lifecycle или инварианты. При расхождении приоритет имеет `Architecture.md`.

Архитектура не зависит от выбора базы данных, оркестратора, execution platform, хранилища, очереди, fuzzing engine или иных сторонних технологий. Конкретные продукты и operational mapping относятся к реализации и ADR.

## 2. Цели и ограничения

### Цели

- Управлять распределёнными fuzzing-кампаниями как долгоживущими domain entities.
- Разделять пользовательский intent, авторитетное состояние и физическое исполнение.
- Обеспечивать авторизованный доступ к Project, воспроизводимость сборок и трассируемость результатов.
- Восстанавливать полезную работу после потери вычислительных ресурсов без потери domain history.
- Поддерживать finite tasks и непрерывный fuzzing через общий execution contract.
- Формализовать scheduling, admission, preemption, corpus, coverage и findings без привязки к технологии.

### Ограничения области

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
5. восстановление выполняется reconciliation, а не изменением истории;
6. длительные операции асинхронны относительно API.

## 4. Access control

Платформа исполняет доверенный код в едином trust domain: запускаемый fuzzing-workload не рассматривается как источник угрозы, отдельной изоляции между запусками не требуется.

Доступ к данным и операциям авторизуется на уровне `Project`. `User` — идентичность, подтверждённая `IdentityProvider`. `Group` объединяет несколько `User`; `Membership` связывает `User` и `Group` без собственной роли. `ProjectAccessGrant` связывает `Group` и конкретный `Project` и несёт два поля: `revisionAllowList` — допустимые для запуска revision/ref-паттерны исходного кода (по умолчанию только `master`), и `capabilities` — разрешённые действия в этом Project (как минимум `RUN_CAMPAIGN`, `MANAGE_OTHERS_CAMPAIGN`, `TRIAGE_FINDINGS`, `MANAGE_ACCESS`).

Создание `Campaign` на `SourceRevision` X в `Project` P требует, чтобы пользователь состоял в `Group`, имеющей `ProjectAccessGrant` на P с capability `RUN_CAMPAIGN`, и чтобы X попадал в `revisionAllowList` этого grant. Другие операции (pause/stop чужой Campaign, изменение triage state Finding, изменение самих grants) проверяются по соответствующей capability аналогично.

Control Plane владеет всеми domain entities. Execution Backend владеет лишь своими физическими ресурсами и их telemetry. Artifact и corpus adapters хранят bytes и manifests, но их domain ownership и права доступа определяет Control Plane.

## 5. Domain model

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
| `FuzzJob` | Логическая непрерывная задача одного target в Campaign; масштабируется горизонтально через N параллельных `FuzzInstance` | `Campaign` |
| `FuzzInstance` | Один из N параллельных independent fuzzing-инстансов одного `FuzzJob`; владеет собственным lease и попытками, периодически синхронизирует corpus с остальными instances того же `FuzzJob` | `FuzzJob` |
| `Build` | Процесс получения artifacts для revision и recipe | `Project` |
| `BuildArtifact` | Immutable content-addressed результат Build | Исходный `Build`, фактически опубликовавший artifact; cache-hit Builds только ссылаются на него |
| `Task` | Конечная работа: build, reproduce, minimize, normalization, coverage, corpus merge/prune, regression analysis или fix verification | Сущность, которая запросила работу; ссылка обязательна |
| `ExecutionAttempt` | Одна историческая попытка физического выполнения FuzzInstance или Task | Соответствующий `FuzzInstance` или `Task` |
| `ResourceLease` | Ограниченное по времени admission-разрешение на ресурсы | Resource Admission от имени `Project` |
| `Corpus` | Именованная логическая линия входных данных | `Project` и `FuzzTarget` |
| `CorpusSnapshot` | Immutable manifest конкретного состояния Corpus | `Corpus` |
| `FindingOccurrence` | Immutable-факт обнаружения с raw artifacts | `FuzzJob` и породивший `ExecutionAttempt` |
| `CrashReport` | Версионированный результат нормализации occurrence | `FindingOccurrence` |
| `Finding` | Дедуплицированная пользовательская проблема и её triage state | `Project` |

Обязательные отношения владения и происхождения:

```text
Project → Campaign → FuzzJob → FuzzInstance → ExecutionAttempt
Task → ExecutionAttempt
Build → build Task → ExecutionAttempt  (только при фактическом исполнении)
FindingOccurrence → CrashReport → Finding
Group → ProjectAccessGrant → Project
```

`Campaign` фиксирует выбранные targets, `SourceRevision`, build parameters, resource policy и stop policy. `FuzzJob` не привязан к worker или размещению и масштабируется горизонтально через `FuzzInstance` — от 1 до N параллельных независимых instances одного target в пределах конфигурируемого `[min, max]`. `ExecutionAttempt` является попыткой выполнения конкретного `FuzzInstance` (или `Task`), а не зеркалом backend resource. Физический идентификатор хранится только как opaque `BackendResourceRef`. `CampaignBatch` не владеет `FuzzJob` и не участвует в lifecycle входящих `Campaign` — это client-facing группировка для совместного просмотра.

`BuildRecipe` — immutable value object со всеми значимыми параметрами сборки и ссылкой на версию build environment. Он не является самостоятельно управляемым aggregate.

## 6. Авторитетное, desired и observed state

Единственный authoritative domain state принадлежит Control Plane. Execution Backend сообщает observations и может потерять или пересоздать физический ресурс, но не становится source of truth для `Campaign`, `FuzzJob`, `Task` или `ExecutionAttempt`.

Управляемые aggregates обязаны содержать:

- монотонный `generation`, увеличиваемый при изменении desired state;
- `observedGeneration`, до которого reconciler подтвердил обработку;
- version token для optimistic concurrency;
- structured conditions с `reason`, `message` и `lastTransitionAt`;
- owner references, проверяемые при каждом изменении.

Запись observed state принимается только при совпадении version token и применимой `generation`. Устаревшее наблюдение не откатывает более новое состояние. Конфликт optimistic concurrency заставляет обработчик перечитать aggregate и повторно оценить intent.

Изменение desired state и запись соответствующего durable domain event выполняются атомарно. Если атомарная операция не состоялась, ни новое состояние, ни событие не считаются опубликованными. Способ обеспечения атомарности является деталью реализации.

## 7. Lifecycle и state machines

Создание `Campaign` означает запрос на запуск: новая Campaign атомарно получает desired=`RUNNING` и observed phase `CREATED`, после чего обычный reconciler переводит её в `BUILDING`. Отдельных сценариев create-paused, create-stopped и неявного draft lifecycle нет; UI может хранить незавершённую конфигурацию вне Campaign. Команды pause и stop меняют desired state уже существующей Campaign. На create обязательно задаётся хотя бы один обязательный target/FuzzJob, иначе команда отклоняется.

Desired state `Campaign` после создания принимает `RUNNING`, `PAUSED` или `STOPPED`. `PAUSED` допустим только после первого подтверждённого полезного запуска (`startedAt` установлен); это исключает паузу пустой, собирающейся или впервые ожидающей admission Campaign. Observed phase отдельно отражает прогресс применения intent.

### Campaign

| Текущая фаза | Условие | Следующая фаза | Обязательный эффект |
|---|---|---|---|
| `CREATED` | create intent сохранён с desired=`RUNNING` | `BUILDING` | Зафиксировать требуемые Build и зависимости, не создавая build Task до cache lookup; опубликовать событие reconciliation |
| `CREATED` | stop опередил первый reconciliation | `STOPPED` | Зафиксировать отмену ещё не начатого запуска и terminal event без создания дочерней работы |
| `BUILDING` | обязательные artifacts готовы | `QUEUED` | Создать FuzzJob сразу в `QUEUED`; каждый FuzzJob сам создаёт свой admission-запрос |
| `QUEUED` | обязательный FuzzJob начал полезное исполнение | `RUNNING` | Монотонно установить `startedAt`, если он пуст; обновить `observedGeneration` и время активного выполнения |
| `RUNNING` или post-resume `QUEUED` | desired=`PAUSED`, `startedAt` установлен | `PAUSING` | Запретить новую работу, отменить pending admission и запросить checkpoint/stop активных workloads |
| `PAUSING` | FuzzJob `PAUSED`, Tasks не имеют active admission/attempt, все attempts и leases terminal | `PAUSED` | Сохранить ссылки на последние успешно опубликованные snapshots и подтвердить `observedGeneration` |
| `PAUSED` | desired=`RUNNING` | `QUEUED` | Запросить новые leases и указать последние успешные snapshots |
| `BUILDING`, `QUEUED`, `RUNNING`, `PAUSING` или `PAUSED` | desired=`STOPPED` | `STOPPING` | Отменить pending work и остановить активные workloads |
| `STOPPING` | дочерние FuzzJob/Task terminal, их attempts и leases terminal, admission отменён | `STOPPED` | Опубликовать terminal event |
| `RUNNING` | все обязательные jobs успешно достигли terminal condition | `COMPLETED` | Зафиксировать итог, доступные snapshots и terminal event |
| `RUNNING` или post-resume `QUEUED` | полезное исполнение было, но есть невосстановимые ошибки обязательных jobs | `COMPLETED_WITH_ERRORS` | Зафиксировать успешные результаты и перечень ошибок |
| `BUILDING` или initial `QUEUED` | полезное исполнение невозможно и recovery исчерпан | `FAILED` | Зафиксировать причины и отменить оставшуюся работу |

Recoverable build failure также не создаёт фазу или self-transition: Campaign остаётся `BUILDING`, получает `BuildRetrying` condition и выполняет ограниченный policy retry. Временная потеря capacity не создаёт отдельную фазу Campaign: она остаётся `RUNNING`, а доступность отражается conditions `Available`/`Degraded`. Recovery принадлежит конкретным `FuzzJob`. Это не смешивает build retry до первого запуска и runtime recovery в одной фазе.

### FuzzJob

`FuzzJob` не исполняется напрямую: он владеет N параллельными `FuzzInstance` (§«FuzzInstance» ниже) и отвечает только за (1) reconciliation желаемого числа instances в пределах конфигурируемого `[min, max]` по доступной quota и (2) агрегацию фазы и coverage epoch по своим `FuzzInstance`. Собственно self-healing одной попытки, checkpoint и resume принадлежат `FuzzInstance`.

| Текущая фаза | Условие | Следующая фаза | Обязательный эффект |
|---|---|---|---|
| `QUEUED` | admission range подтверждён (доступен хотя бы `min`) | `SCALING` | Reconciler вычисляет desired count в `[min, max]` по доступной quota, создаёт desired count `FuzzInstance` |
| `SCALING` | ≥1 `FuzzInstance` перешёл в `RUNNING` | `RUNNING` | Начать учёт active time агрегированной coverage epoch по wall-clock `FuzzJob` |
| `RUNNING` | периодический reconciliation tick | `RUNNING` (self-loop) | Пересчитать desired count по текущей quota/fair-share; создать недостающие либо инициировать graceful stop лишних `FuzzInstance`, не выходя за `[min, max]` |
| `RUNNING` | `FuzzInstance` потерян (LOST/preemption/node loss) | `RUNNING` (self-loop) | Запросить replacement `FuzzInstance` независимо от остальных |
| `QUEUED`, `SCALING`, `RUNNING` | ранее запущенная Campaign переходит к pause | `PAUSING` | Отменить admission на новые instances; для всех текущих `FuzzInstance` запросить checkpoint/stop до общего deadline |
| `PAUSING` | все `FuzzInstance` terminal либо остановлены | `PAUSED` | Исключить период pause из active time и сохранить последний успешно опубликованный canonical snapshot |
| `PAUSED` | Campaign возобновлена | `QUEUED` | Указать последний успешно опубликованный snapshot как input для новых `FuzzInstance` |
| `RUNNING` | terminal condition StopPolicy выполнен на агрегированной coverage epoch, checkpoint/stop handshake завершён для всех `FuzzInstance` | `COMPLETED` | При успешном checkpoint с новым содержимым опубликовать snapshot; зафиксировать итог coverage epoch и terminal event |
| `QUEUED`, `SCALING`, `RUNNING`, `PAUSING` или `PAUSED` | остановка Campaign либо невосстановимая ошибка | `STOPPING` | Отменить admission, прекратить recovery и остановить/fence все текущие `FuzzInstance` |
| `STOPPING` | все `FuzzInstance` terminal | `CANCELLED` или `FAILED` | Выбрать terminal result по исходной причине и сообщить агрегатору Campaign |

FuzzJob создаётся только после готовности BuildArtifact, сразу в `QUEUED`; его reconciler идемпотентно создаёт admission-запрос на первый `FuzzInstance`. Отдельные `CREATED` и `WAITING_FOR_BUILD` не используются, потому что не отражают самостоятельную работу или ожидание уже существующего FuzzJob.

`[min, max]` конфигурируется per target (или per Campaign как default для всех её targets). Периодический reconciliation tick существует, чтобы один continuous target не монополизировал quota, даже когда она формально свободна: платформа может как добавлять `FuzzInstance` сверх текущего числа при появлении свободной quota, так и снимать их при её нехватке, не выходя за `[min, max]`.

До перехода `FuzzJob RUNNING → COMPLETED` reconciler обязан остановить учёт active time, перевести каждый текущий `FuzzInstance` через `STOPPING` в terminal `CANCELLED` с причиной `STOP_POLICY_COMPLETED`, выполнить checkpoint best effort до deadline для каждого из них, остановить backend независимо от результата checkpoint и освободить все `ResourceLease`. Выполнение StopPolicy без этого handshake не является завершением FuzzJob.

### FuzzInstance

`FuzzInstance` — один из N параллельных независимых fuzzing-инстансов, на которые continuous `FuzzJob` масштабируется горизонтально: каждый фаззит тот же target самостоятельно, без синхронной координации с остальными instances, и периодически обменивается находками через `Corpus` (см. §12). Таблица переходов повторяет self-healing, которым раньше владел непосредственно `FuzzJob` — собственный `ResourceLease`, собственный `ExecutionAttempt`, собственные RECOVERING/PAUSING/STOPPING — и добавляет только периодическую синхронизацию corpus:

| Текущая фаза | Условие | Следующая фаза | Обязательный эффект |
|---|---|---|---|
| `QUEUED` | ResourceLease выдан | `STARTING` | Создать новый ExecutionAttempt |
| `STARTING` | попытка сообщает начало полезного fuzzing | `RUNNING` | Начать учёт active time для своего `ExecutionAttempt` |
| `RUNNING` | наступил sync interval (jittered per instance, чтобы instances не рестартовали синхронно) | `SYNCING` | Запросить best-effort checkpoint локального corpus-роста |
| `SYNCING` | checkpoint успешен | `RUNNING` | Опубликовать private interim `CorpusSnapshot`; если после merge появился новый canonical snapshot — recycle: пересоздать `ExecutionAttempt` с этим snapshot как input |
| `SYNCING` | checkpoint не успел в течение sync-deadline | `RUNNING` | Пропустить этот sync-цикл без потери instance — sync best-effort, не должен ронять fuzzing |
| `STARTING` или `RUNNING` | попытка стала `LOST`, recoverable `FAILED` или `CANCELLED` из-за preemption/admission loss, intent остаётся `RUNNING` | `RECOVERING` | Подтвердить terminal attempt и fencing/release прежнего lease; запросить replacement в пределах recovery policy |
| `RECOVERING` | новый lease выдан | `STARTING` | Создать новый ExecutionAttempt с последним canonical snapshot |
| `QUEUED`, `STARTING`, `RUNNING`, `SYNCING` или `RECOVERING` | владеющий `FuzzJob` переходит к pause | `PAUSING` | Отменить admission; запросить checkpoint/stop до общего deadline |
| `PAUSING` | admission отменён, attempt отсутствует или terminal, lease отсутствует или terminal | `PAUSED` | Исключить период pause из active time и сохранить последний успешный snapshot reference |
| `PAUSED` | владеющий `FuzzJob` возобновлён | `QUEUED` | Указать последний успешно опубликованный snapshot |
| `QUEUED`, `STARTING`, `RUNNING`, `SYNCING`, `RECOVERING`, `PAUSING` или `PAUSED` | остановка владеющего `FuzzJob`, down-scale этого instance по решению `FuzzJob` либо невосстановимая ошибка | `STOPPING` | Отменить admission, прекратить recovery и остановить/fence текущую попытку |
| `STOPPING` | attempt отсутствует либо terminal, lease отсутствует либо terminal | `CANCELLED` | Сохранить structured reason и сообщить владеющему `FuzzJob` |

Переход `RUNNING → COMPLETED` по выполнению StopPolicy на `FuzzInstance` **не существует** — StopPolicy оценивается только на уровне `FuzzJob` по агрегированной coverage epoch (§13). `FuzzInstance` завершается терминально только по команде своего `FuzzJob` (через `STOPPING`, в том числе при down-scale), всегда как `CANCELLED` со structured reason, а не самостоятельно как `COMPLETED`.

Джиттер sync-интервала обязателен: без него все `FuzzInstance` одного `FuzzJob` синхронно уходили бы в `SYNCING` и кратковременно теряли бы всю fuzzing-мощность target'а одновременно.

### ExecutionAttempt

| Текущая фаза | Условие | Следующая фаза | Обязательный эффект |
|---|---|---|---|
| `PENDING` | lease и входы подтверждены | `STARTING` | До backend call durable сохранить start intent и operation key, затем идемпотентно запросить запуск |
| `PENDING` | запуск отменён до первого backend call | `CANCELLED` | Освободить lease и записать terminal event без stop handshake |
| `PENDING` | permanent failure проверки immutable inputs | `FAILED` | Сохранить diagnostics и terminal event; освободить lease |
| `STARTING` | backend подтвердил готовность workload | `RUNNING` | Сохранить start time и применимую observation sequence |
| `STARTING` или `RUNNING` | конечный workload опубликовал все обязательные outputs | `SUCCEEDED` | Сохранить output references, подтвердить отсутствие исполнения и освободить lease |
| `STARTING` или `RUNNING` | workload сообщил невосстановимую ошибку запуска или исполнения | `FAILED` | Сохранить diagnostics, подтвердить отсутствие исполнения и освободить lease |
| `STARTING` или `RUNNING` | backend resource потерян или observation протухло | `LOST` | Подтвердить отсутствие resource либо fencing, зафиксировать последний snapshot reference и освободить lease |
| `STARTING` или `RUNNING` | StopPolicy, stop, pause, preemption или admission loss требует остановки | `STOPPING` | Запросить best-effort checkpoint с deadline, затем обязательный stop |
| `STOPPING` | backend подтвердил отсутствие resource либо execution fencing доказуемо исключает продолжение | `CANCELLED` | Сохранить structured reason, результат checkpoint и последний опубликованный resumable state; освободить lease |

Opaque `BackendResourceRef` сохраняется в фазе `STARTING` сразу после подтверждённого backend create/lookup, не ожидая readiness; это observation update, а не отдельный lifecycle transition. `SUCCEEDED`, `FAILED`, `LOST` и `CANCELLED` — terminal phases `ExecutionAttempt`. Retry всегда создаёт новый `ExecutionAttempt`. Истечение checkpoint deadline переводит операцию к принудительному stop, но само по себе не доказывает остановку backend и не разрешает terminal transition или повторное использование lease.

### Build

| Текущая фаза | Условие | Следующая фаза | Обязательный эффект |
|---|---|---|---|
| `PENDING` | найден verified cache hit | `SUCCEEDED` | Связать Build с существующим BuildArtifact и его provenance без новой Task/попытки |
| `PENDING` | связанная build Task вошла в `RUNNING` | `RUNNING` | Сохранить ссылку на build Task и наблюдать её результат, не создавая отдельную попытку |
| `RUNNING` | build Task получила `SUCCEEDED`, artifact опубликован и digest проверен | `SUCCEEDED` | Атомарно связать Build с BuildArtifact, Task, ExecutionAttempt и provenance |
| `PENDING` или `RUNNING` | отмена владельцем и build Task/attempt подтверждённо завершена либо отсутствует | `CANCELLED` | Сохранить причину и terminal event после quiescence дочерней работы |
| `PENDING` или `RUNNING` | build Task получила `FAILED` либо исполнение невозможно до запуска | `FAILED` | Сохранить diagnostics и оповестить зависимые jobs |

Reconciler Build в `PENDING` сначала выполняет идемпотентный cache lookup. Проверенный hit завершает Build без новой Task/попытки и сохраняет provenance исходного artifact. При miss reconciler идемпотентно создаёт и связывает build Task, сохраняя Build в `PENDING`; это side effect, а не self-transition. Если сборка действительно исполняется, цепочка однозначна: `Build → build Task → ExecutionAttempt`. Только build Task создаёт и владеет `ExecutionAttempt`; Build наблюдает lifecycle Task и её результат.

### Task

| Текущая фаза | Условие | Следующая фаза | Обязательный эффект |
|---|---|---|---|
| `PENDING` | обязательная зависимость terminal и недоступна | `FAILED` | Сохранить dependency condition и не создавать admission-запрос |
| `PENDING` | зависимости готовы | `QUEUED` | Создать admission-запрос с project context |
| `QUEUED` | ResourceLease выдан | `RUNNING` | Создать новый ExecutionAttempt |
| `QUEUED` | permanent admission или policy rejection запрещает исполнение | `FAILED` | Сохранить structured rejection и terminal event; отменить admission-запрос и дальнейшие retries |
| `RUNNING` | текущая попытка `SUCCEEDED`, обязательные outputs опубликованы, lease освобождён | `SUCCEEDED` | Сохранить output references и terminal event |
| `RUNNING` | попытка завершилась `LOST` или recoverable `FAILED`, finite retry policy разрешает | `QUEUED` | Сохранить причину и израсходовать attempt budget; следующий запуск создаёт новый ExecutionAttempt |
| `RUNNING` | pause, preemption, admission loss или infrastructure cancellation завершили попытку как `CANCELLED`, disruption/deadline policy разрешает | `QUEUED` | Сохранить resumable state; новый admission запрещён, пока действует pause intent владельца |
| `PENDING` или `QUEUED` | явная отмена владельцем, active attempt отсутствует | `CANCELLED` | Прекратить дальнейшие retries, отменить admission и записать terminal event |
| `RUNNING` | явная отмена владельцем, попытка terminal и lease terminal | `CANCELLED` | До quiescence оставаться `RUNNING` с condition `CancellationRequested` |
| `RUNNING` | attempt terminal, lease terminal, ошибка невосстановима или retries исчерпаны | `FAILED` | Сохранить diagnostics и уведомить владельца Task |

Pause и preemption используют checkpoint best effort до заданного deadline для workload, который объявляет checkpoint capability. После deadline Control Plane принудительно останавливает workload независимо от результата checkpoint; `ExecutionAttempt` получает `CANCELLED` со structured reason `PAUSED` или `PREEMPTED` только после подтверждённой остановки/fencing. Успешный checkpoint не означает `SUCCEEDED`. Resume FuzzJob использует последний успешно опубликованный `CorpusSnapshot`, а не локальное неподтверждённое состояние остановленной попытки. Остановленная pause конечная Task остаётся в `QUEUED` без admission, пока действует pause intent её владельца. Retry Task использует опубликованный resumable state, только если её contract объявляет его совместимым; иначе новая попытка начинает работу заново из immutable inputs. Failure retry budget конечен; pause/preemption могут иметь отдельный disruption budget, но общий deadline/recovery policy обязан исключать бесконечный requeue.

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

`SourceRevision` разрешается в immutable identity до начала Build. Fingerprint Build включает revision, все значимые поля `BuildRecipe`, target, архитектуру выполнения и совместимость instrumentation. Одинаковый fingerprint может переиспользовать только artifact, прошедший integrity check.

`BuildArtifact` является immutable и content-addressed: его `artifactDigest` является provenance identity содержимого. Provenance связывает artifact с Project, SourceRevision, BuildRecipe, исходными Build, Task/ExecutionAttempt и временем публикации. Cache-hit Build ссылается на этот artifact и исходный provenance, не приписывая себе его создание. Потребляющие `FuzzJob`, reproduce/analysis Task и принадлежащие им `ExecutionAttempt` до запуска ссылаются на точный `artifactDigest`.

Build Task и её `ExecutionAttempt` производят `BuildArtifact`, поэтому до успешной публикации не могут ссылаться на его будущий `artifactDigest` как на input. Их обязательные immutable inputs — `SourceRevision`, `BuildRecipe`, ссылка на build environment и остальные входные references. Только после проверки и публикации output digest становится результатом Task и связывается с Build.

Публикация считается успешной только после проверки digest и доступности manifest. Частично записанный artifact не становится видимым через domain state.

## 10. Execution model

`FuzzJob` представляет непрерывный workload, а `Task` — конечный workload. Оба используют `ExecutionAttempt`, но сохраняют независимые lifecycle и retry policies.

Одна попытка соответствует одному созданию физического backend resource. Внутренний restart процесса в рамках того же ресурса остаётся той же попыткой; создание нового ресурса требует новой попытки. Потеря ресурса завершает текущую попытку как `LOST`, но не удаляет её историю и не завершает автоматически владеющий FuzzJob или Task.

Retry всегда создаёт новый `ExecutionAttempt` с новым identity и ссылкой на предыдущую попытку как причину retry. Одновременно активные попытки одного workload допустимы только при явной policy и уникальных ownership scopes для outputs.

`ExecutionAttempt` получает `SUCCEEDED` только после естественного завершения конечного workload и публикации всех обязательных outputs. Остановка из-за StopPolicy, pause, preemption или admission loss всегда даёт текущей попытке `CANCELLED`; владеющий FuzzJob или Task отдельно решает, перейти ли в paused state, вернуться в queue или завершиться по policy.

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
- `projectId`;
- нормализованный resource request;
- priority и policy decision reference;
- `issuedAt`, `expiresAt` и уникальный lease identity.

Запуск без действующего lease запрещён. Минимальный lifecycle lease: `ACTIVE → RELEASED|REVOKED|EXPIRED`; terminal lease не реактивируется, а renewal меняет `expiresAt` только у `ACTIVE`. Lease освобождается после terminal attempt либо после подтверждённого execution fencing. Истечение lease прекращает право на ресурсы и запускает reconciliation; оно не стирает attempt history и само по себе не доказывает остановку workload.

Preemption является поддерживаемым архитектурным контрактом, даже если policy конкретной реализации отключена. Если attempt ещё не создан, отзыв lease лишь отменяет admission и возвращает workload к ожиданию; checkpoint/stop не запускается. Для attempt в `STARTING` или `RUNNING` Control Plane запрашивает checkpoint best effort до deadline, затем Execution Backend обязательно останавливает workload и подтверждает отсутствие resource либо fencing. Только после этого ExecutionAttempt получает `CANCELLED` с причиной `PREEMPTED`, а lease становится terminal. FuzzJob возобновляется с последнего успешно опубликованного snapshot. Конечная Task при разрешённом bounded retry переходит в `QUEUED`: новая попытка использует совместимый опубликованный resumable state либо начинает работу из immutable inputs.

## 12. Corpus и snapshots

`Corpus` — именованная логическая линия для пары Project/FuzzTarget и заданного compatibility scope. Он указывает на canonical snapshot через версионированную ссылку, но не является общей mutable директорией.

`CorpusSnapshot` — immutable manifest, который содержит:

- собственный digest и checksums каждого объекта;
- optional parent snapshot;
- `corpusCompatibilityKey` для target, runtime input format и corpus schema, не подменяющий artifact identity или coverage compatibility;
- Project, Corpus и creator ExecutionAttempt;
- список content-addressed объектов и metadata публикации.

Полезная ExecutionAttempt Campaign может публиковать private campaign snapshot; при отсутствии новых данных или неуспешном checkpoint snapshot может отсутствовать. Несколько Campaign не изменяют один общий mutable corpus. Snapshot становится доступен для resume только после полной публикации manifest и проверки checksums.

Policy автоматически создаёт `CorpusMergeTask` по lifecycle event, интервалу или порогу только при наличии нового совместимого опубликованного snapshot с реальным delta. Ручной on-demand trigger создаёт ту же Task и проходит те же authorization, admission и audit rules, но тот же gate не позволяет создавать пустую merge-работу. Merge/prune фиксирует expected canonical snapshot identity и version, читает immutable inputs и публикует новый snapshot только при непустом effective delta. Затем `CorpusStore` выполняет compare-and-swap (CAS) canonical reference только при совпадении ожидаемых identity и version. При CAS conflict новый snapshot не заменяет текущую ссылку: Task перечитывает новый canonical snapshot и повторно вычисляет merge. Если конкурент уже поглотил весь effective delta, Task завершается успешно без новой публикации и CAS; иначе она повторяет публикацию в пределах finite retry policy. Исчерпание retries завершает `CorpusMergeTask` как `FAILED`, но не завершает Campaign, если merge явно не объявлен обязательным. Предыдущий canonical snapshot сохраняется согласно rollback и retention policy.

Несовместимый `corpusCompatibilityKey` запрещает silent merge или resume. Требуется новая corpus line либо явная преобразующая Task, результат которой имеет новый key и provenance.

Горизонтальное масштабирование continuous `FuzzJob` на N параллельных `FuzzInstance` (§7) переиспользует этот же механизм для периодической синхронизации corpus между instances, пока fuzzing ещё идёт: каждый `FuzzInstance` на `SYNCING` публикует private interim `CorpusSnapshot` своего роста, существующий policy-триггер создаёт `CorpusMergeTask` на новый совместимый delta без отдельного API, а `FuzzInstance` на следующем recycle берёт актуальный canonical snapshot как input — по тому же контракту, что и `RECOVERING → STARTING`. Синхронизация не подгружает corpus в уже работающий процесс fuzzing engine — это осознанная eventual-consistency модель, а не live reload.

## 13. Coverage и условия завершения

Основной stop criterion — `StopPolicy.noCoverageGrowthFor`: отсутствие роста coverage в течение `N` активных часов. Окно считается отдельно для каждого `FuzzJob`, а не на уровне Campaign. Если `FuzzJob` масштабирован на несколько параллельных `FuzzInstance` (§7), окно остаётся привязано к wall-clock active time самого `FuzzJob`, а не суммируется по числу instances: рост throughput от параллелизма не должен искусственно ускорять срабатывание timeout, настроенного в календарных единицах.

Active time и условия `noCoverageGrowthFor`/`maxActiveTime` начинают оцениваться только после первого подтверждённого полезного fuzzing в фазе `RUNNING`. Queue, build, pause, preemption, checkpoint, recovery и ожидание admission исключаются. Потерянные или дублированные telemetry intervals не должны дважды увеличивать active time. Отсутствие здоровой совместимой coverage telemetry не считается отсутствием роста: окно стагнации продвигается только по валидной выборке, а telemetry gap создаёт condition и recovery action.

Coverage record обязан нести два независимых значения: `artifactDigest` для provenance конкретного исполнявшегося содержимого и `coverageCompatibilityKey` для семантической совместимости target, instrumentation, feature schema и normalizer contract. Разные `BuildArtifact` могут иметь одинаковый `coverageCompatibilityKey`; их совместимая telemetry объединяется в текущей epoch с сохранением provenance каждого artifact. Это естественно распространяется на несколько параллельных `ExecutionAttempt` от разных `FuzzInstance` одного `FuzzJob`: все они используют один `BuildArtifact` (см. инвариант в §19) и объединяются в одну coverage epoch `FuzzJob` с сохранением provenance каждого attempt.

Новая coverage epoch начинается только при изменении `coverageCompatibilityKey`, даже если `artifactDigest` не изменился. Смена одного `artifactDigest` при неизменном key не сбрасывает baseline или окно `noCoverageGrowthFor`; несовместимый key создаёт новую epoch и сбрасывает окно для неё. История старых epochs сохраняется.

`StopPolicy` также может содержать `maxActiveTime` как safety limit и `deadline` как абсолютный предел. Условия объединяются по правилу OR: выполнение любого явно настроенного условия инициирует завершение. `noCoverageGrowthFor` остаётся основным criterion. Абсолютный deadline, достигнутый до первого полезного исполнения, завершает Campaign как `FAILED`, а не создаёт фиктивно успешный FuzzJob.

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

`FindingOccurrence` неизменяемо фиксирует момент обнаружения, Project, FuzzJob, FuzzInstance, ExecutionAttempt, artifact digest, input и raw outputs. Повторная обработка не изменяет occurrence.

`FindingNormalizer` создаёт новый `CrashReport` с `normalizerName`, `normalizerVersion`, normalized stack, classification и ссылками на inputs. Смена версии нормализатора создаёт новый report и сохраняет прежние результаты для объяснимости.

`FindingDeduplicator` вычисляет версионированный deduplication key из CrashReport и project policy. Решение merge/split обязано сохранять rule version и evidence. Finding ведёт triage states, включая `OPEN`, `ACKNOWLEDGED`, `FIXED`, `IGNORED` и `REOPENED`.

Новое совместимое occurrence, совпавшее с `FIXED` Finding, переводит его в `REOPENED` и создаёт audit event. Новая версия нормализатора или deduplication rule не переписывает историю; массовая переоценка выполняется отдельной Task.

## 15. Security requirements

Каждая операция аутентифицируется, авторизуется через `ProjectAccessGrant` в scope конкретного `Project` и оставляет audit event для security-sensitive изменений. Workload не получает credentials Control Plane и не может выбирать lease или backend reference.

Запускаемый fuzzing-код рассматривается как доверенный: платформа не выбирает и не применяет отдельный класс изоляции workload, sandbox implementation или runtime restrictions поверх стандартного исполнения в кластере.

Secrets передаются workload только в минимально необходимом scope, не сохраняются в artifacts, logs или corpus и имеют ограниченный срок действия. Все storage operations проверяют ownership на уровне `Project`; content address сам по себе не даёт права чтения.

## 16. Observability

Платформа обязана предоставлять metrics, structured logs, distributed traces и audit events с согласованными correlation identifiers: `projectId`, aggregate identity, `taskId`/`fuzzJobId`, `executionAttemptId` и operation identity.

Telemetry разделяется на два класса:

1. **Normalized coverage/fuzzing telemetry**: active time, coverage epoch, coverage growth, executions, findings, corpus и fuzzer health. Эти данные проходят нормализацию и участвуют в domain policies.
2. **Infrastructure telemetry**: capacity, placement, process/resource health, adapter latency и backend failures. Эти данные объясняют исполнение, но сами не меняют domain lifecycle без reconciliation rule.

Metrics labels обязаны иметь ограниченную cardinality. Нельзя использовать raw input, stack trace, artifact digest, backend resource identity или произвольный пользовательский текст как неограниченный label. Высококардинальные данные помещаются в структурированные logs или trace attributes с retention и access policy.

Audit events неизменяемо фиксируют actor, project scope, command, target, result, policy decision и timestamp. Обязательная audit-запись добавляется через `AuditStore` идемпотентной append-операцией с integrity metadata. Для security-sensitive операции запись должна быть атомарна с изменением authoritative state либо надёжно сохранена как обязательная предпосылка; сбой append, integrity или authorization не допускает изменения domain state. Retention не разрешает преждевременное удаление, а чтение и экспорт audit records всегда project-scoped и отдельно авторизованы.

## 17. Порты и границы модулей

Domain modules зависят от портов, а adapters реализуют порты. Domain code не импортирует backend types и не интерпретирует opaque references.

| Порт | Нормативный контракт |
|---|---|
| Repositories (`ProjectRepository`, `CampaignRepository`, `WorkloadRepository`, `FindingRepository`) | Загружать и атомарно сохранять project-scoped aggregates с optimistic concurrency; поддерживать изменение desired state вместе с durable event |
| `EventPublisher` | Публиковать сохранённые domain events с семантикой at least once, сохраняя aggregate identity и sequence |
| `BuildExecutor` | Выполнять build operation по immutable inputs внутри ExecutionAttempt, принадлежащего build Task; возвращать provenance и artifact digest, не создавая попытку от имени Build |
| `ExecutionBackend` | Идемпотентно start/observe/checkpoint/stop workload; возвращать capabilities и opaque `BackendResourceRef` |
| `ResourceAdmission` | Выдавать, продлевать, отзывать и освобождать `ResourceLease` по quotas, priority и policy |
| `CorpusStore` | Публиковать и читать immutable snapshots, проверять checksums и менять canonical reference через compare-and-swap по expected snapshot identity и version |
| `ArtifactStore` | Публиковать и читать content-addressed artifacts с integrity и project authorization |
| `CoverageAnalyzer` | Нормализовать coverage в совместимой epoch и определять growth без backend-specific semantics |
| `FindingNormalizer` | Создавать версионированный CrashReport из immutable occurrence, не изменяя source fact |
| `FindingDeduplicator` | Принимать версионированное и объяснимое merge/split решение для Finding |
| `TaskExecutor` | Маршрутизировать конечные Task через общий execution contract и возвращать typed output references |
| `IdentityProvider` | Подтверждать identity и memberships; authorization остаётся обязанностью Control Plane |
| `FindingSink` | Идемпотентно экспортировать Finding во внешнюю систему без передачи ей authoritative ownership |
| `AuditStore` | Идемпотентно добавлять immutable project-scoped audit records; обеспечивать integrity, retention и авторизованный доступ; fail closed без изменения security-sensitive state, если обязательную запись нельзя надёжно сохранить |

Входные порты API отвечают за validation, authentication, authorization и idempotency key. Policy modules принимают domain facts и возвращают решения без прямых infrastructure calls. Adapter-specific configuration находится за composition boundary.

## 18. Failure semantics

Ошибки классифицируются как transient, permanent, conflict, stale или policy rejection. Retry допускается только для transient failure и всегда ограничивается policy; permanent failure сохраняется как structured condition. Ни одна ошибка адаптера не даёт права переписать terminal history.

| Сбой | Обнаружение | Обязательная реакция | Восстановление/результат |
|---|---|---|---|
| Потеря backend resource | Not-found, expired observation или подтверждённая потеря | Завершить текущий ExecutionAttempt как `LOST` после подтверждения отсутствия/fencing, освободить lease, сохранить diagnostics | При сохраняющемся intent запросить новый lease и создать новый ExecutionAttempt с последним snapshot |
| Сбой checkpoint | Ошибка публикации или deadline | Записать condition; после deadline перейти к обязательному force-stop, но не считать backend уже остановленным | После подтверждённой остановки/fencing завершить attempt как `CANCELLED`; FuzzJob использует последний успешный snapshot, Task — совместимый resumable state либо immutable inputs; partial checkpoint игнорируется |
| Повторное событие | Уже обработанный event identity или sequence | Вернуть сохранённый логический результат без повторного side effect | Продолжить обработку последующих событий; факт duplicate доступен в telemetry |
| Устаревшее обновление | Старая generation, version или terminal attempt | Отклонить изменение authoritative state | Перечитать aggregate; новое observed state принимается только после reconciliation |
| Потеря admission-разрешения | Lease отозван, истёк или не может быть продлён | Без attempt отменить admission и requeue; с attempt в `STARTING`/`RUNNING` инициировать checkpoint/stop и дождаться terminal/fencing перед release | FuzzJob переходит в `RECOVERING`, а Task с разрешённым bounded retry — в `QUEUED`; новый запуск требует нового lease и нового ExecutionAttempt |
| Недоступность artifact storage | Timeout или failed integrity/readiness check | Не объявлять Build/Task успешной и не публиковать partial reference | Повторять в пределах policy; при исчерпании завершить работу ошибкой, не повреждая прежний artifact |
| Конфликт canonical reference корпуса | CAS обнаружил несовпадение expected snapshot identity или version | Не изменять canonical reference и перечитать актуальный snapshot | Повторно вычислить merge с новым canonical input в пределах finite retry policy; при исчерпании завершить merge Task как `FAILED` |
| Сбой обязательной audit-записи | Append, integrity check или authorization `AuditStore` завершились ошибкой | Не применять security-sensitive domain transition и сохранить диагностируемый отказ без обходного канала | Повторить только идемпотентную append-операцию в пределах policy; восстановить доступ с сохранением retention и access integrity до повторной авторизованной команды |

## 19. Архитектурные инварианты

1. Control Plane является единственным владельцем authoritative domain state; observations инфраструктуры не заменяют его.
2. Доступ к `Project` и операциям над его сущностями авторизуется через `ProjectAccessGrant` соответствующей `Group`.
3. Любой ExecutionAttempt принадлежит ровно одному FuzzInstance или Task; retry создаёт новую попытку и сохраняет историю предыдущей.
4. ExecutionAttempt описывает попытку исполнения, а `BackendResourceRef` остаётся opaque и не участвует в domain identity или lifecycle rules.
5. Изменение desired state и durable domain event атомарны; внешние side effects выполняются только после сохранения intent.
6. Consumers, reconcilers и adapters идемпотентны при повторной доставке команд, событий и observations.
7. `observedGeneration` никогда не опережает `generation`, а stale update не откатывает более новое или terminal состояние.
8. FuzzInstance и Task не запускаются без действующего ResourceLease того же Project и совместимого resource request.
9. BuildArtifact и CorpusSnapshot immutable, content-verified и становятся видимыми только после полной успешной публикации.
10. Canonical CorpusSnapshot меняется только через CAS по expected snapshot identity и version; conflict не заменяет ссылку и требует повторного merge с актуальным canonical input.
11. Потребляющие FuzzJob/Task и их ExecutionAttempt ссылаются на точный `artifactDigest`; если сборка исполняется, build Task/attempt вместо будущего output фиксирует immutable SourceRevision, BuildRecipe, build environment и input references, а provenance опубликованного или найденного в cache результата ведёт к исходному исполнению.
12. Resume и periodic sync FuzzInstance используют только последний успешно опубликованный совместимый CorpusSnapshot.
13. Окно `noCoverageGrowthFor` считается по active time отдельно для каждого FuzzJob (по его wall-clock, а не суммарно по параллельным FuzzInstance) и не включает queue, build, pause, preemption или recovery.
14. `artifactDigest` задаёт provenance, а `coverageCompatibilityKey` — семантическую совместимость; только несовместимый key начинает новую coverage epoch и сбрасывает её окно стагнации.
15. FindingOccurrence неизменяем; CrashReport и deduplication decisions версионированы; повтор исправленной проблемы может создать `REOPENED`.
16. Временная потеря одной попытки не переводит Campaign в `FAILED`; terminal outcome вычисляется по полезному исполнению и обязательным jobs.
17. Pause и preemption после checkpoint deadline обязательно останавливают workload; terminal attempt и release lease допустимы только после подтверждённой остановки либо execution fencing.
18. Ни один workload не получает credentials Control Plane.
19. FuzzInstance принадлежит ровно одному FuzzJob; число одновременно живых FuzzInstance под одним FuzzJob всегда находится в пределах `[min, max]`, кроме переходных окон `SCALING` и replacement после потери instance.
20. Все FuzzInstance одного FuzzJob используют один и тот же BuildArtifact и corpusCompatibilityKey: они шардят исполнение одного target на одной ревизии, а не разные ревизии или targets.
