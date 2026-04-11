# Root Cause Synthesis

Rules and patterns for correlating findings across all diagnostic phases to determine the definitive root cause of a deployment failure.

## Version: 1.0.0

---

## Rules

### RC01: Follow the Decision Tree

Use this decision tree to narrow down root causes based on observed symptoms:

```
Pod Status?
├── ImagePullBackOff
│   └── Image not available → check image-diagnostics
├── Pending
│   └── Cannot schedule → check node resources, taints, tolerations
├── CrashLoopBackOff
│   ├── Exit Code 137 → OOMKilled → check resource-analysis
│   └── Exit Code 1 → Application error
│       ├── MongoTimeoutException in logs → wrong MongoDB URI → check config-inspection
│       ├── UnknownHostException in logs → wrong hostname → check config-inspection
│       ├── BindException in logs → port conflict → check config-inspection
│       └── Other exception → application bug → check log-analysis
├── Running but 0/1 Ready
│   ├── Probe returns 404 → wrong probe path → check pod-diagnostics
│   ├── Readiness probe timeout + MongoTimeoutException in logs → wrong MongoDB URI → check config-inspection
│   ├── Probe connection refused → wrong probe port → check config-inspection
│   └── Probe timeout (no MongoDB error) → application too slow → check resource-analysis
└── Running and 1/1 Ready
    └── Healthy — no issue
```

### RC02: Single Root Cause Principle

Most deployment issues have a single root cause. Resist the temptation to list multiple possible causes. Use evidence to narrow down to ONE definitive root cause.

Example of BAD analysis: "The issue could be wrong config OR not enough memory OR missing image."
Example of GOOD analysis: "The root cause is wrong MongoDB hostname `mongo-primary` in ConfigMap `client-service-config`. Evidence: exit code 1, `UnknownHostException: mongo-primary` in logs, service name verification shows MongoDB Service is named `mongodb`."

### RC03: Evidence Must Be Specific

Every root cause claim must be backed by specific kubectl output:
- **Weak evidence**: "There seems to be a configuration issue"
- **Strong evidence**: "ConfigMap `client-service-config` contains `SPRING_MONGODB_URI=mongodb://mongo-primary:27017/cms-clients`. The hostname `mongo-primary` does not match any Service in namespace `cms`. `kubectl get svc -n cms` shows the MongoDB Service is named `mongodb`."

### RC04: Identify Cascading Failures

One root cause can create multiple symptoms:
- MongoDB down → ALL services in CrashLoopBackOff (root cause is MongoDB, not the services)
- Wrong ConfigMap → one service crashes → its endpoints disappear → dependent services may fail

Always identify the PRIMARY root cause and explain which other symptoms are consequences.

### RC05: Verify the Fix Is Specific

The recommended fix must be actionable:
- **Vague**: "Fix the configuration"
- **Specific**: "Update ConfigMap `client-service-config`: change `SPRING_MONGODB_URI` from `mongodb://mongo-primary:27017/cms-clients` to `mongodb://mongodb:27017/cms-clients`"

### RC06: Present Diagnosis in Structured Format

Always use this format:

```
## Diagnosis

### Affected Resource
<deployment/pod name>

### Status
<current observed status>

### Root Cause
<one-line summary>

### Evidence
1. <specific finding from Phase N with exact output>
2. <specific finding from Phase M with exact output>
3. <cross-reference that confirms the diagnosis>

### Impact
<what is broken as a result>

### Immediate Action

#### What to change
<exact resource, field, current value → correct value>

#### Source file to fix
<path to the Kustomize overlay or base manifest that introduced the bad value, e.g. `k8s/overlays/scenario-wrong-db-uri/patch-client-configmap.yaml`>

#### Commands to apply the fix
```bash
# Step 1: Edit the source manifest (fix the root cause)
<exact kubectl edit or file edit instruction>

# Step 2: Apply the corrected manifest
kubectl apply -k <overlay-path>

# Step 3: Restart the affected deployment
kubectl rollout restart deployment/<name> -n <namespace>

# Step 4: Verify recovery
kubectl get pods -n <namespace> -l app=<service> -w
# Expected: pod reaches 1/1 Ready within 60-90 seconds
```

#### Verification checklist
- [ ] Pod status transitions from <bad state> to Running 1/1 Ready
- [ ] `kubectl get endpoints -n <namespace> <service>` shows pod IP
- [ ] `kubectl logs <pod> -n <namespace>` shows clean startup with no errors
```

### RC07: If No Root Cause Found

If all phases complete without identifying a clear root cause:
1. Report the current state of all resources
2. List any anomalies found (even minor ones)
3. Suggest additional diagnostic commands that might reveal more
4. Do NOT fabricate a root cause

---

## Common Pitfalls

- **Jumping to conclusions**: CrashLoopBackOff does NOT always mean wrong config. It could be OOM, missing dependency, application bug, or even a wrong image. Follow the decision tree.
- **Diagnosing symptoms instead of causes**: "The pod keeps restarting" is a symptom. "The ConfigMap has the wrong MongoDB hostname" is a root cause.
- **Missing cascading failures**: When 3 out of 4 services are crashing, check what they have in common before diagnosing each individually.
- **Recommending fixes without evidence**: Never recommend changing a value unless you have evidence the current value is wrong.
