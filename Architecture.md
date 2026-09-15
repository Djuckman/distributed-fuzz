## 1. Overview

Система предназначена для управления распределёнными fuzzing-кампаниями и выполнения fuzzing workloads на вычислительном кластере.

Основные задачи системы:
- управление проектами и fuzzing campaigns;
- сборка fuzz targets для заданной ревизии;
- распределение fuzzing jobs по доступным ресурсам;
- управление lifecycle кампаний;
- сохранение и повторное использование corpus;
- сбор и triage crashes;
- наблюдаемость fuzzing-процесса;
- восстановление после отказов worker-машин;
- предоставление API для CI/CD и внешних интеграций.

Архитектура строится вокруг разделения **domain/control plane** и **execution infrastructure**.

```
                         ┌─────────────────┐
                         │     UI / API    │
                         └────────┬────────┘
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │      Control Plane      │
                    │                         │
                    │ Projects                │
                    │ Campaigns               │
                    │ Jobs / Instances        │
                    │ Lifecycle               │
                    │ Scheduling policies     │
                    │ Findings / Triage       │
                    └───────────┬─────────────┘
                                │
                      ports / abstractions
                                │
             ┌──────────────────┼──────────────────┐
             ▼                  ▼                  ▼
      Build Executor      Execution Backend     Storage
             │                  │                  │
             ▼                  ▼                  ▼
        Build System         Cluster         Corpus/Artifacts
                                │
                    ┌───────────┼───────────┐
                    ▼           ▼           ▼
                 Instance    Instance    Instance
                    │           │           │
                 Fuzzer      Fuzzer      Fuzzer
```

Control Plane не должен зависеть от конкретной БД, object storage, очереди задач или execution platform. Взаимодействие с инфраструктурой выполняется через специализированные интерфейсы.

---

## 2. Основные сущности

|Сущность|Назначение|
|---|---|
|**Project**|Логический fuzzing-проект. Содержит информацию об исходном коде, доступных fuzz targets и конфигурации проекта.|
|**SourceRevision**|Конкретная версия исходного кода: commit, tag или другая ревизия, используемая для сборки.|
|**FuzzTarget**|Отдельный fuzzing target проекта. Каждый target запускается через унифицированный runtime-интерфейс.|
|**Campaign**|Пользовательская единица управления fuzzing-запуском. Определяет revision, набор targets, engine, sanitizer, build options, resource limits и policy запуска.|
|**Build**|Immutable результат сборки конкретной SourceRevision с определёнными engine/sanitizer/build parameters.|
|**Job**|Логическая fuzzing-задача внутри Campaign. Обычно соответствует комбинации Target + Build + runtime configuration.|
|**Instance**|Конкретный экземпляр выполнения Job на execution backend. В Kubernetes соответствует конкретному Pod.|
|**Corpus**|Логический corpus, связанный с fuzzing target/job и сохраняемый между запусками.|
|**CorpusSnapshot**|Сохранённое состояние corpus на определённый момент времени. Используется для восстановления и переноса выполнения.|
|**Finding**|Обнаруженный crash или другой значимый результат фаззинга: sanitizer finding, timeout и т. п.|
|**TriageTask**|Конечная задача обработки Finding: reproduce, minimize, regression analysis или fix verification.|
|**WorkerResource**|Абстрактное представление вычислительного ресурса execution backend: CPU, RAM, architecture, health и дополнительные attributes.|

---

## 3. Campaign, Job и Instance

Эти сущности намеренно разделяются.

### Campaign

`Campaign` описывает пользовательский intent:

```
"Фаззить revision X проекта Y
 targets A, B и C
 с ASAN/libFuzzer
 в течение 72 часов"
```

Campaign может содержать несколько независимых Jobs.

```
Campaign
   │
   ├── Job: target A
   ├── Job: target B
   └── Job: target C
```

### Job

`Job` является **логической fuzzing-задачей**.

Пример:

