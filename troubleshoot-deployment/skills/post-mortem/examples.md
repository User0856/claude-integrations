# Post-Mortem — Examples

## Example 1: Wrong MongoDB URI Post-Mortem

```markdown
# Incident Post-Mortem: client-service Deployment Failure

## Incident Summary
client-service entered CrashLoopBackOff due to incorrect MongoDB hostname in ConfigMap.

## Severity
**High** — Single service completely unavailable, all client API endpoints returning 503.

## Impact
- **Service affected:** client-service
- **Endpoints unavailable:** All `/api/v1/clients/**` endpoints
- **Duration:** From deployment until diagnosis (~8 minutes observed)
- **User impact:** All client management operations unavailable. Other services (contract, billing, notification) unaffected.

## Timeline
| Time | Event |
|------|-------|
| T+0s | Deployment applied with updated ConfigMap containing `mongo-primary` hostname |
| T+2s | Pod scheduled and container started |
| T+5s | Spring Boot begins initialization, MongoDB driver attempts connection to `mongo-primary:27017` |
| T+35s | MongoTimeoutException after 30s timeout — DNS cannot resolve `mongo-primary` |
| T+35s | Spring application context fails, JVM exits with code 1 |
| T+37s | Kubelet detects crash, schedules restart |
| T+40s | Restart #1 — same failure |
| T+2m | Restart #3 — kubelet begins exponential backoff |
| T+5m | Restart #5 — BackOff delay increasing |
| T+8m | Diagnosis initiated |

## Root Cause
The ConfigMap `client-service-config` contained an incorrect MongoDB connection URI:

```
SPRING_MONGODB_URI=mongodb://mongo-primary:27017/cms-clients
```

The hostname `mongo-primary` does not correspond to any Kubernetes Service in the `cms` namespace. The MongoDB StatefulSet is exposed via a headless Service named `mongodb`. When the Spring Boot application starts, the MongoDB Java driver attempts to resolve `mongo-primary` via Kubernetes DNS, which fails. After the 30-second connection timeout, a `MongoTimeoutException` is thrown, causing the Spring application context initialization to fail and the JVM to exit with code 1. Kubernetes detects the container exit and restarts it, creating the CrashLoopBackOff cycle.

## Evidence
1. Pod describe: `State: Waiting, Reason: CrashLoopBackOff`, `Last State: Terminated, Reason: Error, Exit Code: 1`
2. Previous container logs: `Caused by: java.net.UnknownHostException: mongo-primary`
3. ConfigMap content: `SPRING_MONGODB_URI=mongodb://mongo-primary:27017/cms-clients`
4. Service listing: MongoDB Service is named `mongodb`, no service named `mongo-primary` exists
5. Other services using `mongodb://mongodb:27017/...` are healthy

## Resolution
1. Correct the ConfigMap:
   ```bash
   kubectl edit configmap client-service-config -n cms
   # Change SPRING_MONGODB_URI to: mongodb://mongodb:27017/cms-clients
   ```
2. Restart the deployment:
   ```bash
   kubectl rollout restart deployment/client-service -n cms
   ```
3. Verify recovery:
   ```bash
   kubectl get pods -n cms -l app=client-service -w
   # Wait for 1/1 Ready
   ```

## Prevention
1. **ConfigMap validation in CI**: Add a pipeline step that validates all MongoDB URIs reference known Service names in the Kustomize manifests.
2. **Admission controller**: Deploy an OPA Gatekeeper policy that rejects ConfigMaps with MongoDB URIs referencing non-existent Services.
3. **Readiness gate**: Ensure the readiness probe verifies MongoDB connectivity before marking the pod as Ready, so failures are surfaced immediately.
4. **Deployment smoke test**: Add a post-deployment health check that verifies all pods reach Ready state within 60 seconds, rolling back automatically on failure.

## Lessons Learned
- The deployment pipeline had no validation of ConfigMap values against actual cluster state.
- The 30-second MongoDB connection timeout delays failure detection — each restart cycle takes ~35 seconds before the crash is apparent.
- CrashLoopBackOff exponential backoff means the time between retries grows, so the pod spends more time in Waiting state than actually attempting to start.
- A single character difference in a hostname (`mongo-primary` vs `mongodb`) caused complete service outage — hostname validation should be automated.
```

---

## Example 2: OOMKilled Post-Mortem

```markdown
# Incident Post-Mortem: billing-service OOMKilled

## Incident Summary
billing-service repeatedly killed by OOM killer due to memory limit of 48Mi — far below the ~200Mi minimum required for Java 21 Spring Boot.

## Severity
**High** — Single service completely unavailable.

## Impact
- **Service affected:** billing-service
- **Endpoints unavailable:** All `/api/v1/billing/**` endpoints
- **Duration:** From deployment until diagnosis
- **User impact:** All billing operations unavailable.

## Timeline
| Time | Event |
|------|-------|
| T+0s | Deployment applied with 48Mi memory limit |
| T+2s | Pod scheduled, container started |
| T+3s | JVM begins initialization, allocates metaspace and heap |
| T+5s | Memory usage exceeds 48Mi limit during Spring context initialization |
| T+5s | Kernel OOM killer sends SIGKILL (exit code 137) |
| T+7s | Kubelet restarts container — same immediate OOM |
| T+5m | 4 restarts, diagnosis initiated |

## Root Cause
The Deployment manifest specified a memory limit of 48Mi for the billing-service container. Java 21's JVM requires approximately 150Mi just for the runtime (metaspace, JIT compiler, garbage collector, thread stacks), and Spring Boot's framework initialization adds another 50-100Mi. The 48Mi limit is exceeded within seconds of container startup, before the application can even begin initializing its beans. The kernel's OOM killer sends SIGKILL to the JVM process, producing exit code 137.

## Evidence
1. Pod describe: `Last State: Terminated, Reason: OOMKilled, Exit Code: 137`
2. Resource limits: `limits.memory: 48Mi`, `requests.memory: 32Mi`
3. Container logs truncated mid-startup: `Tomcat initia...` — process killed externally
4. No application exception logged — confirms kernel-level kill
5. Other services with 512Mi limits are healthy

## Resolution
1. Update the Deployment resource limits:
   ```yaml
   resources:
     requests:
       memory: "256Mi"
     limits:
       memory: "512Mi"
   ```
2. Apply and restart:
   ```bash
   kubectl apply -f <updated-manifest>
   kubectl rollout restart deployment/billing-service -n cms
   ```

## Prevention
1. **Resource policy enforcement**: Implement a minimum memory limit policy (e.g., LimitRange in the namespace) that rejects Deployments with limits below 200Mi for Java containers.
2. **CI manifest validation**: Add a linting step that flags Java service Deployments with memory limits below a configurable threshold.
3. **JVM memory flags**: Set `-XX:MaxRAMPercentage=75` to let the JVM auto-size to 75% of the container limit, preventing heap from exceeding the limit.

## Lessons Learned
- Java applications have a high baseline memory requirement that is non-negotiable — the JVM itself needs ~150Mi before application code runs.
- OOMKilled produces no application-level error logs, making it harder to diagnose than application errors. The key diagnostic is exit code 137 + truncated logs.
- The difference between a working Java container (512Mi) and an OOMKilled one (48Mi) is a 10x factor — resource limits should be derived from profiling, not arbitrary values.
```
