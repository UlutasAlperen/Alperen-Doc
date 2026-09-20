---
title: "kubernetes-persistent-volumes"
weight: 14
---
## Persistent Volumes (PV)

Instead of simply adding a volume to a deployment, a persistent volume is a cluster-level resource that is created separately from the pod and then attached to the pod. It's similar to a ConfigMap in that way.

PVs can be created statically or dynamically.

- Static PVs are created manually by a cluster admin
- Dynamic PVs are created automatically when a pod requests a volume that doesn't exist yet

Generally speaking, and especially in the cloud-native world, we want to use dynamic PVs. It's less work and more flexible.

## Persistent Volume Claims (PVC)

A [persistent volume claim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) is a _request_ for a persistent volume. When using dynamic provisioning, a PVC will automatically create a PV if one doesn't exist that matches the claim.

The PVC is then attached to a pod, just like a volume would be.

## Assignment

Create a new file called `api-pvc.yaml` and add the following:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: xxx
spec:
  accessModes:
    - xxx
  resources:
    requests:
      storage: xxx
```

Add the following properties:

- `metadata/name`: `synergychat-api-pvc`
- `spec/accessModes`: An array with one entry:
    - `ReadWriteOnce`
- `spec/resources/requests/storage`: `1Gi`

This creates a new PVC called `synergychat-api-pvc` with a few properties that can be read from and written to by multiple pods at the same time. It also requests 1GB of storage.

Apply the PVC.

Run both of these commands:

```bash
kubectl get pvc
kubectl get pv
```

You should see that a new PV was created automatically!

Now _delete_ the PVC:

```bash
kubectl delete pvc <pvc-name>
```

Make sure both the PVC and PV are gone:

```bash
kubectl get pvc
kubectl get pv
```
---
# Attach Persistence

So far all we've done is create an empty persistent volume. Let's get the `api` application to use it.

## Assignment

Use your `crawler` deployment and the [docs](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) as a reference. Create a new volume in the api-deployment referencing your pvc:

```yaml
volumes:
  - name: synergychat-api-volume
    persistentVolumeClaim:
      claimName: synergychat-api-pvc
```

Then mount it in the container under the `/persist` directory:

```yaml
volumeMounts:
  - name: synergychat-api-volume
    mountPath: /persist
```

Update the `API_DB_FILEPATH` environment variable you added earlier to instead use the new mount path: `/persist/db.json`

Apply the changes, then check to make sure all your pods are healthy:

```bash
kubectl get pods
```

With your tunnel running (`minikube tunnel -c`), open `http://synchat.internal/` in your browser.

1. Send some messages.
2. Delete the `api` pod.
3. Once the new pod is running, refresh the page and make sure your messages are still there. If they are, your persistent volume is working!

`api-deployment.yaml`

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: synergychat-api
  labels:
    app: synergychat-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: synergychat-api
  template:
    metadata:
      labels:
        app: synergychat-api
    spec:
      containers:
        - name: synergychat-api
          image: bootdotdev/synergychat-api:latest
          volumeMounts:
            - name: synergychat-api-volume
              mountPath: /persist
          env:
            - name: API_PORT
              valueFrom:
                configMapKeyRef:
                  name: synergychat-api-configmap
                  key: API_PORT
            - name: API_DB_FILEPATH
              valueFrom:
                configMapKeyRef:
                  name: synergychat-api-configmap
                  key: API_DB_FILEPATH
      volumes:
        - name: synergychat-api-volume
          persistentVolumeClaim:
            claimName: synergychat-api-pvc
---
```

`api-configmap.yaml`

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: synergychat-api-configmap
data:
  API_PORT: "8080"
  API_DB_FILEPATH: /persist/db.json
---
```
for more [statefulsets](kubernetes-statefulsets/)
