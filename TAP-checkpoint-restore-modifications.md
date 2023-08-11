# Modifying TAP build cluster when using JVM Checkpoint/Restore

There are some modifications needed to be able to deploy apps using JVM Checkpoint/Restore on a TAP build cluster (full, iter or build profiles).
These modifications have been tested on a TAP v1.6.1 cluster using the `full` profile.

> Credit: Thanks to [Timo Salm](https://github.com/timosalm) for figuring all of this out and creating scripts for us.

## Changes for a "Build" cluster

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
