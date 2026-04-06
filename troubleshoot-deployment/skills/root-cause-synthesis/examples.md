# Root Cause Synthesis — Examples

## Example 1: Wrong MongoDB URI Diagnosis

### Findings from Previous Phases

| Phase | Finding |
|-------|---------|
| Cluster Overview | client-service: Running but 0/1 Ready, 0 restarts. All other services 1/1 healthy. |
| Pod Diagnostics | Ready: False. Readiness probe timeout. Events: Unhealthy (readiness probe context deadline exceeded). |
| Log Analysis | `MongoSocketOpenException: Exception opening socket` and `UnknownHostException: mongo-primary` in container logs. |
| Config Inspection | `client-service-config` ConfigMap: `SPRING_MONGODB_URI=mongodb://mongo-primary:27017/cms-clients` |
| Network Diagnostics | MongoDB Service name is `mongodb`, no service named `mongo-primary` exists. |
| Resource Analysis | Memory limit 512Mi — adequate. Not OOMKilled. |
| Image Diagnostics | Image present, imageID valid. |

### Synthesized Diagnosis

```
## Diagnosis

### Affected Resource
deployment/client-service (pod/client-service-6d4f8b7c9-x2k4p)

### Status
Running but 0/1 Ready — readiness probe failing (health endpoint reports DOWN due to MongoDB unreachable)

### Root Cause
ConfigMap `client-service-config` contains wrong MongoDB hostname `mongo-primary`. 
The correct hostname is `mongodb` (matching the MongoDB Kubernetes Service name).

### Evidence
1. Container logs show `UnknownHostException: mongo-primary` — DNS cannot resolve this hostname
2. ConfigMap `client-service-config` has `SPRING_MONGODB_URI=mongodb://mongo-primary:27017/cms-clients`
3. `kubectl get svc -n cms` confirms MongoDB Service is named `mongodb`, not `mongo-primary`
4. All other services using `mongodb://mongodb:27017/...` are 1/1 Ready
5. Readiness probe events show timeout — `/actuator/health` blocks waiting for MongoDB health check

### Impact
- client-service is running but not receiving traffic (0/1 Ready)
- All client API endpoints (/api/v1/clients/**) are unreachable — service endpoints are empty
- The pod is alive and liveness probe passes, so it is NOT being restarted

### Recommended Fix
Update ConfigMap `client-service-config`:
- Change: `SPRING_MONGODB_URI=mongodb://mongo-primary:27017/cms-clients`
- To:     `SPRING_MONGODB_URI=mongodb://mongodb:27017/cms-clients`
Then restart the deployment: `kubectl rollout restart deployment/client-service -n cms`
```

---

## Example 2: OOMKilled Diagnosis

### Findings from Previous Phases

| Phase | Finding |
|-------|---------|
| Cluster Overview | billing-service: CrashLoopBackOff, 4 restarts. Others healthy. |
| Pod Diagnostics | Last State: Terminated, OOMKilled, Exit Code 137. Memory limit: 48Mi. |
| Log Analysis | Logs truncated mid-line during Tomcat initialization. No exception. |
| Config Inspection | ConfigMap correct — `mongodb://mongodb:27017/cms-billing`. |
| Resource Analysis | Memory limit 48Mi, well below 200Mi minimum for Java 21 Spring Boot. |

### Synthesized Diagnosis

```
## Diagnosis

### Affected Resource
deployment/billing-service (pod/billing-service-7a2b3c4d5-p8r2s)

### Status
CrashLoopBackOff — 4 restarts, OOMKilled (exit code 137)

### Root Cause
Memory limit of 48Mi is insufficient for a Java 21 Spring Boot application.
The JVM alone requires ~150Mi, plus Spring Boot framework (~50Mi), plus application code.
The container is killed by the kernel OOM killer during startup before the application can initialize.

### Evidence
1. `kubectl describe pod` shows Last State: Terminated, Reason: OOMKilled, Exit Code: 137
2. Resource limits show memory: 48Mi (request: 32Mi)
3. Container logs are truncated mid-startup (`Tomcat initia...`) — killed externally
4. No application exception in logs — confirms OOM kill, not application error
5. All other services with 512Mi limits are running successfully

### Impact
- billing-service is completely unavailable (0/1 replicas ready)
- All billing API endpoints return 503

### Recommended Fix
Increase billing-service memory allocation in the Deployment:
- Change requests.memory: `32Mi` → `256Mi`
- Change limits.memory: `48Mi` → `512Mi`
Then restart: `kubectl rollout restart deployment/billing-service -n cms`
```

---

## Example 3: Wrong Readiness Probe Diagnosis

### Findings from Previous Phases

| Phase | Finding |
|-------|---------|
| Cluster Overview | contract-service: Running but 0/1 Ready. Others healthy. |
| Pod Diagnostics | Ready: False. Readiness probe: `/health/ready`. Events: Unhealthy x40, 404. |
| Log Analysis | Application started successfully. WARN about `/health/ready` path not found. |
| Config Inspection | ConfigMap correct. |
| Network Diagnostics | contract-service endpoints: `<none>` (expected — pod not Ready). |

### Synthesized Diagnosis

```
## Diagnosis

### Affected Resource
deployment/contract-service (pod/contract-service-5c7d9e8f1-m3n7q)

### Status
Running but 0/1 Ready — readiness probe failing with HTTP 404

### Root Cause
Readiness probe path is misconfigured as `/health/ready`. Spring Boot Actuator exposes 
health checks at `/actuator/health`. The path `/health/ready` does not exist, returning 404.

### Evidence
1. `kubectl describe pod` shows readiness probe: `http-get http://:8082/health/ready`
2. Events show `Readiness probe failed: HTTP probe failed with statuscode: 404` (x40 over 6m)
3. Application logs show successful startup — `Started ContractServiceApplication in 3.5 seconds`
4. Liveness probe uses correct path `/actuator/health` and is passing (pod is not being restarted)
5. Application WARN log: `Path with "health/ready" was not found`

### Impact
- contract-service is running but receiving NO traffic
- Service endpoints list is empty — clients get connection refused
- All contract API endpoints (/api/v1/contracts/**) are unreachable

### Recommended Fix
Change the readiness probe path in the contract-service Deployment:
- Change: `readinessProbe.httpGet.path: /health/ready`
- To:     `readinessProbe.httpGet.path: /actuator/health`
Then restart: `kubectl rollout restart deployment/contract-service -n cms`
```
