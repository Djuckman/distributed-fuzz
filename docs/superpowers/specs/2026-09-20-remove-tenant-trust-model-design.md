# Убор multi-tenancy и isolation-модели; новая авторизация (design)

- Статус: согласован с пользователем в чате, ожидает финального review этого файла
- Дата: 2026-09-20
- Затрагиваемые документы: `Architecture.md`, `Implementation.md`, `README.md`, `docs/open-questions-devops-security.md`

## Контекст

Текущая архитектура (`Architecture.md`) построена на двух предположениях:
1. Запускаемый fuzzing-код (workload) недоверен.
2. Платформа обслуживает несколько взаимно недоверенных `Tenant`, отсюда — multi-tenancy, tenant-scoped изоляция данных и доступа, и классы исполнения `ExecutionPolicy` (`STANDARD`/`STRONG`/`DEDICATED`).

Оба предположения решено снять: код считается доверенным, платформа работает в одном trust domain без разделения на tenants. Изоляция workload (network policy, seccomp, capability drops, gVisor/Kata, привилегированные режимы CASR) больше не нужна ни в каком виде — не только на MVP, а как архитектурное решение.

При этом нужна возможность различать, кто запустил Campaign и с какого revision, и разграничивать права: не все пользователи могут запускать fuzzing с произвольного revision или управлять чужими запусками.

Обсуждение выявило, что `Project` — это существующая сущность, которая уже 1:1 соответствует понятию "project" в OSS-Fuzz (папка `projects/<name>/` с build-рецептом и fuzz targets одной цели). Путаницу создавал не сам термин, а его вложенность в `Tenant` (доменная организационная иерархия конкурировала со смыслом слова "project" из OSS-Fuzz). После удаления `Tenant` эта причина путаницы исчезает, и `Project` остаётся под тем же именем — без переименования в `Repository` (этот вариант был предложен и отклонён: слово "репозиторий" сталкивается сразу с двумя разными вещами — мета-репозиторием `google/oss-fuzz` и собственным git-репозиторием цели фаззинга — и создаёт путаницу хуже исходной).

## Решение

### 1. Trust model

Убирается весь раздел про trust boundaries между tenants и workload. Платформа — один trust domain. `Architecture.md` §4 переименовывается из "Trust boundaries и multi-tenancy" в "Access control" и описывает только авторизацию пользователей, а не изоляцию workload или данных между организациями.

### 2. Domain model — авторизация

Убираются: `Tenant`, `Team`.

`Project` остаётся top-level aggregate под тем же именем (владеет `SourceRevision`, `FuzzTarget`, `Campaign`, `Corpus` — без изменений относительно текущей модели, кроме отсутствия родителя `Tenant`).

Добавляются:

| Сущность | Назначение | Владелец |
|---|---|---|
| `Project` | Контекст исходного кода, targets, campaigns, corpus — top-level aggregate (без изменений, кроме отсутствия `Tenant` сверху) | Control Plane; корневой aggregate |
| `User` | Identity, подтверждённая `IdentityProvider` | — |
| `Group` | Набор `User` | Control Plane |
| `Membership` | Связь `User` ↔ `Group`, без роли | `Group` |
| `ProjectAccessGrant` | Связь `Group` ↔ `Project`, несёт `revisionAllowList` и `capabilities` | `Project` |

`ProjectAccessGrant` — единый механизм авторизации, объединяющий два измерения одним объектом:
- `revisionAllowList` — гибкий настраиваемый allow-list revision/ref-паттернов (например `master`, `release/*`); default policy при создании `Project` — разрешён только `master`;
- `capabilities` — набор control-plane действий: как минимум `RUN_CAMPAIGN`, `MANAGE_OTHERS_CAMPAIGN` (пауза/остановка чужих Campaign), `TRIAGE_FINDINGS`, `MANAGE_ACCESS` (изменение самих grants).

