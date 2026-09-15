## 1. Purpose

Этот документ определяет, какие части fuzzing platform необходимо реализовать самостоятельно, а для каких следует использовать или адаптировать существующие open-source компоненты.

Основной принцип:

> Собственная реализация должна содержать domain-specific orchestration и product logic. Общие задачи Kubernetes orchestration, fuzzing runtime, crash analysis, build execution и infrastructure management по возможности должны переиспользовать существующие mature implementations.

Предпочтительный порядок выбора:

```
Use as dependency/runtime
        ↓
Wrap with adapter
        ↓
Adapt/port isolated logic
        ↓
Implement from scratch
```

Fork существующего большого проекта рассматривается как нежелательный вариант.

---

## 2. Summary

|Module|Strategy|Candidate|Что переиспользуется|
|---|---|---|---|
|Project / Campaign domain|**Implement**|—|Собственная domain model|
|Job / Instance lifecycle|**Implement**|—|Собственная lifecycle semantics|
|Reconciliation|**Reuse**|`kubernetes-sigs/controller-runtime`|Controller infrastructure, watches, retries, reconciliation|
|Resource admission / quotas|**Reuse**|`kubernetes-sigs/kueue`|Queues, priorities, quotas, fair sharing, preemption|
|Execution backend|**Adapt**|Kubernetes API / controller-runtime|Pod/Job lifecycle, Node information|
|Build|**Reuse + Adapt**|`google/oss-fuzz`, `moby/buildkit`|Fuzzer build environment, OCI build/cache|
|Generic Fuzz Runner|**Adapt**|OSS-Fuzz runtime|Target execution semantics|
|Corpus persistence|**Reuse**|`google/go-cloud/blob` or equivalent|Portable blob storage access|
|Corpus merge/pruning|**Reuse + Adapt**|OSS-Fuzz / ClusterFuzzLite|Engine-specific corpus minimization|
|Finding normalization|**Reuse**|`ispras/casr`|Stack parsing, crash reports, severity|
|Finding deduplication|**Reuse / Adapt**|CASR, ClusterFuzz reference|Crash deduplication and clustering|
|Reproduce|**Reuse**|OSS-Fuzz|Reproducer execution|
|Minimize|**Adapt**|ClusterFuzz + fuzz engine|Testcase minimization workflow|
|Regression analysis|**Adapt**|ClusterFuzz|Revision bisection workflow|
|Fix verification|**Adapt**|ClusterFuzz|Reproduce-on-new-build workflow|
|Coverage|**Reuse**|OSS-Fuzz|Coverage builds and reports|
|Coverage aggregation|**Optional Adapt**|FuzzBench|Coverage statistics/processing|
|Fuzz metrics|**Implement thin adapter**|Prometheus client ecosystem|Metrics transport/export|
|Infrastructure metrics|**Reuse**|Kubernetes monitoring ecosystem|Node/Pod/CPU/RAM health|
|Worker discovery|**Reuse**|Kubernetes Node API|Capacity, architecture, health|
|Finite task execution|**Reuse**|Kubernetes Job|Execution/retry/status|
|Workload isolation|**Reuse**|gVisor / Kata Containers|Sandbox boundary|
|Authentication|**Reuse**|OIDC infrastructure|Authentication|
|Authorization|**Implement initially**|OpenFGA optional|Project/team permissions|
|External integrations|**Implement adapters**|Existing APIs/SDKs|Jira/GitHub/GitLab/Webhooks|
|UI / API|**Implement**|—|Product-specific interface|

---

# 3. Domain Model

## Project / Campaign / Job / Instance

**Strategy: Implement**

These concepts represent the core domain of the platform and should not be inherited from ClusterFuzz or Kubernetes.

Required entities include:

```
Project
SourceRevision
FuzzTarget
Campaign
Build
Job
Instance
Corpus
Finding
TriageTask
```

Existing systems use different semantics for these concepts. Adopting their internal models would unnecessarily couple the platform to their execution architecture.

In particular:

```
Job != Kubernetes Job
Job != ClusterFuzz Job
Instance != Kubernetes Pod
```

Kubernetes and other systems should remain implementation details behind adapters.

---

# 4. Control Plane Reconciliation

## controller-runtime

**Strategy: Reuse as dependency**

Repository:

`kubernetes-sigs/controller-runtime`

