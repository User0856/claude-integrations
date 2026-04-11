# Post-Mortem Generation

Rules and patterns for generating structured incident post-mortem reports after diagnosing a Kubernetes deployment failure.

## Version: 1.0.0

---

## Rules

### PM01: Only Generate When Root Cause Is Identified

A post-mortem requires a confirmed root cause. If diagnosis was inconclusive or all resources are healthy, skip post-mortem generation and report the healthy status instead.

### PM02: Follow the Standard Template

Every post-mortem must include these sections in order:

1. **Incident Summary** — one-line description
2. **Severity** — Critical / High / Medium / Low
3. **Impact** — what was affected and for how long
4. **Timeline** — chronological sequence of events
5. **Root Cause** — detailed technical explanation
6. **Evidence** — key diagnostic outputs that confirmed the root cause
7. **Resolution** — specific steps to fix the issue
8. **Prevention** — how to prevent this class of issue in the future
9. **Lessons Learned** — key takeaways for the team

### PM03: Severity Classification

| Severity | Criteria |
|----------|----------|
| Critical | Multiple services down, entire system unavailable |
| High | Single service completely down, no traffic served |
| Medium | Service degraded but partially functional |
| Low | Non-customer-facing issue, cosmetic, or performance only |

### PM04: Timeline Must Be Evidence-Based

Use timestamps from `kubectl describe pod` events and log timestamps to reconstruct the timeline:

```
10:15:00 — Deployment applied with updated ConfigMap
10:15:02 — Pod created and scheduled to node
10:15:05 — Container started
10:15:08 — Application failed to connect to MongoDB (30s timeout begins)
10:15:38 — MongoTimeoutException thrown, application exits with code 1
10:15:40 — Kubelet detects container exit, schedules restart
10:15:42 — Restart #1 begins (same failure)
...
10:20:00 — Diagnosis initiated after 5 restarts
```

### PM05: Root Cause Must Explain the Mechanism

Don't just state what is wrong — explain WHY it caused the failure:

- **Shallow**: "The MongoDB URI was wrong."
- **Deep**: "The ConfigMap `client-service-config` contained hostname `mongo-primary` in the MongoDB URI. This hostname does not correspond to any Kubernetes Service in the namespace. When the Spring Boot application starts, the MongoDB driver attempts DNS resolution of `mongo-primary`, which fails after the 30-second timeout. The resulting `MongoTimeoutException` causes the Spring application context to fail initialization, and the JVM exits with code 1."

### PM06: Resolution Must Be Step-by-Step

Provide copy-pasteable commands:
```
1. kubectl edit configmap client-service-config -n cms
   Change SPRING_MONGODB_URI from:
     mongodb://mongo-primary:27017/cms-clients
   To:
     mongodb://mongodb:27017/cms-clients

2. kubectl rollout restart deployment/client-service -n cms

3. kubectl get pods -n cms -w  (verify pod reaches Running 1/1 Ready)
```

### PM07: Prevention Must Be Actionable

Prevention recommendations should be concrete, not generic:

- **Generic**: "Review configurations more carefully."
- **Actionable**: "Add a CI validation step that checks all ConfigMap MongoDB URIs against known Kubernetes Service names before deployment. Use a conftest policy or OPA Gatekeeper admission controller."

### PM08: Lessons Learned Should Be Honest

Lessons learned should identify what could have caught this earlier:
- Missing monitoring/alerting
- Missing validation in CI/CD
- Missing health check in deployment pipeline
- Gap in documentation or runbook

### PM09: Professional Tone

Write as if presenting to an engineering team during an incident review. Be factual, specific, and non-blaming. Focus on systemic improvements, not individual errors.

### PM10: Save Post-Mortem to File

The post-mortem MUST be saved as a markdown file, not just printed to the console. Save it to:

```
postmortems/YYYY-MM-DD-<service>-<short-description>.md
```

Create the `postmortems/` directory in the project root if it does not exist.

Examples:
- `postmortems/2026-04-09-client-service-wrong-mongodb-uri.md`
- `postmortems/2026-04-09-billing-service-oomkilled.md`
- `postmortems/2026-04-09-contract-service-wrong-probe-path.md`

Use the current date. The file must contain the complete post-mortem — it is the deliverable of this diagnostic run.

---

## Common Pitfalls

- **Writing the post-mortem before confirming root cause**: Never speculate in a post-mortem. Only confirmed findings belong in the report.
- **Skipping the timeline**: Without a timeline, the report reads like a bug report instead of an incident analysis.
- **Vague prevention recommendations**: "Be more careful" is not prevention. Propose specific tooling, automation, or process changes.
- **Blaming individuals**: Post-mortems are about systemic improvement, not blame. Focus on what the system could do better, not what a person should have done.
