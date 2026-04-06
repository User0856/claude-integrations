# Network Diagnostics — Examples

## Example 1: Healthy Endpoints

```
$ kubectl get endpoints -n cms
NAME                   ENDPOINTS            AGE
mongodb                172.17.0.4:27017     10m
client-service         172.17.0.5:8081      10m
contract-service       172.17.0.6:8082      10m
billing-service        172.17.0.7:8083      10m
notification-service   172.17.0.8:8084      10m
```

**Analysis:** All services have endpoints — pods are Ready and receiving traffic. No networking issues.

---

## Example 2: Missing Endpoint Due to Readiness Probe Failure

```
$ kubectl get endpoints -n cms
NAME                   ENDPOINTS            AGE
mongodb                172.17.0.4:27017     10m
client-service         172.17.0.5:8081      10m
contract-service       <none>               10m
billing-service        172.17.0.7:8083      10m
notification-service   172.17.0.8:8084      10m
```

**Analysis:**
- `contract-service` has `<none>` for endpoints — no traffic can reach it
- This is because the contract-service pod is not Ready (readiness probe failing)
- The fix is in the pod/probe configuration, not in the Service
- Cross-reference with `kubectl get pods -n cms` to confirm the pod is Running but 0/1 Ready

---

## Example 3: Verifying Service Port Mapping

```
$ kubectl get svc client-service -n cms -o yaml
apiVersion: v1
kind: Service
spec:
  selector:
    app: client-service
  ports:
    - port: 8081        # Service port (what other pods connect to)
      targetPort: 8081  # Container port (where traffic is forwarded)
```

**Analysis:** Service port 8081 maps to targetPort 8081. The container must be listening on 8081 (set by SERVER_PORT in ConfigMap). The chain is consistent.
