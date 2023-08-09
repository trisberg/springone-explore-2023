# Modifying TAP cluster when using JVM Checkpoint/Restore

There are some modifications needed to be able to apps using JVM Checkpoint/Restore on a TAP cluster.
These modifications have been tested on a TAP v1.6.1 cluster using the `full` profile.

> Note: This is experimental and should only be used on deployments used for testing or exploration.

## For a TAP cluster with pod security enabled via Kyverno

> Note: This requires cluster admin permissions

Remove the security policy enforcements for the `apps` namespace using the following (adjust the namespace as needed):

```sh
export APP_NAMESPACE=apps
kubectl patch configmap kyverno -n kyverno --type merge -p '{"data":{"webhooks":"[{\"namespaceSelector\": {\"matchExpressions\": [{\"key\":\"kubernetes.io/metadata.name\",\"operator\":\"NotIn\",\"values\":[\"kyverno\"]},{\"key\":\"kubernetes.io/metadata.name\",\"operator\":\"NotIn\",\"values\":[\"'$APP_NAMESPACE'\"]}]}}]"}}'
kubectl label ns $APP_NAMESPACE pod-security.kubernetes.io/enforce-
```
## Set your namespace for the current context

This will make it easier for the next steps.

```
kubectl config set-context --current --namespace=$APP_NAMESPACE
```

## Changes for a "Run" cluster (full, iter or run profiles)

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

## Changes fFor a "Build" cluster (full, iter or build profiles)

### Update configuration and exclude the convention-template and config-template

On `macOS` run:

```sh
UPDATED_TAP_VALUES=$(kubectl get secret tap-tap-install-values -n tap-install -o jsonpath='{.data.values\.yaml}' | base64 -d | grep -v '.*#! ' | yq '.ootb_templates += {"excluded_templates": ["convention-template","config-template"]}'| base64)
```

On `Linux` run:

```sh
UPDATED_TAP_VALUES=$(kubectl get secret tap-tap-install-values -n tap-install -o jsonpath='{.data.values\.yaml}' | base64 -d | grep -v '.*#! ' | yq '.ootb_templates += {"excluded_templates": ["convention-template","config-template"]}'| base64 -w0)
```

Then run this:

```sh
kubectl patch secret tap-tap-install-values -n tap-install --type json -p="[{\"op\" : \"replace\" ,\"path\" : \"/data/values.yaml\" ,\"value\" : ${UPDATED_TAP_VALUES}}]"
tanzu package installed kick tap -n tap-install -y
```

### Add configuration with overlay for the convention-template and config-template

First, look up NFS server IP

```sh
NFS_SERVER_IP=$(kubectl get service nfs-server -o jsonpath='{.spec.clusterIP}')
```

Next, re-create the config-template:

```sh
cat <<EOF | kubectl apply -f -
apiVersion: carto.run/v1alpha1
kind: ClusterConfigTemplate
metadata:
  name: config-template
spec:
  configPath: .data
  healthRule:
    alwaysHealthy: {}
  lifecycle: mutable
  ytt: |
    #@ load("@ytt:data", "data")
    #@ load("@ytt:yaml", "yaml")

    #@ def merge_labels(fixed_values):
    #@   labels = {}
    #@   if hasattr(data.values.workload.metadata, "labels"):
    #@     labels.update(data.values.workload.metadata.labels)
    #@   end
    #@   labels.update(fixed_values)
    #@   return labels
    #@ end

    #@ def pv():
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      annotations:
        kapp.k14s.io/update-strategy: skip
      name: #@ data.values.workload.metadata.name
    spec:
      capacity:
        storage: 500Mi
      accessModes:
        - ReadWriteMany
      nfs:
        server: $NFS_SERVER_IP
        path: "/"
    #@ end

    #@ def pvc():
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: #@ data.values.workload.metadata.name
    spec:
      accessModes:
        - ReadWriteMany
      storageClassName: ""
      resources:
        requests:
          storage: 500Mi
    #@ end

    #@ def delivery():
    apiVersion: serving.knative.dev/v1
    kind: Service
    metadata:
      name: #@ data.values.workload.metadata.name
      #! annotations NOT merged because knative annotations would be invalid here
      annotations:
        ootb.apps.tanzu.vmware.com/servicebinding-workload: "true"
        ootb.apps.tanzu.vmware.com/apidescriptor-ref: "true"
        kapp.k14s.io/change-rule: "upsert after upserting servicebinding.io/ServiceBindings"
      labels: #@ merge_labels({ "app.kubernetes.io/component": "run", "carto.run/workload-name": data.values.workload.metadata.name })
    spec:
      template: #@ data.values.config
    #@ end

    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: #@ data.values.workload.metadata.name
      labels: #@ merge_labels({ "app.kubernetes.io/component": "config" })
    data:
      delivery.yml: #@ yaml.encode(delivery()) + "---\n" + yaml.encode(pv()) + "---\n" + yaml.encode(pvc())
EOF
```

