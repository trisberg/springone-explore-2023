# Modifying Knative configuration when using JVM Checkpoint/Restore

This also applies to TAP run clusters (full, iter or run profiles).

There are some modifications needed to be able to deploy apps using JVM Checkpoint/Restore on Knative.

> Credit: Thanks to Sébastien Deleuze, Timo Salm and Violeta Georgieva for figuring all of this out and also creating scripts for us.

> Note: This requires cluster admin permissions

## Define the namespace used for deploying apps

We need to define an `$APP_NAMESPACE` env var - setting this to `apps`, adjust this to the namespace you are using.

```sh
export APP_NAMESPACE=apps
```

Set your namespace for the current context, this will make it easier for the next steps:

```
kubectl config set-context --current --namespace=$APP_NAMESPACE
```

## Changes for a Knative or TAP "Run" cluster (full, iter or run profiles)

### Modify Knative configuration

Enable the ability to add needed capabilities to a Knative service/TAP workload

```sh
kubectl patch cm -n knative-serving config-features --type merge -p '{"data":{"kubernetes.containerspec-addcapabilities":"enabled","kubernetes.podspec-securitycontext":"enabled","kubernetes.podspec-persistent-volume-claim":"enabled","kubernetes.podspec-persistent-volume-write":"enabled","kubernetes.podspec-fieldref":"enabled"}}'
```

### Add cluster role and role bindings for NFS volume server

> Note: This uses the same namespace defined earlier in the `$APP_NAMESPACE` env var

```sh
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: create-pvc
rules:
- apiGroups: [""]
  resources: [persistentvolumeclaims]
  verbs: ['*']
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: default-create-pvc
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: create-pvc
subjects:
- kind: ServiceAccount
  name: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: create-pv
rules:
- apiGroups: [""]
  resources: [persistentvolumes]
  verbs: ['*']
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: default-create-pv
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: create-pv
subjects:
- kind: ServiceAccount
  name: default
  namespace: $APP_NAMESPACE
EOF
```

### Create the NFS server

> Note: You need to be able to run privileged containers on the cluster

```sh
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: "standard"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-server
spec:
  replicas: 1
  selector:
    matchLabels:
      role: nfs-server
  template:
    metadata:
      labels:
        role: nfs-server
    spec:
      containers:
      - name: nfs-server
        image: gcr.io/google_containers/volume-nfs:0.8
        ports:
          - name: nfs
            containerPort: 2049
          - name: mountd
            containerPort: 20048
          - name: rpcbind
            containerPort: 111
        securityContext:
          privileged: true
        volumeMounts:
          - mountPath: /exports
            name: nfs-pvc
      volumes:
        - name: nfs-pvc
          persistentVolumeClaim:
              claimName: nfs-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: nfs-server
spec:
  ports:
    - name: nfs
      port: 2049
    - name: mountd
      port: 20048
    - name: rpcbind
      port: 111
  selector:
    role: nfs-server
EOF
```
