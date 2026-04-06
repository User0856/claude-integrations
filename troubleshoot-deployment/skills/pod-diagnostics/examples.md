# Pod Diagnostics — Examples

## Example 1: CrashLoopBackOff Due to Application Error (Exit Code 1)

```
$ kubectl describe pod client-service-6d4f8b7c9-x2k4p -n cms

Name:             client-service-6d4f8b7c9-x2k4p
Namespace:        cms
Status:           Running
Containers:
  client-service:
    Image:         cms/client-service:latest
    Port:          8081/TCP
    State:         Waiting
      Reason:      CrashLoopBackOff
    Last State:    Terminated
      Reason:      Error
      Exit Code:   1
      Started:     Sat, 05 Apr 2026 10:15:32 +0000
      Finished:    Sat, 05 Apr 2026 10:15:35 +0000
    Restart Count: 5
    Limits:
      cpu:     500m
      memory:  512Mi
    Requests:
      cpu:     250m
      memory:  256Mi
    Readiness:  http-get http://:8081/actuator/health delay=30s timeout=1s period=10s
    Liveness:   http-get http://:8081/actuator/health delay=45s timeout=1s period=15s
    Environment Variables from:
      client-service-config  ConfigMap  Optional: false
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
Events:
  Type     Reason     Age                 From               Message
  ----     ------     ----                ----               -------
  Normal   Scheduled  8m                  default-scheduler  Successfully assigned cms/client-service-6d4f8b7c9-x2k4p to minikube
  Normal   Pulled     6m (x5 over 8m)    kubelet            Container image "cms/client-service:latest" already present on machine
  Normal   Created    6m (x5 over 8m)    kubelet            Created container client-service
  Normal   Started    6m (x5 over 8m)    kubelet            Started container client-service
  Warning  BackOff    45s (x22 over 7m)  kubelet            Back-off restarting failed container
```

**Analysis:**
- State: `Waiting: CrashLoopBackOff` — container keeps crashing
- Last State: `Terminated: Error, Exit Code 1` — application error (NOT OOMKilled)
- Container lived ~3 seconds (Started 10:15:32, Finished 10:15:35) — crashed during startup
- 5 restarts, BackOff events confirm repeated crashes
- Memory limits are adequate (512Mi) — not an OOM issue
- **Next step:** Check container logs for the Java exception causing exit code 1

---

## Example 2: OOMKilled (Exit Code 137)

```
$ kubectl describe pod billing-service-7a2b3c4d5-p8r2s -n cms

Name:             billing-service-7a2b3c4d5-p8r2s
Namespace:        cms
Containers:
  billing-service:
    Image:         cms/billing-service:latest
    Port:          8083/TCP
    State:         Waiting
      Reason:      CrashLoopBackOff
    Last State:    Terminated
      Reason:      OOMKilled
      Exit Code:   137
      Started:     Sat, 05 Apr 2026 10:20:01 +0000
      Finished:    Sat, 05 Apr 2026 10:20:04 +0000
    Restart Count: 4
    Limits:
      cpu:     500m
      memory:  48Mi      ← THIS IS THE PROBLEM
    Requests:
      cpu:     100m
      memory:  32Mi
Events:
  Type     Reason     Age                From      Message
  ----     ------     ----               ----      -------
  Normal   Scheduled  5m                 default-scheduler  Successfully assigned...
  Normal   Pulled     3m (x4 over 5m)   kubelet   Container image already present
  Normal   Created    3m (x4 over 5m)   kubelet   Created container
  Normal   Started    3m (x4 over 5m)   kubelet   Started container
  Warning  BackOff    15s (x14 over 4m) kubelet   Back-off restarting failed container
```

**Analysis:**
- Last State: `Terminated: OOMKilled, Exit Code 137` — killed by OOM killer
- Memory limit is `48Mi` — far too low for a Java 21 Spring Boot application (needs ~200Mi minimum)
- Container lived ~3 seconds before being killed
- **Root cause:** Memory limit is insufficient. Java 21 JVM + Spring Boot framework require at least 200Mi at startup.
- **Fix:** Increase memory limit to at least 256Mi (requests) / 512Mi (limits)

---

## Example 3: Readiness Probe Failure (Running but Not Ready)

```
$ kubectl describe pod contract-service-5c7d9e8f1-m3n7q -n cms

Name:             contract-service-5c7d9e8f1-m3n7q
Namespace:        cms
Containers:
  contract-service:
    Image:         cms/contract-service:latest
    Port:          8082/TCP
    State:         Running
      Started:     Sat, 05 Apr 2026 10:10:00 +0000
    Ready:         False
    Restart Count: 0
    Readiness:     http-get http://:8082/health/ready delay=30s timeout=1s period=10s
    Liveness:      http-get http://:8082/actuator/health delay=45s timeout=1s period=15s
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
Events:
  Type     Reason     Age                 From     Message
  ----     ------     ----                ----     -------
  Normal   Scheduled  7m                  default-scheduler  Successfully assigned...
  Normal   Pulled     7m                  kubelet  Container image already present
  Normal   Created    7m                  kubelet  Created container
  Normal   Started    7m                  kubelet  Started container
  Warning  Unhealthy  2s (x40 over 6m)   kubelet  Readiness probe failed: HTTP probe failed with statuscode: 404
```

**Analysis:**
- State: `Running` but `Ready: False` — container is alive, probe is failing
- 0 restarts — liveness probe is passing (path: `/actuator/health`), so the container is not being killed
- Readiness probe path: `/health/ready` — this is NOT a valid Spring Boot Actuator endpoint
- Event: `Readiness probe failed: HTTP probe failed with statuscode: 404` — confirms the path doesn't exist
- Liveness probe uses correct path `/actuator/health` — this is why the pod stays Running
- **Root cause:** Readiness probe path is misconfigured. Spring Boot Actuator exposes `/actuator/health`, not `/health/ready`.
- **Fix:** Change readiness probe path from `/health/ready` to `/actuator/health`