Finally, re-create the convention-template:

```sh
cat <<EOF | kubectl apply -f -
apiVersion: carto.run/v1alpha1
kind: ClusterConfigTemplate
metadata:
  name: convention-template
spec:
  configPath: .status.template
  healthRule:
    singleConditionType: Ready
  lifecycle: mutable
  params:
  - default: default
    name: serviceAccount
  ytt: |
    #@ load("@ytt:data", "data")

    #@ def param(key):
    #@   if not key in data.values.params:
    #@     return None
    #@   end
    #@   return data.values.params[key]
    #@ end

    #@ def merge_labels(fixed_values):
    #@   labels = {}
    #@   if hasattr(data.values.workload.metadata, "labels"):
    #@     labels.update(data.values.workload.metadata.labels)
    #@   end
    #@   labels.update(fixed_values)
    #@   return labels
    #@ end

    #@ def build_fixed_annotations():
    #@   fixed_annotations = { "developer.conventions/target-containers": "workload" }
    #@   if param("debug"):
    #@     fixed_annotations["apps.tanzu.vmware.com/debug"] = param("debug")
    #@   end
    #@   if param("live-update"):
    #@     fixed_annotations["apps.tanzu.vmware.com/live-update"] = param("live-update")
    #@   end
    #@   return fixed_annotations
    #@ end

    #@ def merge_annotations(fixed_values):
    #@   annotations = {}
    #@   if hasattr(data.values.workload.metadata, "annotations"):
    #@     # DEPRECATED: remove in a future release
    #@     annotations.update(data.values.workload.metadata.annotations)
    #@   end
    #@   if type(param("annotations")) == "dict" or type(param("annotations")) == "struct":
    #@     annotations.update(param("annotations"))
    #@   end
    #@   annotations.update(fixed_values)
    #@   return annotations
    #@ end

    apiVersion: conventions.carto.run/v1alpha1
    kind: PodIntent
    metadata:
      name: #@ data.values.workload.metadata.name
      labels: #@ merge_labels({ "app.kubernetes.io/component": "intent" })
    spec:
      serviceAccountName: #@ data.values.params.serviceAccount
      template:
        metadata:
          annotations: #@ merge_annotations(build_fixed_annotations())
          labels: #@ merge_labels({ "app.kubernetes.io/component": "run", "carto.run/workload-name": data.values.workload.metadata.name })
        spec:
          serviceAccountName: #@ data.values.params.serviceAccount
          containers:
            - name: workload
              image: #@ data.values.image
              securityContext:
                runAsUser: 0
                capabilities:
                  add:
                  - CHECKPOINT_RESTORE
                  - NET_ADMIN
                  - SYS_PTRACE
              env:
              - name: REVISION_NAME
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.labels['serving.knative.dev/revision']
              - name: REVISION_UID
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.labels['serving.knative.dev/revisionUID']
              - name: CHECKPOINT_RESTORE_FILES_DIR
                value: /var/checkpoint-restore/\$(REVISION_NAME)/\$(REVISION_UID)
              #@ if hasattr(data.values.workload.spec, "env"):
              #@ for var in data.values.workload.spec.env:
              - name: #@ var.name
                #@ if/end hasattr(var, "value"):
                value: #@ var.value
                #@ if/end hasattr(var, "valueFrom"):
                valueFrom: #@ var.valueFrom
              #@ end
              #@ end
              #@ if/end hasattr(data.values.workload.spec, "resources"):
              resources: #@ data.values.workload.spec["resources"]
              volumeMounts:
              - name: checkpoint-restore-cache
                mountPath: /var/checkpoint-restore
          volumes:
          - name: checkpoint-restore-cache
            persistentVolumeClaim:
              claimName: #@ data.values.workload.metadata.name
EOF
```
