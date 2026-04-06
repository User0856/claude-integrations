# Resource Analysis

Rules and patterns for analyzing CPU/memory resource requests, limits, OOMKilled detection, and JVM sizing in Kubernetes.

## Version: 1.0.0

---

## Rules

### RA01: Java 21 Spring Boot Minimum Memory Requirements

A Java 21 Spring Boot application with MongoDB needs at minimum:
- **Startup**: ~200Mi (JVM initialization + Spring context + embedded Tomcat)
- **Steady state**: ~150-256Mi depending on load
- **Recommended limits**: 256Mi request / 512Mi limit

Setting memory limits below 200Mi will result in OOMKilled during startup.

### RA02: Detect OOMKilled via Last Termination State

```bash
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[0].lastState.terminated.reason}{"\t"}{.status.containerStatuses[0].lastState.terminated.exitCode}{"\n"}{end}'
```

If output shows `OOMKilled` with exit code `137`, the container exceeded its memory limit.

### RA03: Compare Limits Against Requirements

When investigating OOMKilled:
1. Read the pod's memory limit: `kubectl get pod <pod> -n <ns> -o jsonpath='{.spec.containers[0].resources.limits.memory}'`
2. Compare against the minimum requirement (200Mi for Java 21 Spring Boot)
3. If limit < 200Mi → insufficient, guaranteed OOMKilled on startup

### RA04: Memory Request vs Limit

- **Request**: What the scheduler uses to place the pod. Pod is guaranteed this much memory.
- **Limit**: Hard ceiling. If the container tries to use more, it is OOMKilled.
- Request should be close to typical usage (256Mi)
- Limit should accommodate spikes (512Mi)
- If request > available node memory, pod stays Pending

### RA05: CPU Throttling vs OOM

CPU limits cause throttling (slowness), not crashes. Memory limits cause OOMKill (crashes).
- Slow startup but eventually works → CPU may be too restrictive
- Crash during startup → memory is too restrictive
- Never diagnose OOMKill as a CPU issue

### RA06: Check All Pods' Resource Configuration

```bash
kubectl get pods -n <namespace> -o custom-columns='NAME:.metadata.name,MEM_REQ:.spec.containers[0].resources.requests.memory,MEM_LIM:.spec.containers[0].resources.limits.memory,CPU_REQ:.spec.containers[0].resources.requests.cpu,CPU_LIM:.spec.containers[0].resources.limits.cpu'
```

This gives a quick overview of resource allocation across all pods.

---

## Common Pitfalls

- **Confusing Mi with M**: `48Mi` = 48 mebibytes. `48M` = 48 megabytes. In practice, nearly the same, but be precise.
- **Blaming the application for OOMKilled**: OOMKilled is a resource constraint issue, not an application bug. The fix is increasing limits, not changing code.
- **Setting requests = limits**: This creates a Guaranteed QoS class, which is good for predictability but wastes resources if the application doesn't always use the full amount.
- **Ignoring JVM overhead**: The JVM itself uses ~100-150Mi before any application code runs. Spring Boot framework adds another ~50-100Mi. Application code is on top of that.