controller-runtime provides Go libraries for building Kubernetes controllers and is used by Kubebuilder and Operator SDK. citeturn843416view1

It should provide infrastructure for:

- watches;
- reconcile loops;
- work queues;
- retries;
- cached Kubernetes state;
- leader election;
- controller lifecycle.

The platform should implement only the domain-specific reconciliation:

```
Campaign desired state
        +
actual Instances
        ↓
required actions
```

Example:

```
desired:
    Job A → RUNNING

actual:
    Instance A → LOST

result:
    create replacement Instance
```

### Recommendation

Use controller-runtime rather than implementing an internal reconciliation framework.

**Reuse level: high.**

---

# 5. Scheduling, Priority and Quotas

## Kueue

**Strategy: Reuse as infrastructure component**

Repository:

`kubernetes-sigs/kueue`

Kueue is a Kubernetes-native job-level admission system. It supports queueing, priorities, resource management, fair sharing, cohorts and preemption, and integrates with regular Jobs, Pods and Pod Groups. citeturn840603search3

This overlaps strongly with the platform requirements for:

```
project quotas
team quotas
campaign priority
resource admission
fair sharing
preemption
```

The division of responsibilities should be:

```
Fuzzing Platform
      │
      │ "Job requires 8 CPU / 16 GiB"
      ▼
ResourceAdmission
      │
      ▼
Kueue
      │
      │ admitted
      ▼
Kubernetes Scheduler
      │
      ▼
Node
```

The fuzzing platform remains responsible for deciding **what logical workload should exist**.

Kueue determines **whether sufficient quota exists to admit it**.

The Kubernetes scheduler determines **where the workload runs**.

### Do not expose Kueue in the domain model

Use an internal abstraction such as:

```
ResourceAdmission
    Submit()
    Suspend()
    Status()
```

with Kueue as the first implementation.

### Alternative: Volcano

Volcano is useful for HPC/gang-scheduled workloads but is unnecessarily complex for the initial model:

```
1 Job → 1 active Instance
```

It should be reconsidered only if simultaneous admission of multiple tightly coupled workers becomes a requirement.

**Reuse level: very high.**

---

# 6. Execution Backend

## Kubernetes

**Strategy: Implement thin adapter**

The platform requires:

```
ExecutionBackend

Start()
Stop()
Status()
Capacity()
```

The Kubernetes implementation maps:

```
Instance → Pod
TriageTask → Job
WorkerResource → Node
```

Kubernetes already solves:

- Pod scheduling;
- restart primitives;
- node health;
- resource requests/limits;
- architecture constraints;
- affinity;
- taints/tolerations;
- workload termination.

The platform should not implement equivalents.

### What remains custom

The adapter must convert domain objects:

```
InstanceSpec
```

into Kubernetes resources and translate Kubernetes states back into:

```
PENDING
STARTING
RUNNING
FAILED
LOST
TERMINATED
```

**Reuse level: high. Custom code: small/medium.**

---

# 7. Build Execution

## OSS-Fuzz

**Strategy: Reuse runtime/build conventions**

Repository:

`google/oss-fuzz`

OSS-Fuzz already provides operations for:

```
build_fuzzers
check_build
run_fuzzer
coverage
reproduce
```

through its helper infrastructure. citeturn406201search0turn406201search3

The current system already exposes OSS-Fuzz targets through a unified interface. That interface should remain the boundary consumed by the control plane.

The control plane should not learn OSS-Fuzz implementation details.

```
BuildExecutor
      │
      ▼
OSS-Fuzz adapter
      │
      ▼
existing unified build interface
```

## BuildKit

**Strategy: Reuse if OCI builds are performed inside the cluster**

Repository:

`moby/buildkit`

BuildKit provides concurrent builds, caching, cache import/export, distributed workers, multiple output formats and rootless execution. citeturn840603search1turn840603search2

It is preferable to implementing custom Docker-in-Docker build infrastructure.

Suggested architecture:

```
BuildTask
   ↓
BuildExecutor
   ↓
OSS-Fuzz build
   ↓
BuildKit
   ↓
OCI artifact / image
```

### Build artifact

The platform should own the metadata:

```
Build {
    sourceRevision
    architecture
    engine
    sanitizer
    buildOptions
    artifactRef
}
```

but not the mechanics of compilation/container image creation.

**Reuse level: high.**

---