```
Project:   nginx
Revision:  abc123
Target:    http_request_fuzzer
Engine:    libFuzzer
Sanitizer: ASAN
Build:     build-42
```

Job не должен быть связан с конкретной worker-машиной или Pod.

### Instance

`Instance` является **конкретным физическим исполнением Job**.

```
Job A
  │
  └── Instance #17
         │
         └── Pod
```

Разделение `Job` и `Instance` необходимо прежде всего для корректной обработки отказов.

Например:

```
Job A
  │
  ├── Instance #17 → LOST
  │
  └── Instance #18 → RUNNING
```

Worker может умереть, Pod может быть пересоздан или перемещён на другую машину, но сама логическая Job продолжает существовать.

Это позволяет:

- восстанавливать выполнение после отказов;
- хранить историю запусков;
- отделить desired state от физического размещения;
- в будущем поддержать несколько Instances одной Job без изменения domain model.

В текущей модели обычно выполняется:

```
1 Job → 1 active Instance
```

Поддержка нескольких одновременно работающих Instances одной Job не является обязательным требованием первой версии.

---

## 4. Desired state и reconciliation

Управление системой строится вокруг **desired-state model**, а не последовательности низкоуровневых команд.

Например:

```
Campaign desired state = RUNNING
Job desired instances  = 1
```

Control Plane периодически сравнивает desired state с actual state:

```
desired instances = 1
actual instances  = 0
        ↓
create new Instance
```

Если worker или Pod исчезает:

```
Instance → LOST
```

Job не считается потерянной. Control Plane создаёт replacement Instance и восстанавливает corpus из последнего checkpoint.

Такой подход позволяет естественно реализовать fault tolerance без отдельного recovery workflow.

---

## 5. Lifecycle

### Campaign

Основные состояния:

```
CREATED
   ↓
QUEUED
   ↓
BUILDING
   ↓
RUNNING
   ├──→ PAUSED
   ├──→ STOPPED
   ├──→ COMPLETED
   └──→ FAILED
```

### Instance

```
PENDING
   ↓
STARTING
   ↓
RUNNING
   ├──→ TERMINATED
   ├──→ FAILED
   └──→ LOST
```

Состояние Campaign не должно напрямую зависеть от кратковременного состояния отдельного Instance.

Например, замена потерянного Instance не должна переводить Campaign в `FAILED`.

### Pause

`pause` означает:

1. сохранить актуальный corpus;
2. прекратить выполнение Instances;
3. сохранить Campaign и Job;
4. позволить позднее продолжить выполнение с накопленным corpus.

### Restart

`restart` создаёт новое физическое исполнение Job, но сохраняет:

- Campaign;
- Build;
- Job;
- corpus;
- ранее найденные Findings.

---

## 6. Build и runtime

Build отделён от выполнения fuzzing Job.

```
SourceRevision
      │
      ▼
   Build
      │
      ▼
 BuildArtifact
      │
      ▼
     Job
```

Build должен быть immutable и идентифицировать как минимум:

- source revision;
- architecture;
- engine;
- sanitizer;
- build mode/options.

Это позволяет:

- повторно использовать результат сборки;
- гарантировать воспроизводимость Finding;
- запускать reproduce/minimize на том же build;
- однозначно связывать crash с кодом, на котором он был найден.

### Unified runtime

Все fuzz targets запускаются через единый runtime contract.

Control Plane не должен знать engine-specific детали запуска.

Условно:

```
Generic Runner
      │
      ▼
run(target, corpus, runtimeOptions)
```

Runner отвечает за:

- восстановление corpus;
- запуск fuzz target;
- health monitoring;
- сбор engine-specific metrics;
- публикацию Findings;
- checkpoint corpus;
- graceful shutdown.

---

## 7. Scheduling

Scheduling разделён на два уровня.

### Fuzzing-level scheduling

Control Plane определяет:

- какие Jobs могут быть запущены;
- сколько ресурсов может использовать Campaign;
- priorities;
- quotas;
- ограничения проекта или команды.

