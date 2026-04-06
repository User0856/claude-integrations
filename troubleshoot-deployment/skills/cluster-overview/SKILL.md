# Cluster Overview

Rules and patterns for performing high-level Kubernetes cluster and namespace health assessment.

## Version: 1.0.0

---

## Rules

### CO01: Always Start with `kubectl get all`

The first command for any namespace diagnosis must be `kubectl get all -n <namespace>`. This gives a complete picture of all resource types and their states.

### CO02: Check Pod STATUS Column First

The STATUS column is the primary triage signal:
- `Running` — container is alive (but check READY column)
- `CrashLoopBackOff` — container starts and immediately crashes, kubelet is backing off restarts
- `ImagePullBackOff` / `ErrImagePull` — cannot pull the container image
- `Pending` — pod cannot be scheduled onto a node
- `Terminating` — pod is being deleted
- `Init:CrashLoopBackOff` — init container is crash-looping
- `ContainerCreating` — image pulled, container being set up

### CO03: READY Column Reveals Probe Failures

A pod showing `0/1` in the READY column while STATUS is `Running` means the readiness probe is failing. The container is alive but Kubernetes will not route traffic to it.

### CO04: RESTARTS Column Indicates Instability

- 0 restarts = stable
- 1-2 restarts = may have had a transient issue during startup
- 3+ restarts = persistent problem, container keeps crashing
- High restarts with `CrashLoopBackOff` = the container cannot stay running

### CO05: Check Deployment AVAILABLE Column

For Deployments, the `AVAILABLE` column shows how many replicas are actually serving traffic:
- `0/1` = no healthy replicas, service is down
- `1/1` = healthy
- Look at `UP-TO-DATE` vs `AVAILABLE` — if UP-TO-DATE > AVAILABLE, a rollout may be stuck

### CO06: Services with No Endpoints Are Silent Failures

A Service exists but routes to nothing if its selector matches no healthy pods. This often happens when pods are not Ready. Check with `kubectl get endpoints -n <namespace>`.

### CO07: StatefulSet Ordering Matters

StatefulSets create pods sequentially (pod-0 must be Ready before pod-1 starts). If pod-0 of MongoDB is not Ready, no application pods that depend on it will function correctly.

### CO08: Classify and Prioritize

After initial assessment, classify each unhealthy resource:
1. **Infrastructure dependencies first** (databases, message queues)
2. **Application services second**
3. **Cascading failures** — a MongoDB failure will cause ALL services to fail; diagnose MongoDB first before investigating individual service crashes

---

## Common Pitfalls

- **Confusing Running with Healthy**: A pod can be `Running` but `0/1 Ready`. This means it is alive but not serving traffic.
- **Ignoring AGE column**: A pod that was recently recreated (AGE: 5s) in the middle of older pods suggests it just restarted.
- **Missing cascading failures**: If MongoDB is down, all 4 services will be in CrashLoopBackOff. The root cause is MongoDB, not the services.
- **Not checking all resource types**: `kubectl get all` may not include ConfigMaps, Secrets, or PVCs. For config issues, run `kubectl get configmap -n <namespace>` separately.
