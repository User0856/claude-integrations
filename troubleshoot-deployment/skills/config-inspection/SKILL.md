# Config Inspection

Rules and patterns for verifying ConfigMaps, environment variables, and application configuration in Kubernetes deployments.

## Version: 1.0.0

---

## Rules

### CI01: Read ConfigMaps Before Checking Logs

Configuration errors are the most common cause of deployment failures. Always inspect ConfigMaps early in the diagnostic process — they frequently contain the root cause.

```bash
kubectl get configmap -n <namespace>
kubectl describe configmap <name> -n <namespace>
```

### CI02: Verify MongoDB URI Format

The correct MongoDB URI format for services in the same namespace:
```
mongodb://mongodb:27017/<database-name>
```

Where:
- `mongodb` is the Kubernetes Service name for the MongoDB StatefulSet
- `27017` is the standard MongoDB port
- `<database-name>` is service-specific

Expected database names for this system:
| Service | Database |
|---------|----------|
| client-service | `cms-clients` |
| contract-service | `cms-contracts` |
| billing-service | `cms-billing` |
| notification-service | `cms-notifications` |

### CI03: Cross-Reference ConfigMap Values Against Service Names

The hostname in `SPRING_MONGODB_URI` must match an actual Kubernetes Service in the namespace. Common mistakes:
- `mongo-primary` instead of `mongodb` — wrong hostname
- `localhost` instead of `mongodb` — won't work in Kubernetes
- `mongodb-0.mongodb` — StatefulSet pod DNS, works but fragile
- Missing port or wrong port number

### CI04: Verify Environment Variables Are Injected

Check that pods actually receive their ConfigMap values:
```bash
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].envFrom}'
```

If `envFrom` is empty, the deployment may not reference the ConfigMap. Check the deployment spec.

### CI05: Check for ConfigMap Naming Mismatches

A deployment referencing `client-service-config` ConfigMap will fail to start if the actual ConfigMap is named differently. Compare:
- What the Deployment `.spec.template.spec.containers[].envFrom[].configMapRef.name` expects
- What ConfigMaps actually exist: `kubectl get configmap -n <namespace>`

### CI06: SERVER_PORT Must Match Container and Probe Ports

Three places define the port, and they must all agree:
1. `SERVER_PORT` in ConfigMap → what Spring Boot listens on
2. `containerPort` in Deployment → what Kubernetes knows about
3. Probe `port` in Deployment → what health checks hit

A mismatch means probes fail even though the application is healthy.

### CI07: Compare Expected vs Actual Values

When diagnosing a configuration issue, always present the comparison:
```
Expected: mongodb://mongodb:27017/cms-clients
Actual:   mongodb://mongo-primary:27017/cms-clients
                    ^^^^^^^^^^^^^^
                    Wrong hostname — 'mongo-primary' does not match any Service in the namespace
```

---

## Common Pitfalls

- **Assuming ConfigMaps are correct because they exist**: A ConfigMap can exist with wrong values. Always read and verify the actual content.
- **Missing environment variable precedence**: If a Deployment sets both `env` and `envFrom`, explicit `env` entries override `envFrom` values. Check both.
- **Not checking if ConfigMap was updated after deployment**: ConfigMap changes are NOT automatically picked up by running pods. The deployment must be restarted for changes to take effect.
- **Ignoring database name in URI**: The hostname may be correct but the database name wrong, causing the application to connect to a different (empty) database.