Создание `Campaign` на `SourceRevision` X в `Project` P требует: пользователь состоит в `Group`, у которой есть `ProjectAccessGrant` на P с capability `RUN_CAMPAIGN` и X, попадающим в `revisionAllowList` этого grant. Другие capabilities проверяются аналогично при соответствующих операциях.

Ownership chain: `Project → Campaign → FuzzJob → ExecutionAttempt` (без `Tenant` сверху; `FuzzTarget` остаётся собственностью `Project`, `Campaign` только ссылается на подмножество существующих `FuzzTarget`, не владеет ими — это не меняется).

### 2.1. Запуск нескольких Project одновременно — `CampaignBatch`

`Campaign` остаётся строго 1:1 с `Project`: один `SourceRevision`, один build recipe, одна `StopPolicy` на Campaign — это существующее ограничение не меняется и не должно меняться (у разных OSS-Fuzz project разные репозитории и ревизии, "один SourceRevision на несколько Project" не имеет смысла).

Чтобы запустить несколько OSS-Fuzz project через Control Plane одновременно, добавляется новая сущность:

| Сущность | Назначение | Владелец |
|---|---|---|
| `CampaignBatch` | Необязательная группировка нескольких `Campaign` (в т.ч. из разных `Project`) для совместного просмотра статуса/находок | Control Plane |

Ограничения `CampaignBatch`:
- не владеет `FuzzJob` и не участвует в state machine `Campaign` — каждая входящая `Campaign` живёт своим независимым lifecycle;
- не поддерживает каскадные команды (pause/stop всего batch одной командой) — это осознанно вынесено за скоуп v1;
- `Campaign` состоит в `CampaignBatch` необязательно (0..1); создание `CampaignBatch` — client-side/API-удобство поверх независимого создания нескольких `Campaign`, а не новый уровень authoritative ownership.

### 3. Security requirements

Из `Architecture.md` §15 и `Implementation.md` §16 убирается целиком:
- `ExecutionPolicy` и классы `STANDARD`/`STRONG`/`DEDICATED`;
- network policy между workload разных tenants/projects, deny-by-default segmentation;
- seccomp profile, dropped capabilities, read-only rootfs, RuntimeClass-требования как обязательные;
- оценка gVisor/Kata как isolation-кандидатов;
- отдельный security-review статус для CASR GDB/`ptrace`-режима как isolation-риска — убирается полностью, режим используется как любой другой privileged-инструмент без специального review.

Остаётся:
- аутентификация (`IdentityProvider`) и авторизация через `Group`/`ProjectAccessGrant`;
- immutable audit trail для security-sensitive изменений (изменение `ProjectAccessGrant`, запуск Campaign);
- запрет передачи Control Plane credentials в workload — остаётся архитектурным инвариантом независимо от доверия к коду.

### 4. Ресурсы и observability

- `ResourceLease` теряет `tenantId`; `projectId` остаётся единицей масштабирования квот/fair-sharing (без изменений относительно текущей модели).
- Correlation identifiers в Observability (`Architecture.md` §16) — `tenantId` убирается, `projectId` остаётся.

### 5. Инварианты (`Architecture.md` §19)

- Убрать инвариант "ownership chain не пересекает tenant boundary".
- Переформулировать "Каждый Project принадлежит ровно одному Tenant..." → "Каждый доступ к `Project` авторизуется через `ProjectAccessGrant` соответствующей `Group`".
- В инварианте про credentials убрать часть про `ExecutionPolicy`, оставить только запрет credentials.
- Убрать все формулировки про `Tenant`; список ренумеровать при правке.

## Изменения по файлам