# 8. Generic Fuzz Runner

**Strategy: Implement thin wrapper around existing OSS-Fuzz runtime**

OSS-Fuzz already defines how fuzz targets are executed in its fuzzing environment and provides `run_fuzzer` support with an external corpus directory. citeturn406201search0turn406201search3

The Generic Runner should therefore remain small.

Responsibilities:

```
restore corpus
     ↓
prepare runtime
     ↓
invoke unified OSS-Fuzz target
     ↓
collect metrics
     ↓
collect Findings
     ↓
checkpoint corpus
     ↓
graceful shutdown
```

It should **not** reimplement:

- libFuzzer invocation logic;
    
- AFL++ invocation logic;
    
- sanitizer setup;
    
- OSS-Fuzz environment preparation.
    

The runner should instead normalize engine-specific output into the platform API.

**Reuse level: medium/high. Custom code: medium.**

---

# 9. Corpus Persistence

## Storage abstraction

**Strategy: Implement domain interface, reuse storage implementation**

Domain interface:

```
CorpusStore

Restore()
Checkpoint()
Delete()
```

The implementation should use a portable storage library where possible.

For a Go control plane, `google/go-cloud/blob` is a suitable candidate. Its blob package provides a portable storage API with provider-specific drivers rather than requiring storage-specific code in the application. citeturn406201search10turn406201search11

Architecture:

```
CorpusStore
     │
     ▼
Blob adapter
     │
 ┌───┼────┐
 ▼   ▼    ▼
S3  GCS  filesystem
```

The platform should keep its own `CorpusStore` interface even if Go CDK is used underneath it.

This prevents the external library from becoming part of the domain API.

**Reuse level: high.**

---

# 10. Corpus Merge and Pruning

**Strategy: Reuse engine-specific algorithms**

Corpus minimization is fuzz-engine-specific and should not be recreated in the control plane.

ClusterFuzzLite already treats corpus pruning as a standalone periodic operation and describes it as removing redundant corpus elements while retaining coverage. citeturn840603search0

Model it as:

```
CorpusPruneTask
       ↓
TaskExecutor
       ↓
engine / OSS-Fuzz pruning
       ↓
new CorpusSnapshot
```

For the initial:

```
1 Job → 1 Instance
```

model, live distributed corpus synchronization is unnecessary.

The control plane only needs orchestration of:

```
Restore
Checkpoint
Prune
```

Future multi-instance execution can add:

```
Merge
```

without changing the current lifecycle.

**Reuse level: algorithm high; orchestration custom.**

---

# 11. Finding Normalization

## CASR

**Strategy: Reuse**

Repository:

`ispras/casr`

CASR provides crash report collection, stacktrace parsing, severity estimation, deduplication and clustering. It supports sanitizer reports including ASAN, MSAN and UBSAN and exposes the functionality through LibCASR and command-line tools. citeturn406201search2

Suggested pipeline:

```
raw testcase + stderr
        ↓
       CASR
        ↓
normalized crash report
        ↓
      Finding
```

The platform's `Finding` remains its own domain entity.

CASR should act as a normalization engine.

### Integration approaches

Preferred initial integration:

```
CASR CLI / dedicated task container
```

rather than linking CASR implementation directly into the control plane.

This provides language/runtime isolation and makes later replacement simpler.

**Reuse level: very high.**

---

# 12. Finding Deduplication

**Strategy: Reuse initially, keep replaceable**

CASR includes crash deduplication and clustering based on stacktrace analysis. citeturn406201search2

ClusterFuzz also provides production crash deduplication and can be used as an additional reference implementation. ClusterFuzz explicitly lists accurate crash deduplication among its core functionality. citeturn843416view0

Suggested abstraction:

```
FindingNormalizer
FindingDeduplicator
```

Initial implementation:

```
CASR
```

Possible later extension:

```
CASR
 +
custom project-specific rules
```

Do not encode deduplication as a persistence trick such as:

```
UNIQUE(stack_hash)
```

because deduplication semantics will likely evolve.

**Reuse level: high.**

---

# 13. Reproduce

## OSS-Fuzz

**Strategy: Reuse**

OSS-Fuzz already provides a `reproduce` operation alongside its build/run functionality. citeturn406201search0

The platform only needs orchestration:

```
Finding
   ↓
ReproduceTask
   ↓
TaskExecutor
   ↓
OSS-Fuzz runtime
   ↓
reproduced / not reproduced
```

