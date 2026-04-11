---
description: Autonomously diagnose Kubernetes deployment failures with structured root cause analysis and post-mortem generation
---

# Troubleshoot Deployment

Diagnose why a Kubernetes deployment is unhealthy. Perform read-only analysis across all layers — pods, logs, config, networking, resources, images — then synthesize a root cause and generate a post-mortem report.

## Instructions

Follow these eleven phases in strict order. Do not skip phases. If a phase produces no findings, note that and proceed.

**CRITICAL: This command is READ-ONLY. Never run `kubectl apply`, `kubectl delete`, `kubectl edit`, `kubectl patch`, `kubectl scale`, or any command that mutates cluster state.**

---

## Phase 1: Load Skills

Load all diagnostic skills by reading each skill's SKILL.md file. These skills define the rules you must follow throughout all subsequent phases.

**Skills to load (all required):**

| Skill | Purpose |
|-------|---------|
| `cluster-overview` | High-level cluster and namespace health assessment |
| `pod-diagnostics` | Pod status, events, restart analysis |
| `log-analysis` | Container log parsing and error pattern recognition |
| `config-inspection` | ConfigMap, env var, and secret verification |
| `network-diagnostics` | Service, endpoint, and connectivity analysis |
| `resource-analysis` | CPU/memory limits, OOMKilled detection |
| `image-diagnostics` | Image pull status, tag verification |
| `root-cause-synthesis` | Cross-layer correlation and diagnosis |
| `post-mortem` | Structured incident report generation |

**Loading procedure — try each location in order until found:**

For each skill listed above, read the SKILL.md file from the first location that exists:
1. `.claude/skills/<skill-name>/SKILL.md` (project-level)
2. `~/.claude/skills/<skill-name>/SKILL.md` (user-level)

For skills that also have an `examples.md` file in the same directory, read that too.

Load all 9 skills before proceeding to Phase 2.

---

## Phase 2: Input Parsing

Parse the input from `$ARGUMENTS`.

### Input Handling

The argument can be one of:
1. **Namespace only** (e.g., `cms`) — Diagnose all unhealthy resources in the namespace
2. **Namespace + service** (e.g., `cms client-service`) — Focus diagnosis on a specific service
3. **No arguments** — Default to namespace `cms`

### Extract Parameters

- **Namespace**: The Kubernetes namespace to inspect (default: `cms`)
- **Target service**: Optional specific deployment/service to focus on
- **Scope**: `full` (all resources) or `targeted` (specific service)

Proceed immediately after parsing.

---

## Phase 3: Cluster Overview

Apply the `cluster-overview` skill.

### 3.1 Namespace Health Check

```bash
kubectl get all -n <namespace>
```

### 3.2 Identify Unhealthy Resources

Look for:
- Pods not in `Running` state or with `0/1` READY
- Pods with high restart counts (> 2)
- Deployments with `0/1` available replicas
- Services with no endpoints

### 3.3 Triage

Classify each unhealthy resource by symptom:
- **CrashLoopBackOff** — container starts and crashes repeatedly
- **ImagePullBackOff** — cannot pull container image
- **Pending** — cannot be scheduled (resource constraints, node issues)
- **Running but not Ready** — readiness probe failing
- **OOMKilled** — container killed due to memory limit

Record findings and proceed to deeper analysis for each unhealthy resource.

---

## Phase 4: Pod Diagnostics

Apply the `pod-diagnostics` skill.

For each unhealthy pod identified in Phase 3:

### 4.1 Detailed Pod Status

```bash
kubectl describe pod <pod-name> -n <namespace>
```

### 4.2 Key Sections to Analyze

1. **Status/Conditions** — Ready, ContainersReady, PodScheduled
2. **Container State** — Waiting (reason), Running, Terminated (exit code, reason)
3. **Last State** — Previous termination reason (OOMKilled, Error, etc.)
4. **Events** — Recent events (Failed, Unhealthy, BackOff, Pulled, Created)
5. **Restart Count** — How many times the container restarted

### 4.3 Record Findings

Document:
- Current state and reason
- Last termination reason and exit code
- Restart count and pattern
- Relevant events with timestamps

---

## Phase 5: Log Analysis

Apply the `log-analysis` skill.

### 5.1 Current Logs

```bash
kubectl logs <pod-name> -n <namespace> --tail=200
```

### 5.2 Previous Container Logs (if restarting)

```bash
kubectl logs <pod-name> -n <namespace> --previous --tail=200
```

### 5.3 Error Pattern Matching

Look for:
- Java/Spring Boot stack traces (`Exception`, `Caused by`, `at com.`)
- MongoDB connection errors (`MongoTimeoutException`, `MongoSocketOpenException`)
- Port binding failures (`Address already in use`)
- Configuration errors (`Could not resolve placeholder`)
- OOM indicators (`java.lang.OutOfMemoryError`)
- Startup failures (`APPLICATION FAILED TO START`)

Record the specific error messages and stack traces.

---

## Phase 6: Config Inspection

Apply the `config-inspection` skill.

### 6.1 Read ConfigMaps

```bash
kubectl get configmap -n <namespace>
kubectl describe configmap <configmap-name> -n <namespace>
```

### 6.2 Verify Environment Variables

```bash
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].env[*]}'
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].envFrom[*]}'
```

### 6.3 Cross-Reference

Compare configured values against expected values:
- MongoDB URI format: `mongodb://<service-name>:27017/<database>`
- Service hostnames must match actual Kubernetes service names
- Ports must match service/container port definitions
- Database names must be correct for each service

Record any mismatches between configured and expected values.

---

## Phase 7: Network Diagnostics

