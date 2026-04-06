# Config Inspection — Examples

## Example 1: Detecting Wrong MongoDB Hostname

```
$ kubectl get configmap -n cms
NAME                          DATA   AGE
client-service-config         2      10m
contract-service-config       2      10m
billing-service-config        2      10m
notification-service-config   2      10m

$ kubectl describe configmap client-service-config -n cms
Name:         client-service-config
Namespace:    cms
Data
====
SPRING_MONGODB_URI:
----
mongodb://mongo-primary:27017/cms-clients
SERVER_PORT:
----
8081
```

**Analysis:**
```
Expected: mongodb://mongodb:27017/cms-clients
Actual:   mongodb://mongo-primary:27017/cms-clients
                    ^^^^^^^^^^^^^^
```
- The hostname `mongo-primary` does not match any Kubernetes Service in the namespace
- The MongoDB Service is named `mongodb` (verify with `kubectl get svc -n cms`)
- **Root cause:** ConfigMap `client-service-config` has incorrect MongoDB hostname

---

## Example 2: All ConfigMaps Healthy

```
$ kubectl describe configmap client-service-config -n cms
Data
====
SPRING_MONGODB_URI:  mongodb://mongodb:27017/cms-clients
SERVER_PORT:              8081

$ kubectl describe configmap contract-service-config -n cms
Data
====
SPRING_MONGODB_URI:  mongodb://mongodb:27017/cms-contracts
SERVER_PORT:              8082

$ kubectl describe configmap billing-service-config -n cms
Data
====
SPRING_MONGODB_URI:  mongodb://mongodb:27017/cms-billing
SERVER_PORT:              8083

$ kubectl describe configmap notification-service-config -n cms
Data
====
SPRING_MONGODB_URI:  mongodb://mongodb:27017/cms-notifications
SERVER_PORT:              8084
```

**Analysis:** All ConfigMaps use correct MongoDB hostname (`mongodb`) and correct database names. Ports match expected values. Configuration is healthy.

---

## Example 3: Verifying Pod Receives ConfigMap Values

```
$ kubectl get pod client-service-6d4f8b7c9-x2k4p -n cms -o jsonpath='{.spec.containers[0].envFrom}' | python3 -m json.tool
[
    {
        "configMapRef": {
            "name": "client-service-config"
        }
    }
]
```

**Analysis:** Pod references `client-service-config` ConfigMap via `envFrom`. All keys in the ConfigMap will be injected as environment variables. Spring Boot's relaxed binding converts `SPRING_MONGODB_URI` to `spring.data.mongodb.uri` automatically.

---

## Example 4: Cross-Referencing Service Names

```
$ kubectl get svc -n cms
NAME                   TYPE        CLUSTER-IP       PORT(S)     AGE
mongodb                ClusterIP   None             27017/TCP   10m
client-service         ClusterIP   10.96.142.31     8081/TCP    10m
contract-service       ClusterIP   10.96.142.32     8082/TCP    10m
billing-service        ClusterIP   10.96.142.33     8083/TCP    10m
notification-service   ClusterIP   10.96.142.34     8084/TCP    10m
```

**Analysis:** The MongoDB Service name is `mongodb`. Any ConfigMap referencing a different hostname (e.g., `mongo-primary`, `mongo`, `mongodb-0`) for the MongoDB URI is incorrect. The URI must use `mongodb://mongodb:27017/...`.
