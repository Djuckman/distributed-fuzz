# Горизонтальное масштабирование FuzzJob (N параллельных instances с sync corpus)

## 1. Статус и область действия

`PROPOSED`. Дополняет Architecture.md и Implementation.md: вводит возможность запускать один
`FuzzTarget` как N параллельных независимых fuzzing-инстансов с периодической синхронизацией
corpus между собой (модель ClusterFuzz), вместо текущего жёсткого маппинга «один FuzzJob — один
активный ExecutionAttempt».

Документ описывает только domain-модель и lifecycle-изменения. Конкретные PoC, provider corpus
sync и Kueue-интеграция для elastic re-admission остаются `PROPOSED`/открытыми, как и остальные
адаптеры платформы.

## 2. Мотивация

Сейчас (Architecture.md §7 «FuzzJob») каждый continuous `FuzzJob` управляет ровно одной активной
попыткой (`ExecutionAttempt`): retry, recovery после потери пода, pause/resume — всё это работает
в терминах «текущая попытка». Один target = один под.

Для повышения throughput фаззинга нужен горизонтальный скейлинг: N независимых подов, каждый
фаззит тот же target самостоятельно (без синхронной координации между собой), и периодически
обмениваются найденными интересными входами через общий corpus, как это делает ClusterFuzz.
Простое увеличение ресурсов одного пода такого эффекта не даёт — coverage exploration
выигрывает от разнообразия independent random walks, а не от одного более быстрого воркера.

## 3. Domain model: новая сущность FuzzInstance

Между `FuzzJob` и `ExecutionAttempt` вводится `FuzzInstance`:

```
Campaign → FuzzJob → FuzzInstance (1..N) → ExecutionAttempt
```

`FuzzInstance` перенимает весь self-healing state machine, которым сегодня владеет `FuzzJob`
(Architecture.md:139-152) практически без изменений: собственный `ResourceLease`, собственный
`ExecutionAttempt`, собственные RECOVERING/PAUSING/STOPPING переходы. Это переиспользование
существующего паттерна — `Task` точно так же напрямую владеет `ExecutionAttempt`
(Architecture.md:186-199) — а не новый механизм.

`FuzzJob` упрощается до двух обязанностей:

1. **Reconciliation желаемого числа instances** в пределах `[min, max]` (задаётся в конфигурации
   target/Campaign) исходя из доступной quota — не только замена потерь, но и оппортунистическое
   добавление/снятие instances при изменении свободной quota (см. §5).
2. **Агрегация** фазы и coverage epoch по своим `FuzzInstance` (см. §6).

Рассмотренные и отклонённые альтернативы:

- **Раздуть сам `FuzzJob`** до управления N попытками напрямую — отклонено: фаза `FuzzJob`
  превратилась бы в комбинаторную функцию состояний N попыток, а per-instance recovery/checkpoint
  пришлось бы хранить как неявную структуру внутри одной сущности — фактически reinvent той же
  `FuzzInstance`, но без явных границ.
- **N независимых `FuzzJob` на один target с общим Corpus**, без новой сущности — отклонено:
  `noCoverageGrowthFor` и StopPolicy уже считаются per `FuzzJob` (инвариант 13); с N отдельными
  `FuzzJob` они рассинхронизируются, и всё равно потребовалась бы группирующая сущность для
  elastic-решений — проблема просто переименовывается. Также ломает текущий контракт «Campaign
  требует ровно один FuzzJob на target» (Architecture.md:116).

## 4. Lifecycle: FuzzJob

Заменяет текущую таблицу Architecture.md:139-152 для continuous target со scale-out:

