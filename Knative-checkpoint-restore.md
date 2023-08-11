# Using JVM Checkpoint/Restore on Knative

## Prepare your Knative cluster

There are some modifications needed to be able to deploy apps using JVM Checkpoint/Restore on Knative.

See the [Modifying Knative/Tap run cluster configuration when using JVM Checkpoint/Restore](Knative-checkpoint-restore-modifications.md) doc.

## Clone a `hello-world` sample app

We can use the sample app available at [https://github.com/trisberg/hello-world](https://github.com/trisberg/hello-world), let's just clone it and change to the `hello-world` directory:

```sh
git clone https://github.com/trisberg/hello-world
cd hello-world
```

## Build the app

The app comes with a `Dockerfile` and a `build.sh` script. You can run the build on both ARM systems as well as Intel systems but we need to specify the architecture for the target cluster. The `TARGET_ARCH` env var should be set to either `amd64` to `arm64`:

```sh
export TARGET_ARCH=amd64
```

After this, we also set the env var `REGISTRY_PREFIX` to our desired registry/account:

```sh
export REGISTRY_PREFIX=docker.io/springdeveloper
```

Now, we can run the build script:


```sh
./build.sh
```

## Deploy the Knative Service

The project has a Knative Service manifest in `cat knative/service.yaml`

```yaml
#@ load("@ytt:data", "data")
---
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello-world
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/min-scale: "0"
    spec:
      containers:
      - name: workload
        image: #@ data.values.registry_prefix+"/hello-world:"+data.values.arch+"-"+data.values.version
        imagePullPolicy: Always
        env:
        - name: REVISION_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.labels['serving.knative.dev/revision']
        - name: REVISION_UID
          valueFrom:
            fieldRef:
              fieldPath: metadata.labels['serving.knative.dev/revisionUID']
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: CRAC_FILES_DIR
          value: /var/crac/${REVISION_NAME}/${REVISION_UID}/${NODE_NAME}
        securityContext:
          capabilities:
            add:
            - CHECKPOINT_RESTORE
            - NEW_ADMIN
            - SYS_PTRACE
          runAsUser: 0
        volumeMounts:
        - mountPath: /var/crac
          name: crac-cache
      volumes:
      - name: crac-cache
        persistentVolumeClaim:
          claimName: hello-world-knative
```

It has some YTT variables that will get filled in when we deploy.

There is also a `knative/checkpoint-pvc.yaml` file to create a PVC for storing the checkpoint files:

```yaml
#@ load("@ytt:data", "data")
---
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    kapp.k14s.io/update-strategy: skip
  name: hello-world-knative
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: #@ data.values.nfs_server_ip
    path: "/"
  storageClassName: "nfs"
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: hello-world-knative
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: "nfs"
  resources:
    requests:
      storage: 500Mi
```

To deploy this, run:

```sh
ytt -f knative \
  --data-value=registry_prefix=$REGISTRY_PREFIX \
  --data-value=arch=$TARGET_ARCH \
  --data-value=version=$(cat VERSION) \
  --data-value=nfs_server_ip=$(kubectl get service nfs-server -o jsonpath='{.spec.clusterIP}') | \
kubectl create -f -
```

## Accessing the app deployed to your cluster

Determine the URL to use for the accessing the app by running:

```bash
kn service list
```

or, using kubectl:

```bash
kubectl get service.serving.knative.dev
```

To access the deployed app use the URL shown under "Workload Knative Services".

```bash
APP_URL=<Knative-service-URL>
```

Then, use `curl` or some other utility to access the URL:

```bash
curl $APP_URL
```

## Check the logs from the app

```sh
kubectl logs deploy/hello-world-00001-deployment
```