Apply the `network-diagnostics` skill.

### 7.1 Service and Endpoints

```bash
kubectl get services -n <namespace>
kubectl get endpoints -n <namespace>
```

### 7.2 Verify Connectivity

For each service:
- Check that endpoints list includes pod IPs
- Verify port mappings (service port → target port)
- Check if headless services (ClusterIP: None) resolve correctly

### 7.3 DNS Verification

Verify service DNS names are correct:
- Internal format: `<service-name>.<namespace>.svc.cluster.local`
- Short form: `<service-name>` (within same namespace)

Record any services with empty endpoints or port mismatches.

---

## Phase 8: Resource Analysis

Apply the `resource-analysis` skill.

### 8.1 Resource Configuration

```bash
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].resources}{"\n"}{end}'
```

### 8.2 OOMKilled Detection

```bash
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[0].lastState}{"\n"}{end}'
```

### 8.3 Analysis

- Compare memory limits against JVM requirements (Java 21 needs minimum ~200Mi)
- Check if CPU limits are too restrictive for startup
- Identify if OOMKilled is in last termination state
- Verify requests vs limits ratios are reasonable

Record any resource constraint issues.

---

## Phase 9: Image Diagnostics

Apply the `image-diagnostics` skill.

### 9.1 Image Status

```bash
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\t"}{.status.containerStatuses[0].imageID}{"\n"}{end}'
```

### 9.2 Check for Pull Issues

- Look for `ImagePullBackOff` or `ErrImagePull` in pod events
- Verify image tags exist (`:latest` vs specific versions)
- Check `imagePullPolicy` setting (should be `IfNotPresent` for local images with minikube)
- Verify images are loaded into minikube's image cache

Record any image-related issues.

---

## Phase 10: Root Cause Synthesis

Apply the `root-cause-synthesis` skill.

### 10.1 Correlate Findings

Cross-reference findings from all previous phases:

| Symptom | Common Root Cause |
|---------|-------------------|
| Running 0/1 Ready + MongoTimeoutException in logs | Wrong MongoDB URI in ConfigMap |
| CrashLoopBackOff + connection refused in logs | Dependency service not running |
| OOMKilled in last state + low memory limit | Memory limit too low for JVM |
| Running but 0/1 Ready + 404 in probe logs | Wrong readiness probe path |
| ImagePullBackOff + no imageID | Image not available locally |
| Pending + no events | Insufficient cluster resources |

### 10.2 Determine Root Cause

For each unhealthy resource, determine:
1. **Root cause** — The specific configuration or resource issue
2. **Evidence** — The kubectl output that proves this
3. **Affected components** — What is broken and what downstream effects exist
4. **Recommended fix** — The specific change needed (but DO NOT apply it)

### 10.3 Present Diagnosis

Output a structured diagnosis:

```
## Diagnosis

### Affected Resource: <resource-name>

**Status:** <current status>
**Root Cause:** <one-line description>

**Evidence:**
- <finding 1 with kubectl output reference>
- <finding 2>

**Recommended Fix:**
<specific change — e.g., "Update ConfigMap client-service-config: change SPRING_MONGODB_URI from 'mongodb://mongo-primary:27017/cms-clients' to 'mongodb://mongodb:27017/cms-clients'">
```

---

## Phase 11: Post-Mortem Generation

Apply the `post-mortem` skill.

**Generate a post-mortem ONLY if a root cause was identified in Phase 10.** If all resources are healthy or no root cause was found, skip this phase and report the healthy status instead.

### 11.1 Generate Structured Post-Mortem

Produce a post-mortem report following the template in the `post-mortem` skill. The report must include:

1. **Incident Summary** — One-line description of what happened
2. **Impact** — What services/functionality were affected
3. **Timeline** — Sequence of events from deployment to detection
4. **Root Cause** — Detailed technical explanation
5. **Evidence** — Key kubectl outputs that confirmed the diagnosis
6. **Resolution** — Specific steps to fix the issue (with exact commands)
7. **Prevention** — How to prevent this class of issue in the future
8. **Lessons Learned** — What this incident teaches about the deployment process

### 11.2 Save to File

Save the post-mortem as a markdown file at:
```
postmortems/YYYY-MM-DD-<service>-<short-description>.md
```

Create the `postmortems/` directory if it does not exist. Use today's date. The saved file IS the deliverable of this diagnostic run — always confirm the file path in the output.

---

## Output

When complete, present:

1. **Cluster Health Summary** — Table of all resources with status
2. **Diagnosis** — For each unhealthy resource: root cause, evidence, immediate action with exact commands and verification checklist
3. **Post-Mortem** — Saved to `postmortems/` directory as a markdown file (if root cause was found). Confirm the file path.
4. **Next Steps** — Prioritized list of actions to resolve the issues

---

## Error Handling

- **Namespace not found**: Report that the namespace does not exist and suggest checking `kubectl get namespaces`
- **No unhealthy resources**: Report that all resources are healthy — no post-mortem needed
- **kubectl not available**: Report that kubectl is not installed or not configured
- **No cluster access**: Report authentication/context issue and suggest checking `kubectl config current-context`
- **Partial access**: If some commands fail due to RBAC, note the limitation and continue with available data
- **Multiple issues found**: Diagnose each independently, then note any cascading relationships

---

## Notes

- This command runs fully autonomously — no confirmation prompts between phases
- All analysis is READ-ONLY — never mutate cluster state
- If the namespace contains many resources, focus on unhealthy ones first
- Cross-reference between phases is critical — a single symptom often has evidence across multiple layers
- The post-mortem should be written as if presenting to an engineering team during an incident review
