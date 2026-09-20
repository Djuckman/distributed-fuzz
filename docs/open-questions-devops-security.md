# Открытые вопросы: инфраструктура и supply chain

Ненормативный документ. Не изменяет `Architecture.md` и не вводит domain-сущности. Фиксирует вопросы DevOps/Security, не покрытые текущей документацией, для последующей проработки в ADR.

## 1. Инфраструктура кластера (требует проработки)

- **Cluster autoscaling vs Kueue scheduling gates.** Admission через Kueue (quota, fair sharing, preemption) описан, но не описан сценарий нехватки физической capacity кластера (а не квоты) — сколько FuzzJob/Task ждёт в очереди, когда должен сработать cluster-autoscaler и как это отражается в condition `Available`/`Degraded`. Нужен PoC: latency autoscaler vs время жизни `ResourceLease`.
- **Владение жизненным циклом Kueue/Kubernetes.** Kueue разворачивается администратором кластера один раз, но процесс регулярного обновления/патчинга Kueue и Kubernetes и проверки совместимости с уже закреплёнными версиями не описан. Нужен owner и cadence, иначе зафиксированный в ADR pin устареет сразу после принятия.

## 2. Что стоит упростить для первых версий

### Supply chain

Полная verification-цепочка (SBOM, подпись образов, проверка provenance builder/runner images) избыточна для MVP:

- digest pinning без cosign/sigstore verification — pin фиксируется вручную через ADR, без автоматической проверки подписи на admission;
- SBOM для builder/runner images не требуется на MVP-этапе; вернуться к вопросу при появлении второго provider или внешнего заказчика с повышенными требованиями;
- зафиксировать это как явное, осознанное упрощение в ADR при выборе `BuildExecutor`/`ExecutionBackend`, а не как забытый пункт.