То есть отвечает на вопрос:

```
"Что сейчас должно выполняться?"
```

### Infrastructure scheduling

Execution backend определяет конкретное физическое размещение Instance:

```
Instance A → worker 17
Instance B → worker 22
```

То есть отвечает на вопрос:

```
"Где это должно выполняться?"
```

Такое разделение позволяет не дублировать возможности underlying cluster scheduler.

---

## 8. Infrastructure abstraction

Control Plane не должен напрямую зависеть от Kubernetes или другой платформы.

Для этого используется интерфейс типа:

```
ExecutionBackend
    Start(instance)
    Stop(instance)
    Status(instance)
    Capacity()
```

Первая реализация может использовать Kubernetes:

```
ExecutionBackend
       │
       ▼
 Kubernetes
       │
       ▼
      Pod
```

Но domain model не должен содержать Kubernetes-specific сущности вроде Pod, Node, Deployment или Job.

Это позволяет:

- тестировать Control Plane без Kubernetes;
- потенциально поддержать другой backend;
- избежать проникновения infrastructure details в business logic.

---

## 9. Storage model

Архитектура не должна зависеть от конкретных технологий хранения.

Вместо generic abstractions вида `Database` или `S3Client` используются domain-oriented interfaces.

```
CampaignRepository
CorpusStore
ArtifactStore
```

### CorpusStore

Отвечает за:

```
Restore
Checkpoint
Merge
Prune
```

Физически corpus может храниться где угодно.

### ArtifactStore

Хранит immutable artifacts:

- crashing testcase;
- minimized testcase;
- stack trace;
- coverage reports;
- logs;
- дополнительные результаты triage.

### Metadata

Metadata Campaign, Job, Finding и других сущностей хранится через repository interfaces.

Конкретная реализация persistence не является частью domain architecture.

---

## 10. Corpus lifecycle

В базовом сценарии:

```
1 Job → 1 active Instance
```

поэтому online synchronization corpus между workers не требуется.

Основной lifecycle:

```
Central Corpus
      │
    restore
      ▼
Local Corpus
      │
    fuzzing
      │
  checkpoint
      ▼
Central Corpus
```

При остановке, pause или отказе Instance используется последний сохранённый snapshot.

Corpus должен переживать:

- restart Instance;
- migration на другую worker-машину;
- restart Campaign;
- запуск нового Build при необходимости повторного использования corpus.

Merge/pruning рассматриваются как отдельные finite operations и не должны усложнять основной fuzzing loop.

---

## 11. Findings и triage

Fuzzing worker отвечает только за обнаружение и сохранение исходного Finding.

```
Fuzzer
   │
 crash
   ▼
Finding
```

Finding должен содержать как минимум:

- project;
- target;
- build/source revision;
- sanitizer;
- timestamp;
- testcase/reproducer;
- normalized stack trace;
- crash type;
- status.

Дальнейшая обработка выполняется отдельно:

```
Finding
   │
   ├── reproduce
   ├── minimize
   ├── regression analysis
   └── fix verification
```

Triage не должен блокировать основной fuzzing process.

Все эти операции моделируются как `TriageTask` — конечные задачи, которые могут независимо планироваться и повторяться.

---

## 12. Task execution

Для finite operations используется отдельная abstraction:

```
TaskExecutor
```

Через неё выполняются:

- build;
- reproduce;
- minimize;
- corpus pruning;
- coverage calculation;
- regression;
- fix verification.

Архитектура не требует наличия отдельной message queue.

Если underlying execution platform уже умеет запускать, отслеживать и retry finite tasks, этого достаточно.

Отдельный queue backend может быть добавлен позднее без изменения domain model.

---

## 13. Observability

Метрики разделяются на два класса.

### Infrastructure metrics

- CPU;
- RAM;
- worker health;
- Instance state;
- resource availability.

Их предоставляет execution/infrastructure layer.

### Fuzzing metrics