The exact same Build should be used where possible.

**Reuse level: very high.**

---

# 14. Testcase Minimization

## ClusterFuzz

**Strategy: Adapt workflow, reuse engine minimizer**

Repository:

`google/clusterfuzz`

ClusterFuzz already supports testcase minimization as a first-class fuzzing operation. citeturn843416view0

The complete ClusterFuzz task implementation should not be imported because it is tied to ClusterFuzz's own infrastructure model.

Instead reuse:

- minimization semantics;
    
- engine-specific invocation;
    
- timeout handling;
    
- validation approach;
    
- reproduction before/after minimization.
    

Platform workflow:

```
Finding
   ↓
MinimizeTask
   ↓
reproduce original
   ↓
engine minimization
   ↓
reproduce minimized testcase
   ↓
store minimized artifact
```

**Reuse level: medium. Adapt rather than fork.**

---

# 15. Regression Analysis

## ClusterFuzz

**Strategy: Adapt algorithm/workflow**

ClusterFuzz supports regression finding using revision bisection. citeturn843416view0

This functionality maps naturally to existing platform abstractions:

```
RegressionTask
      │
      ├── BuildExecutor
      │
      └── Reproducer
```

Generic workflow:

```
known good revision
known bad revision
        ↓
select midpoint
        ↓
build revision
        ↓
reproduce testcase
        ↓
good / bad
        ↓
repeat
```

The useful part to reuse is the algorithm and edge-case handling.

The following ClusterFuzz-specific pieces should **not** be reused:

```
ClusterFuzz queues
ClusterFuzz bot model
ClusterFuzz datastore
ClusterFuzz build infrastructure
```

**Reuse level: medium.**

---

# 16. Fix Verification

## ClusterFuzz progression model

**Strategy: Adapt**

ClusterFuzz has an equivalent workflow for checking whether an existing crash continues to reproduce on newer builds.

Platform model:

```
Finding
   │
new Build
   ↓
FixVerificationTask
   ↓
Reproduce(testcase, new build)
   │
 ┌─┴──────────┐
 ▼            ▼
still fails   fixed
```

The workflow itself is simple but ClusterFuzz can be used as reference for handling:

- flaky reproduction;
    
- build failures;
    
- transitions between fixed/unfixed states;
    
- repeated verification.
    

**Reuse level: medium.**

---

# 17. Coverage

## OSS-Fuzz

**Strategy: Reuse**

OSS-Fuzz already supports dedicated coverage builds and coverage generation. citeturn406201search0turn406201search3

ClusterFuzzLite also models coverage generation as a periodic standalone task using the latest corpus. citeturn840603search0

Platform workflow:

```
CorpusSnapshot
      +
SourceRevision
      ↓
CoverageTask
      ↓
OSS-Fuzz coverage
      ↓
CoverageArtifact
```

Coverage should not run synchronously with the main fuzzing loop.

## FuzzBench

**Strategy: Optional source of processing logic**

Repository:

`google/fuzzbench`

FuzzBench contains reusable coverage data analysis utilities, including its `analysis/coverage_data_utils.py`. citeturn406201search12

It should not be used as the orchestration backend because its experiment model solves a different problem.

Possible reuse:

```
coverage data aggregation
statistics
report processing
```

**Reuse level: optional.**

---

# 18. Fuzzing Metrics

**Strategy: Implement normalization only**

Infrastructure for exposing metrics should use the standard monitoring ecosystem.

The only significant fuzzing-specific code required is translation:

```
libFuzzer output
AFL++ output
other engine output
        ↓
normalized metrics
```

Example common metrics:

```
fuzz_execs_total
fuzz_execs_per_second
fuzz_corpus_items
fuzz_coverage
fuzz_findings_total
fuzz_last_new_coverage_timestamp
```

The Generic Runner should expose these independently of the underlying engine.

The platform should not implement a custom time-series database or metrics transport.

**Custom code: small/medium.**

---

# 19. Infrastructure Metrics and Worker Discovery

## Kubernetes

**Strategy: Reuse**

For the Kubernetes backend, Node and Pod state are already available through the Kubernetes API.

Worker information can be derived from:

```
Node.status.capacity
Node.status.allocatable
Node.status.conditions
Node.labels
Node.taints
architecture
```

