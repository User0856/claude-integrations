# Pod Diagnostics

Rules and patterns for analyzing pod status, events, container states, and restart behavior using `kubectl describe pod`.

## Version: 1.0.0

---

## Rules

### PD01: Always Run `kubectl describe pod` for Unhealthy Pods

`kubectl get` only shows summary status. `kubectl describe pod` reveals:
- Container state details (waiting reason, exit codes)
- Last termination state (critical for CrashLoopBackOff)
- Events section (scheduling, pulling, creating, health check results)
- Resource limits actually applied
- Probe configuration
- Environment variables and ConfigMap references

### PD02: Parse the Conditions Section

The Conditions section shows boolean readiness gates:
```
Conditions:
  Type              Status
  Initialized       True
  Ready             False    ← pod not serving traffic
  ContainersReady   False    ← container not passing readiness probe
  PodScheduled      True
```
- `Ready: False` + `ContainersReady: False` = readiness probe failing
- `PodScheduled: False` = cannot be placed on any node

### PD03: Container State Reveals the Mechanism

Under `Containers:` look for the `State:` and `Last State:` fields:

**Waiting states:**
- `Waiting: CrashLoopBackOff` — container crashed, kubelet backing off before restart
- `Waiting: ImagePullBackOff` — image cannot be pulled
- `Waiting: ContainerCreating` — still initializing

**Terminated states (in Last State):**
- `Terminated: OOMKilled (exit code 137)` — killed by kernel OOM killer, memory limit too low
- `Terminated: Error (exit code 1)` — application exited with error (check logs)
- `Terminated: Completed (exit code 0)` — application exited normally (unexpected for a server)

### PD04: Exit Code 137 = OOMKilled

Exit code 137 means the process received SIGKILL (128 + 9). In Kubernetes, this almost always means the container exceeded its memory limit and was killed by the OOM killer.

### PD05: Exit Code 1 = Application Error

Exit code 1 means the Java process threw an unhandled exception during startup. The root cause will be in the container logs (Phase 5), not in `kubectl describe`.

### PD06: Events Section Is Chronological Evidence

The Events section at the bottom of `kubectl describe pod` shows a timeline:
```
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  5m                 default-scheduler  Successfully assigned...
  Normal   Pulled     3m (x4 over 5m)   kubelet            Container image already present
  Normal   Created    3m (x4 over 5m)   kubelet            Created container
  Normal   Started    3m (x4 over 5m)   kubelet            Started container
  Warning  BackOff    30s (x12 over 4m)  kubelet            Back-off restarting failed container
```
- `(x4 over 5m)` means this event happened 4 times — the container was created 4 times in 5 minutes
- `Warning BackOff` confirms CrashLoopBackOff
- Look for `Warning Unhealthy` events — these indicate failed probes

### PD07: Probe Failure Events

```
Events:
  Warning  Unhealthy  2s (x30 over 5m)  kubelet  Readiness probe failed: HTTP probe failed with statuscode: 404
```
- `statuscode: 404` means the probe path does not exist on the application
- `connection refused` means the container port is not listening
- `timeout` means the application is too slow to respond

### PD08: Check Probe Configuration

Under `Containers:` look for `Readiness:` and `Liveness:` fields:
```
Readiness:  http-get http://:8082/health/ready delay=30s timeout=1s period=10s
Liveness:   http-get http://:8082/actuator/health delay=45s timeout=1s period=15s
```
- Verify the path matches what the application actually exposes
- Spring Boot Actuator health endpoint is `/actuator/health` — NOT `/health/ready`, `/healthz`, or `/health`
- Verify the port matches the container's SERVER_PORT

### PD09: Restart Count Pattern Analysis

- Restarts increasing rapidly (every 10-30s) = immediate crash on startup (config error, missing dependency)
- Restarts increasing slowly (every few minutes) = application starts but crashes under load or after some time
- Restarts stopped increasing = kubelet backoff has reached maximum (5 minutes), but the problem persists

---

## Common Pitfalls

- **Ignoring Last State**: The current state may be `Waiting: CrashLoopBackOff`, but the *reason* for the crash is in `Last State: Terminated`. Always check both.
- **Confusing liveness and readiness probes**: Liveness probe failure → pod gets killed and restarted. Readiness probe failure → pod stays alive but gets no traffic. Different symptoms, different remedies.
- **Missing the exit code**: Exit code 137 (OOMKilled) looks like a crash but is actually a resource issue. Exit code 1 is an application error. The fix is completely different.
- **Not correlating events with states**: Events say "what happened when." Container state says "what is the current situation." You need both.