Нормализуются Generic Runner независимо от fuzzing engine:

- exec/s;
- total executions;
- runtime;
- corpus size;
- coverage;
- coverage growth;
- unique Findings;
- last crash / last new coverage.

UI работает с нормализованной моделью и не должен напрямую разбирать output конкретного fuzzing engine.

---

## 14. Worker resources

Worker не является основной domain-сущностью кампании.

Execution backend предоставляет унифицированное представление доступных ресурсов:

```
WorkerResource

id
architecture
cpu
memory
availability
health
attributes
```

Регистрация, обнаружение отказов и изменение capacity относятся к infrastructure layer.

При исчезновении worker Control Plane работает не с восстановлением машины, а с восстановлением потерянных Instances.

---

## 15. Security

Fuzzing workloads считаются недоверенным кодом.

Domain layer описывает необходимые ограничения через execution policy:

```
resource limits
network restrictions
filesystem restrictions
privilege restrictions
runtime isolation requirements
```

Конкретный execution backend отвечает за преобразование этих требований в механизмы своей платформы.

Control Plane, storage credentials и другие чувствительные компоненты должны быть изолированы от fuzzing workload.

---

## 16. High-level module boundaries

Логически система разделяется на следующие модули:

```
project
    Projects, source revisions, targets

campaign
    Campaign configuration and lifecycle

build
    Build model and build execution

job
    Jobs and Instances

scheduler
    Priorities, quotas and admission decisions

execution
    ExecutionBackend and Instance lifecycle

corpus
    Corpus persistence and maintenance

finding
    Findings, normalization and deduplication

triage
    Reproduce, minimize, regression, fix verification

telemetry
    Fuzzing metrics and health

infrastructure
    Worker resources and cluster capacity

identity
    Users, teams and access control

integration
    CI/CD, webhooks and issue trackers
```

Эти модули являются **логическими boundaries**, а не обязательными отдельными сервисами.

Для первой версии предпочтителен modular monolith Control Plane плюс отдельный Generic Runner.

```
┌─────────────────────────────────────┐
│            Control Plane            │
│                                     │
│ project                             │
│ campaign                            │
│ build                               │
│ job                                 │
│ scheduler                           │
│ execution                           │
│ corpus                              │
│ finding                             │
│ triage                              │
│ telemetry                           │
│ API                                 │
└──────────────────┬──────────────────┘
                   │
                   ▼
            ports / adapters
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
     Cluster     Storage     External
     backend                 systems


        Generic Fuzz Runner
                 │
                 ▼
             Fuzz Target
```

Такое разделение позволяет избежать преждевременного перехода к микросервисной архитектуре, сохраняя при этом чёткие границы между функциональными областями.

---

## 17. Ключевые архитектурные решения

1. **Campaign является основной пользовательской единицей**, а не Pod или отдельный process.
2. **Job отделён от Instance**, чтобы логическая fuzzing-задача переживала restart, worker failure и migration.
3. **Desired-state reconciliation используется вместо imperative orchestration**, что упрощает recovery после отказов.
4. **Control Plane отвечает за fuzzing-level scheduling, infrastructure scheduler — за физическое размещение.**
5. **Build является immutable сущностью**, необходимой для воспроизводимости crashes и triage.
6. **OSS-Fuzz/runtime details скрываются за Generic Runner**, поэтому Control Plane не зависит от конкретного fuzzing engine.
7. **Corpus хранится независимо от Instance**, благодаря чему fuzzing может продолжаться после потери worker.
8. **Fuzzing и triage разделены:** crash processing не должен останавливать основной fuzzing loop.
9. **Persistence определяется через domain-oriented interfaces**, а не через конкретную БД, object storage или очередь.
10. **Execution backend является заменяемым adapter**, поэтому Kubernetes не проникает в core domain model.
11. **Finite tasks и continuous fuzzing workloads моделируются отдельно.**
12. **Модули являются логическими boundaries, а не набором обязательных микросервисов.**