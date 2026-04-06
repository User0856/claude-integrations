# Network Diagnostics

Rules and patterns for analyzing Kubernetes Services, Endpoints, DNS resolution, and connectivity between pods.

## Version: 1.0.0

---

## Rules

### ND01: Check Endpoints Before Assuming Network Issues

A Service with no endpoints means no traffic can reach the target pods:
```bash
kubectl get endpoints -n <namespace>
```

Empty endpoints usually means:
- No pods match the Service selector
- Matching pods exist but are not Ready (readiness probe failing)

### ND02: Verify Service Selector Matches Pod Labels

A Service routes traffic to pods whose labels match its `spec.selector`:
```bash
kubectl get svc <service-name> -n <namespace> -o jsonpath='{.spec.selector}'
kubectl get pods -n <namespace> --show-labels
```

If the selector is `app: client-service`, pods must have the label `app: client-service`.

### ND03: Headless Services for StatefulSets

MongoDB uses a headless Service (ClusterIP: None). This means:
- No cluster IP is assigned
- DNS returns pod IPs directly
- Pods are accessible as `mongodb-0.mongodb.<namespace>.svc.cluster.local`
- The short name `mongodb` still resolves within the same namespace

### ND04: Port Mapping Chain

Traffic flows through a chain of port definitions:
```
Client → Service port → targetPort → containerPort → Application SERVER_PORT
```

All must agree. A mismatch at any point means traffic doesn't reach the application.

### ND05: Services with Empty Endpoints Are Not Errors by Themselves

If a pod is not Ready (readiness probe failing), it is automatically removed from Service endpoints. This is expected Kubernetes behavior — the fix is in the pod/probe, not the Service.

### ND06: DNS Resolution Within Namespace

Services in the same namespace can be reached by short name:
- `mongodb` resolves to the MongoDB Service
- `client-service` resolves to the client-service Service

Cross-namespace access requires FQDN: `<service>.<namespace>.svc.cluster.local`

---

## Common Pitfalls

- **Treating empty endpoints as a network issue**: Usually it's a pod readiness issue, not a networking problem.
- **Confusing Service port with containerPort**: Service port is external-facing, targetPort maps to the container. They can be different.
- **Assuming DNS failure when the real issue is application crash**: If client-service can't reach MongoDB, check if MongoDB pods are running first. DNS resolution failure vs. connection refusal vs. timeout are different symptoms.
