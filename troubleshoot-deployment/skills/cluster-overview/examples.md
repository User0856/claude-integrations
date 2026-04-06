# Cluster Overview — Examples

## Example 1: Healthy Namespace

```
$ kubectl get all -n cms

NAME                                        READY   STATUS    RESTARTS   AGE
pod/mongodb-0                               1/1     Running   0          10m
pod/client-service-6d4f8b7c9-x2k4p         1/1     Running   0          10m
pod/contract-service-5c7d9e8f1-m3n7q        1/1     Running   0          10m
pod/billing-service-7a2b3c4d5-p8r2s         1/1     Running   0          10m
pod/notification-service-8e9f0a1b2-t5u9v    1/1     Running   0          10m

NAME                           TYPE        CLUSTER-IP       PORT(S)     AGE
service/mongodb                ClusterIP   None             27017/TCP   10m
service/client-service         ClusterIP   10.96.142.31     8081/TCP    10m
service/contract-service       ClusterIP   10.96.142.32     8082/TCP    10m
service/billing-service        ClusterIP   10.96.142.33     8083/TCP    10m
service/notification-service   ClusterIP   10.96.142.34     8084/TCP    10m

NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/client-service         1/1     1            1           10m
deployment.apps/contract-service       1/1     1            1           10m
deployment.apps/billing-service        1/1     1            1           10m
deployment.apps/notification-service   1/1     1            1           10m

NAME                                READY   AGE
statefulset.apps/mongodb            1/1     10m
```

**Assessment:** All pods Running and 1/1 Ready. Zero restarts. All deployments fully available. Namespace is healthy.

---

## Example 2: Running But Not Ready — Wrong MongoDB URI

```
$ kubectl get all -n cms

NAME                                        READY   STATUS    RESTARTS   AGE
pod/mongodb-0                               1/1     Running   0          65s
pod/client-service-65cf4799d9-btn59         0/1     Running   0          65s
pod/contract-service-64467c47f-4c2q9        1/1     Running   0          65s
pod/billing-service-86f68d7cc-tzqpv         1/1     Running   0          65s
pod/notification-service-5dbbc4499f-9rp5r   1/1     Running   0          65s
```

**Assessment:**
- `client-service` is `Running` but `0/1` Ready — readiness probe failing, app is alive but unhealthy
- MongoDB is healthy (1/1 Running, 0 restarts) — so MongoDB itself is fine
- Other services are healthy — so the problem is specific to client-service configuration
- Zero restarts — the app starts and stays alive, but the `/actuator/health` readiness probe reports DOWN
- **Next steps:** Check client-service readiness probe events (Phase 4), logs for MongoDB errors (Phase 5), and ConfigMap (Phase 6)

---

## Example 3: OOMKilled

```
$ kubectl get all -n cms

NAME                                        READY   STATUS             RESTARTS      AGE
pod/mongodb-0                               1/1     Running            0             5m
pod/client-service-6d4f8b7c9-x2k4p         1/1     Running            0             5m
pod/contract-service-5c7d9e8f1-m3n7q        1/1     Running            0             5m
pod/billing-service-7a2b3c4d5-p8r2s         0/1     CrashLoopBackOff   4 (18s ago)   5m
pod/notification-service-8e9f0a1b2-t5u9v    1/1     Running            0             5m
```

**Assessment:**
- `billing-service` is CrashLoopBackOff with 4 restarts
- Other services including MongoDB are healthy — problem is billing-service specific
- **Next steps:** Check `kubectl describe pod` for OOMKilled in last termination state (Phase 4), check resource limits (Phase 8)

---

## Example 4: Readiness Probe Failure

```
$ kubectl get all -n cms

NAME                                        READY   STATUS    RESTARTS   AGE
pod/mongodb-0                               1/1     Running   0          7m
pod/client-service-6d4f8b7c9-x2k4p         1/1     Running   0          7m
pod/contract-service-5c7d9e8f1-m3n7q        0/1     Running   0          7m
pod/billing-service-7a2b3c4d5-p8r2s         1/1     Running   0          7m
pod/notification-service-8e9f0a1b2-t5u9v    1/1     Running   0          7m
```

**Assessment:**
- `contract-service` is `Running` but `0/1` Ready — readiness probe is failing
- STATUS is Running (not CrashLoopBackOff), so the container itself is alive
- 0 restarts means it has never crashed — the issue is probe configuration, not application failure
- **Next steps:** Check probe configuration in `kubectl describe pod` (Phase 4), look for 404s in events