Therefore no independent worker registration protocol is required.

Mapping:

```
Kubernetes Node
       ↓
WorkerResource
```

Infrastructure monitoring should use the normal Kubernetes monitoring stack rather than duplicating it in the fuzzing platform.

The fuzzing platform should store only the domain view required by scheduling/UI.

**Reuse level: very high.**

---

# 20. Finite Task Execution

## Kubernetes Job

**Strategy: Reuse**

Finite operations include:

```
Build
Reproduce
Minimize
Regression step
Fix verification
Coverage
Corpus pruning
```

They naturally map to finite execution objects such as Kubernetes Jobs.

Therefore no message broker is required for the initial architecture.

```
TriageTask
    ↓
TaskExecutor
    ↓
Kubernetes Job
```

The domain interface remains:

```
TaskExecutor

Submit()
Cancel()
Status()
```

If scale or operational requirements later justify a dedicated queue, another implementation can be added without changing triage logic.

### Explicitly avoid initially

```
Kafka
RabbitMQ
NATS
custom distributed queue
```

unless a concrete requirement appears.

**Reuse level: very high.**

---

# 21. Workload Isolation

Fuzz targets should be considered untrusted workloads.

## gVisor

**Strategy: Evaluate as first sandbox option**

Repository:

`google/gvisor`

gVisor integrates with Kubernetes and can be selected for Pods through a `RuntimeClass`. citeturn406201search5

This makes it possible to run:

```
control-plane workloads → normal runtime

fuzzing workloads       → gVisor runtime
```

without changing the control-plane architecture.

## Kata Containers

**Strategy: Alternative for stronger VM isolation**

Repository:

`kata-containers/kata-containers`

Kata also integrates with Kubernetes through `RuntimeClass` and runs workloads inside lightweight virtual machines. citeturn406201search1turn406201search6

Potential policy:

```
normal fuzz target
      ↓
gVisor

high-risk / incompatible target
      ↓
Kata / dedicated worker
```

The domain should expose only:

```
ExecutionPolicy.isolation
```

and let the Kubernetes adapter select the appropriate runtime.

**Reuse level: very high.**

---

# 22. Authentication and Authorization

## Authentication

**Strategy: Reuse existing OIDC infrastructure**

Do not implement a separate password/authentication system.

The API should trust identities supplied by an external identity provider / authentication layer.

## Authorization

**Strategy: Implement simple domain RBAC initially**

Initial model is likely sufficient:

```
User
Team
Project

viewer
maintainer
admin
```

Authorization rules are domain-specific and should remain in the control plane.

A system such as OpenFGA should only be introduced if requirements evolve toward complex relationship-based authorization.

**Custom code: small initially.**

---

# 23. External Integrations

**Strategy: Implement thin adapters**

Required integrations may include:

```
Jira
GitHub
GitLab
CI/CD
Webhooks
```

ClusterFuzz supports automated bug filing and integrations including Jira and can be used as a behavioral reference. citeturn843416view0

However, its integration layer should not be imported wholesale because it is coupled to ClusterFuzz's Finding/testcase model.

Define:

```
FindingSink

Create()
Update()
Close()
```

and implement separate adapters.

Prefer official client libraries where available.

**Custom code: medium.**

---

# 24. Components That Should Not Be Reused as the Platform Core

Several existing projects are valuable references but should not become the architectural foundation.

## ClusterFuzz

Use for:

```
minimization semantics
regression algorithms
fix verification
crash-processing reference
```

Do not reuse:

```
bot infrastructure
task queues
datastore model
cloud deployment architecture
job model
web application architecture
```

ClusterFuzz is a complete fuzzing infrastructure and therefore carries architectural decisions that the new control plane intentionally replaces. It includes crash deduplication, minimization, regression finding and issue-tracker automation, making it particularly valuable as a donor/reference project. citeturn843416view0

## ClusterFuzzLite

Use for:

```
CI-oriented fuzzing flows
corpus pruning
coverage task patterns
artifact layout ideas
```

Do not use it as the campaign orchestrator.

Its documented workflows already model pruning and coverage as independent periodic operations, which matches the proposed Task model. citeturn840603search0

## FuzzBench

Use for:

```
coverage processing
fuzzer statistics
engine integration ideas
```

Do not reuse:

```
experiment orchestration
dispatcher architecture
database model
```