### `Architecture.md`
- §2 Цели — убрать "tenant isolation" из списка целей.
- §3 Принципы — убрать принцип "tenant context обязателен на каждой границе"; ренумеровать.
- §4 — переименовать в "Access control", переписать под User/Group/Membership/ProjectAccessGrant.
- §5 Domain model table — убрать строки Tenant/Team, добавить User/Group/ProjectAccessGrant, строка Project — убрать владельца `Tenant`.
- §6–§14 — убрать формулировки про изоляцию tenant-данных (например §9 про изоляцию artifacts/build cache разных tenants).
- §15 Security requirements — переписать согласно разделу 3.
- §16 Observability — `tenantId` убрать из correlation identifiers, `projectId` оставить.
- §17 Ports — снять "tenant-scoped" формулировки у Repositories/AuditStore.
- §19 Инварианты — согласно разделу 5.

### `Implementation.md`
- §2 Критические отображения — убрать tenant/isolation-специфичные пункты.
- §8 ResourceAdmission — "tenant-oriented LocalQueue" → project-oriented; "клэмпит по tenant policy" → "по project policy".
- §12 Findings/CASR — убрать абзац про обязательный отдельный security review GDB-режима как isolation-риска.
- §15 Identity и authorization — переписать полностью под User/Group/Membership/ProjectAccessGrant; обновить PoC-критерии (убрать cross-tenant проверки, добавить проверку enforcement `revisionAllowList`).
- §16 Workload isolation — заменить коротким абзацем: MVP запускает workload как обычный Pod со стандартными resource limits, без специального isolation profile.
- §18 MVP roadmap — убрать пункт про trust-boundary controls класса `STANDARD`.
- §19–20 Матрица соответствия — обновить строки инвариантов 2/3/19.
- §21 PoC-реестр — убрать `POC-ISOLATION-001` целиком; переименовать tenant policy → project policy в `POC-RESPROFILE-001`.
- §22 Референсы — убрать ссылки на gVisor/Kata Containers.

### `README.md`
- FAQ №2 (сравнение с ClusterFuzz) — убрать "multi-tenant изоляцию" из списка причин собственного Control Plane.
- FAQ №4 — "по tenant policy" → "по project policy".
- Таблица «Предлагаемая реализация» — строки Artifacts/Identity: убрать tenant authorization/tenant policy формулировки.
- Mermaid-диаграммы (Уровень 1-3) — "в очереди tenant" → "в очереди project".
- Абзац про RBAC service account — "свой namespace на tenant/project" → "свой namespace на Project" без изоляционной аргументации.
- Новая секция «Иерархия сущностей и доступа» (после FAQ, до «Как система получает и выделяет ресурсы») с двумя диаграммами:
  1. доменная иерархия: `Project` (пример — два разных OSS-Fuzz project) → `Campaign` (1:1 с Project) → `FuzzJob` → `ExecutionAttempt`, плюс `CampaignBatch` как опциональная группировка Campaign из разных Project только для просмотра;
  2. модель авторизации: `User` → (`Membership`) → `Group` → (`ProjectAccessGrant`: `revisionAllowList` + `capabilities`) → `Project`.

### `docs/open-questions-devops-security.md`
- Убрать пункт «Онбординг Tenant» (неактуален без Tenant).
- Убрать раздел «Изоляция и privileged-режимы» целиком (решение принято окончательно, а не как MVP-упрощение).
- Остальные разделы (autoscaling vs Kueue, владение жизненным циклом Kueue/Kubernetes, Supply chain) не меняются.

## Вне scope

- Lifecycle/state machines Campaign/FuzzJob/Task/Build/ExecutionAttempt не меняются по существу — только удаление ссылок на `Tenant` там, где они встречаются.
- `docs/adr/README.md` не меняется — процесс ADR не завязан на Tenant.
- Названия и содержание capabilities внутри `ProjectAccessGrant` не фиксируются жёстко списком навсегда — минимальный набор для MVP: `RUN_CAMPAIGN`, `MANAGE_OTHERS_CAMPAIGN`, `TRIAGE_FINDINGS`, `MANAGE_ACCESS`; расширение — предмет будущего ADR, а не блокер этого изменения.
