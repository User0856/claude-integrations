# Resource Analysis — Examples

## Example 1: Detecting OOMKilled with Low Memory Limit

```
$ kubectl get pods -n cms -o custom-columns='NAME:.metadata.name,MEM_REQ:.spec.containers[0].resources.requests.memory,MEM_LIM:.spec.containers[0].resources.limits.memory,STATUS:.status.phase,RESTARTS:.status.containerStatuses[0].restartCount'

NAME                                       MEM_REQ   MEM_LIM   STATUS    RESTARTS
mongodb-0                                  256Mi     512Mi     Running   0
client-service-6d4f8b7c9-x2k4p            256Mi     512Mi     Running   0
contract-service-5c7d9e8f1-m3n7q           256Mi     512Mi     Running   0
billing-service-7a2b3c4d5-p8r2s            32Mi      48Mi      Running   4
notification-service-8e9f0a1b2-t5u9v       256Mi     512Mi     Running   0
```

**Analysis:**
- `billing-service` has memory limit of `48Mi` — far below the ~200Mi minimum for Java 21 Spring Boot
- 4 restarts confirm repeated crashes
- All other services have healthy 256Mi/512Mi allocation

```
$ kubectl get pod billing-service-7a2b3c4d5-p8r2s -n cms -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
OOMKilled
```

**Diagnosis confirmed:** billing-service is OOMKilled because its 48Mi memory limit cannot accommodate the JVM (~150Mi) + Spring Boot framework (~50Mi) + application code.

**Fix:** Increase memory requests to 256Mi and limits to 512Mi.

---

## Example 2: Healthy Resource Allocation

```
$ kubectl get pods -n cms -o custom-columns='NAME:.metadata.name,MEM_REQ:.spec.containers[0].resources.requests.memory,MEM_LIM:.spec.containers[0].resources.limits.memory,CPU_REQ:.spec.containers[0].resources.requests.cpu,CPU_LIM:.spec.containers[0].resources.limits.cpu'

NAME                                       MEM_REQ   MEM_LIM   CPU_REQ   CPU_LIM
mongodb-0                                  256Mi     512Mi     250m      500m
client-service-6d4f8b7c9-x2k4p            256Mi     512Mi     250m      500m
contract-service-5c7d9e8f1-m3n7q           256Mi     512Mi     250m      500m
billing-service-7a2b3c4d5-p8r2s            256Mi     512Mi     250m      500m
notification-service-8e9f0a1b2-t5u9v       256Mi     512Mi     250m      500m
```

**Analysis:** All services have adequate resource allocation. 256Mi request and 512Mi limit is sufficient for Java 21 Spring Boot applications. No resource issues detected.
