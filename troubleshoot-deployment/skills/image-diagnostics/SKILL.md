# Image Diagnostics

Rules and patterns for diagnosing container image pull failures, tag issues, and registry access problems in Kubernetes.

## Version: 1.0.0

---

## Rules

### ID01: ImagePullBackOff Means the Image Is Not Available

`ImagePullBackOff` or `ErrImagePull` in pod status means Kubernetes cannot pull the container image. Common causes:
- Image doesn't exist in the registry
- Wrong image name or tag
- Private registry without pull credentials
- Image not loaded into minikube's local cache

### ID02: Check `imagePullPolicy` Setting

- `IfNotPresent` — only pull if not already cached locally. **Required for minikube with local images.**
- `Always` — pull from registry every time. Will fail for local-only images.
- `Never` — never pull, use only local cache.

If using minikube with locally built images, `imagePullPolicy` must be `IfNotPresent` or `Never`.

### ID03: Verify Images Are Loaded in Minikube

For minikube, images built on the host are not automatically available inside the cluster:
```bash
minikube image ls | grep cms
```

If images are missing, they need to be loaded:
```bash
minikube image load cms/<service-name>:latest
```

### ID04: Check Image Reference Format

Expected format for local images: `cms/<service-name>:latest`

Common mistakes:
- Missing tag: `cms/client-service` (defaults to `:latest` but can be ambiguous)
- Wrong prefix: `docker.io/cms/client-service:latest` (tries to pull from Docker Hub)
- Typo in service name: `cms/clent-service:latest`

### ID05: Verify Image ID Exists

```bash
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\t"}{.status.containerStatuses[0].imageID}{"\n"}{end}'
```

If `imageID` is empty, the image was never successfully pulled. If `imageID` exists, the image is available and the problem is elsewhere.

---

## Common Pitfalls

- **Building images outside minikube context**: `docker build` on the host doesn't make images available inside minikube. Use `minikube image load` or `eval $(minikube docker-env)` before building.
- **Using `:latest` tag with `imagePullPolicy: Always`**: This will try to pull from a registry and fail for local images.
- **Confusing ImagePullBackOff with CrashLoopBackOff**: ImagePullBackOff means the container never started. CrashLoopBackOff means it started but crashed. Completely different problems.