| Текущая фаза | Условие | Следующая фаза | Обязательный эффект |
|---|---|---|---|
| `QUEUED` | admission range подтверждён (доступен хотя бы `min`) | `SCALING` | Reconciler вычисляет desired count в `[min, max]` по доступной quota, создаёт desired count `FuzzInstance` |
| `SCALING` | ≥1 `FuzzInstance` перешёл в RUNNING | `RUNNING` | Начать учёт active time агрегированной coverage epoch по wall-clock FuzzJob |
| `RUNNING` | периодический reconciliation tick | `RUNNING` (self-loop) | Пересчитать desired count по текущей quota/fair-share; создать недостающие либо инициировать graceful stop лишних `FuzzInstance`, не выходя за `[min, max]` |
| `RUNNING` | `FuzzInstance` потерян (LOST/preemption/node loss) | `RUNNING` (self-loop) | Запросить replacement `FuzzInstance` независимо от остальных — как раньше делал сам FuzzJob для единственной попытки |
| `RUNNING` | StopPolicy выполнен на агрегированной coverage epoch | `COMPLETED` | Остановить все текущие `FuzzInstance` с checkpoint best effort до общего deadline; зафиксировать итоговую coverage epoch и terminal event |
| `QUEUED`, `SCALING`, `RUNNING` | pause/stop Campaign либо невосстановимая ошибка | `PAUSING`/`STOPPING` | Отменить admission на новые instances; применить checkpoint/stop handshake ко всем текущим `FuzzInstance` |
| `PAUSING` | все `FuzzInstance` terminal/остановлены | `PAUSED` | Исключить период pause из active time; сохранить последний успешно опубликованный canonical snapshot |
| `STOPPING` | все `FuzzInstance` terminal | `CANCELLED` / `FAILED` | Выбрать terminal result по исходной причине, сообщить агрегатору Campaign |

`min`/`max` — конфигурация per target (или per Campaign, применяется ко всем её targets по
умолчанию). Elastic reconciliation существует, чтобы один continuous target не монополизировал
quota, даже когда она формально свободна (явное требование: «динамическое контролирование, чтобы
фаззинг-таргет не забирал слишком много ресурсов»).

## 5. Lifecycle: FuzzInstance

Таблица переходов идентична текущей таблице `FuzzJob` (Architecture.md:139-152) — просто
владелец `ExecutionAttempt` теперь `FuzzInstance`, а не `FuzzJob` — плюс одно новое дополнение
для периодической синхронизации corpus:

| Текущая фаза | Условие | Следующая фаза | Обязательный эффект |
|---|---|---|---|
| `RUNNING` | наступил sync interval (jittered per instance, чтобы instances не рестартовали синхронно) | `SYNCING` | Запросить best-effort checkpoint локального corpus-роста |
| `SYNCING` | checkpoint успешен | `RUNNING` | Опубликовать private interim `CorpusSnapshot`; если после merge появился новый canonical snapshot — recycle: пересоздать `ExecutionAttempt` с этим snapshot как input |
| `SYNCING` | checkpoint не успел в течение sync-deadline | `RUNNING` | Пропустить этот sync-цикл без потери instance — sync best-effort, не должен ронять fuzzing |

Остальные переходы (`QUEUED → STARTING`, `RECOVERING`, `PAUSING`, `STOPPING` и т.д.) наследуются
от текущей таблицы `FuzzJob` без изменения смысла — просто «FuzzJob» в её условиях читается как
«FuzzInstance», а «Campaign» продолжает означать Campaign. Исключение — переход
`RUNNING → COMPLETED` по выполнению StopPolicy: он **не** переносится на `FuzzInstance`, так как
StopPolicy теперь оценивается только на уровне `FuzzJob` по агрегированной coverage epoch (§7).
`FuzzInstance` завершается терминально только по команде своего `FuzzJob` (через тот же
`PAUSING`/`STOPPING` handshake, что и при pause/down-scale) — как `CANCELLED` со structured reason,
а не самостоятельно как `COMPLETED`.

Jitter между instances обязателен: без него все instances под одним `FuzzJob` синхронно уходили
бы в `SYNCING` и кратковременно теряли бы всю fuzzing-мощность одновременно.

## 6. Corpus sync через существующий CorpusMergeTask

Механизм полностью переиспользует §12 Architecture.md, без нового API surface:

1. На `SYNCING` `FuzzInstance` публикует **private interim `CorpusSnapshot`** (существующее
   понятие — Architecture.md:286 уже описывает «private campaign snapshot»); `parent` — последний
   snapshot, который instance потреблял как input.
2. Существующий policy-триггер `CorpusMergeTask` («по lifecycle event, интервалу или порогу»,
   Architecture.md:288) реагирует на новый совместимый delta и мёржит его в canonical через CAS —
   без изменений, включая уже описанное поведение empty-delta no-op и conflict-retry
   (Architecture.md:288, Implementation.md:262-264).