The platform is not a fuzzer benchmarking system.

---

# 25. Recommended Integration Boundary

The resulting architecture should look approximately like:

```
                    OUR CONTROL PLANE
┌────────────────────────────────────────────────┐
│                                                │
│ Project / Campaign / Job / Instance            │
│ Lifecycle                                      │
│ Finding / Triage domain                        │
│ API / authorization / integrations             │
│                                                │
│ ---------------------------------------------- │
│ Ports                                          │
│                                                │
│ ExecutionBackend                               │
│ ResourceAdmission                              │
│ BuildExecutor                                  │
│ TaskExecutor                                   │
│ CorpusStore                                    │
│ ArtifactStore                                  │
│ FindingNormalizer                              │
└───────────────┬────────────────────────────────┘
                │
             adapters
                │
    ┌───────────┼─────────────────────────────────┐
    │           │                 │               │
    ▼           ▼                 ▼               ▼

Kubernetes     Kueue            OSS-Fuzz         CASR
    │                             │
    │                             ├── build
    │                             ├── fuzz
    │                             ├── reproduce
    │                             └── coverage
    │
    ├── Pods
    ├── Jobs
    └── Nodes

        BuildKit       Blob Storage       gVisor/Kata
```

This keeps third-party implementations below stable domain interfaces.

---

# 26. Implementation Priorities

## MVP

The initial version should require only:

|Area|Implementation|
|---|---|
|Domain|Custom|
|Campaign lifecycle|Custom|
|Job/Instance reconciliation|Custom + controller-runtime|
|Execution|Kubernetes adapter|
|Admission/quotas|Kueue|
|Fuzz execution|Existing unified OSS-Fuzz interface|
|Corpus|CorpusStore + blob implementation|
|Findings|CASR integration|
|Reproduce|OSS-Fuzz|
|Metrics|Generic Runner adapter|
|Finite tasks|Kubernetes Jobs|

This is sufficient for:

```
create Campaign
      ↓
build
      ↓
queue/admit
      ↓
run fuzz target
      ↓
checkpoint corpus
      ↓
collect Finding
      ↓
reproduce
```

## Phase 2

Add:

```
testcase minimization
coverage tasks
corpus pruning
issue tracker integrations
stronger workload isolation
```

## Phase 3

Add:

```
regression bisection
fix verification
advanced deduplication
fair-sharing tuning
multi-cluster execution
multiple Instances per Job
```

Multi-instance execution should not influence the MVP implementation beyond retaining the existing `Job != Instance` model.

---

# 27. What Must Actually Be Built

After reuse of existing components, the genuinely platform-specific implementation is reduced mainly to:

### 1. Domain model

```
Project
Campaign
Job
Instance
Build
Finding
TriageTask
```

### 2. Lifecycle logic

```
start
stop
pause
resume
restart
recovery
configuration updates
```

### 3. Reconciliation

```
desired domain state
        ↓
actual infrastructure state
        ↓
required operations
```

### 4. Adapters

```
Kubernetes
Kueue
OSS-Fuzz
CASR
Storage
```

### 5. Product layer

```
REST/gRPC API
UI
RBAC
CI/CD integration
notifications
issue trackers
```

Everything else should be treated first as a reuse/adaptation problem rather than a new implementation problem.

---

# 28. Repository Shortlist

The following repositories are the primary sources for implementation reuse or architectural reference:

|Repository|Purpose|
|---|---|
|`google/oss-fuzz`|Build, fuzz execution, reproduce, coverage|
|`google/clusterfuzz`|Minimize, regression, fix verification, crash-management reference|
|`google/clusterfuzzlite`|Corpus pruning and CI-oriented finite workflows|
|`ispras/casr`|Crash parsing, reports, deduplication and clustering|
|`kubernetes-sigs/controller-runtime`|Controller/reconciliation infrastructure|
|`kubernetes-sigs/kueue`|Resource admission, quotas, priorities and fair sharing|
|`moby/buildkit`|OCI build execution and build caching|
|`google/go-cloud`|Portable blob-storage implementation|
|`google/fuzzbench`|Coverage/statistics processing reference|
|`google/gvisor`|Sandboxed container execution|
|`kata-containers/kata-containers`|VM-backed sandboxed execution|

Before copying source code rather than consuming a project as a dependency/tool, its current license and dependency licenses must be reviewed separately.