# Image Diagnostics — Examples

## Example 1: Healthy Image Status

```
$ kubectl get pods -n cms -o custom-columns='NAME:.metadata.name,IMAGE:.spec.containers[0].image,IMAGE_ID:.status.containerStatuses[0].imageID'

NAME                                       IMAGE                            IMAGE_ID
mongodb-0                                  mongo:7.0                        docker.io/library/mongo@sha256:abc123...
client-service-6d4f8b7c9-x2k4p            cms/client-service:latest        docker.io/cms/client-service@sha256:def456...
contract-service-5c7d9e8f1-m3n7q           cms/contract-service:latest      docker.io/cms/contract-service@sha256:ghi789...
billing-service-7a2b3c4d5-p8r2s            cms/billing-service:latest       docker.io/cms/billing-service@sha256:jkl012...
notification-service-8e9f0a1b2-t5u9v       cms/notification-service:latest  docker.io/cms/notification-service@sha256:mno345...
```

**Analysis:** All images have valid IMAGE_ID entries — images were successfully pulled/loaded. No image-related issues.

---

## Example 2: ImagePullBackOff

```
$ kubectl get pods -n cms
NAME                                       READY   STATUS             RESTARTS   AGE
client-service-6d4f8b7c9-x2k4p            0/1     ImagePullBackOff   0          3m

$ kubectl describe pod client-service-6d4f8b7c9-x2k4p -n cms
...
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  3m                 default-scheduler  Successfully assigned...
  Normal   Pulling    2m (x3 over 3m)   kubelet            Pulling image "cms/client-service:latest"
  Warning  Failed     2m (x3 over 3m)   kubelet            Failed to pull image "cms/client-service:latest": image not found
  Warning  Failed     2m (x3 over 3m)   kubelet            Error: ErrImagePull
  Normal   BackOff    30s (x5 over 2m)  kubelet            Back-off pulling image "cms/client-service:latest"
```

**Analysis:**
- Image `cms/client-service:latest` not found — it hasn't been loaded into minikube
- The `imagePullPolicy: IfNotPresent` setting means Kubernetes tried to find it locally first, then attempted to pull — both failed
- **Fix:** Build the image and load it: `docker build -t cms/client-service:latest workspace/client-service && minikube image load cms/client-service:latest`

---

## Example 3: Checking Minikube Image Cache

```
$ minikube image ls | grep cms
docker.io/cms/client-service:latest
docker.io/cms/contract-service:latest
docker.io/cms/notification-service:latest
```

**Analysis:** `billing-service` image is missing from minikube's cache while the other 3 services are present. If billing-service pod shows ImagePullBackOff, this confirms the image wasn't loaded.