3. На следующий recycle `FuzzInstance` пересоздаёт `ExecutionAttempt`, беря на вход актуальный
   canonical snapshot — тот же контракт, что уже есть для `RECOVERING → STARTING`
   («Создать новый ExecutionAttempt с последним snapshot», Architecture.md:146).

Из-за джиттера sync-интервалов мёржи размазаны по времени, а не идут локстепом — instances
сходятся к общим находкам с лагом ~1-2 sync-интервала. Это осознанно eventual-consistency модель,
как в ClusterFuzz, а не live-reload корпуса в работающий процесс fuzzing engine (что потребовало
бы API поддержки от конкретного движка и было бы менее переносимо между libFuzzer/AFL/другими).

## 7. Coverage epoch и StopPolicy

Агрегируются на уровне `FuzzJob`, не `FuzzInstance`:

- Coverage epoch уже поддерживает несколько источников с сохранением provenance каждого источника
  (Architecture.md:298 — разные `BuildArtifact` объединяются в одну epoch с сохранением
  provenance); это естественно расширяется на несколько параллельных `ExecutionAttempt` от разных
  `FuzzInstance` одного `FuzzJob`.
- `noCoverageGrowthFor` (инвариант 13, Architecture.md:401) считается по **wall-clock active time
  `FuzzJob`**, а не суммарно по N instances. Иначе timeout «нет роста 30 минут» наступал бы
  пропорционально быстрее при увеличении N, что не соответствует смыслу настройки (она измеряет
  календарное время застоя exploration, а не совокупный CPU-time).

## 8. Изменения в инвариантах (§19 Architecture.md)

- Инвариант 3 меняется с «Любой ExecutionAttempt принадлежит ровно одному FuzzJob или Task» на
  «ровно одному `FuzzInstance` или `Task`».
- Новый инвариант: `FuzzInstance` принадлежит ровно одному `FuzzJob`; число живых `FuzzInstance`
  под одним `FuzzJob` всегда находится в `[min, max]`, кроме переходных окон `SCALING` и
  replacement после потери instance.
- Новый инвариант: все `FuzzInstance` одного `FuzzJob` используют один и тот же `BuildArtifact` и
  `corpusCompatibilityKey` — они шардят исполнение одного и того же target на одной ревизии, а не
  разные ревизии или targets.
- Инвариант 12 («Resume только с опубликованного compatible snapshot») распространяется на
  recycle `FuzzInstance` при sync так же, как на `RECOVERING → STARTING`.

## 9. Изменения в Implementation.md

- Таблица маппинга на Kubernetes (Implementation.md:136-138) получает новую строку:
  `FuzzInstance ↔ long-running Pod` — один Pod UID соответствует ровно одной попытке одного
  instance, аналогично текущему маппингу `FuzzJob ↔ Long-running Pod`.
- `ResourceLease` запрашивается **per-instance**, а не per-FuzzJob: Kueue admission и scheduling
  gate (README.md:190-208) работают ровно так же, просто на уровень ниже — по одному под на
  instance, N подов на FuzzJob.
- Elastic re-admission (§4, tick reconciliation) требует, чтобы `ResourceAdmission`/Kueue adapter
  поддерживал повторный admission-запрос без пересоздания `FuzzJob`, и graceful stop конкретного
  instance при down-scale (checkpoint best effort перед освобождением lease, тот же handshake, что
  при pause/stop).
- Новый PoC-сценарий (аналог `POC-K8S-ATTEMPT-001`/`POC-KUEUE-001`, Implementation.md:454-455):
  проверить race между elastic scale-down и CAS-конфликтом при merge, поведение при down-scale
  ниже `min` из-за quota pressure (recoverable состояние `FuzzJob`, не terminal), и корректность
  jitter (instances не должны массово синхронизироваться на один sync tick со временем).

## 10. Вне области действия (YAGNI)

- Live corpus reload в работающий процесс fuzzing engine без restart — не рассматривается,
  recycle через checkpoint+restart покрывает потребность при приемлемой сложности.
- Прямой обмен corpus между instances «мимо» `CorpusStore` (peer-to-peer) — не рассматривается,
  весь sync идёт через shared `CorpusStore`, как в ClusterFuzz.
- Шардинг одного target по разным ревизиям/build одновременно — явно запрещено новым инвариантом
  (§8): все instances одного `FuzzJob` обязаны использовать один `BuildArtifact`.
