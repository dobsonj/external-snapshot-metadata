
# Changed Block Tracking (Dev Preview) notes

https://issues.redhat.com/browse/STOR-2364


## Use git branches with OCP deployment hacks

These instructions depend on deployment changes in these two git branches:
* https://github.com/dobsonj/external-snapshot-metadata/tree/ocp-hostpath-test
* https://github.com/dobsonj/csi-driver-host-path/tree/ocp-hostpath-test

Set environment variables to point to those branches:
```
export EXTERNAL_SNAPSHOT_METADATA_REPO=https://raw.githubusercontent.com/dobsonj/external-snapshot-metadata/refs/heads/ocp-hostpath-test
export CSI_DRIVER_HOST_PATH_REPO=https://raw.githubusercontent.com/dobsonj/csi-driver-host-path/refs/heads/ocp-hostpath-test
```


## Install host-path + external-snapshot-metadata on OpenShift

### Apply SnapshotMetadataService CRD

Before [this PR](https://github.com/openshift/cluster-csi-snapshot-controller-operator/pull/239) merges, you may deploy this CRD manually:
```
oc apply -f ${EXTERNAL_SNAPSHOT_METADATA_REPO}/client/config/crd/cbt.storage.k8s.io_snapshotmetadataservices.yaml
```

After it merges, you can get the CRD by setting DevPreviewNoUpgrade:
```
$ oc edit featuregate cluster
featuregate.config.openshift.io/cluster edited
$ oc get featuregate cluster -o json | jq .spec
{
  "featureSet": "DevPreviewNoUpgrade"
}
$ oc get crd | grep -i snapshotmetadataservice
snapshotmetadataservices.cbt.storage.k8s.io                       2025-06-24T17:55:55Z
```

### Create ClusterRoles

```
oc apply -f ${EXTERNAL_SNAPSHOT_METADATA_REPO}/deploy/snapshot-metadata-client-cluster-role.yaml
oc apply -f ${EXTERNAL_SNAPSHOT_METADATA_REPO}/deploy/snapshot-metadata-cluster-role.yaml
```

### Create Service and SnapshotMetadataService

Create the Service first:
```
oc apply -f ${EXTERNAL_SNAPSHOT_METADATA_REPO}/deploy/example/csi-driver/csi-driver-service.yaml
```

Make sure the csi-snapshot-metadata-certs secret was created:
```
$ oc get secrets -n openshift-cluster-csi-drivers csi-snapshot-metadata-certs
NAME                          TYPE                DATA   AGE
csi-snapshot-metadata-certs   kubernetes.io/tls   2      17s
```

Extract the certificate from the secret and copy it into a new SnapshotMetadataService manifest locally:
```
cat | sed "s/GENERATED_CA_CERT/$(oc get secrets -n openshift-cluster-csi-drivers csi-snapshot-metadata-certs -o json | jq -r '.data["tls.crt"]')/" > snapshotmetadataservice.yaml <<EOF
apiVersion: cbt.storage.k8s.io/v1alpha1
kind: SnapshotMetadataService
metadata:
  name: hostpath.csi.k8s.io
spec:
  address: csi-snapshot-metadata.openshift-cluster-csi-drivers.svc:6443
  caCert: GENERATED_CA_CERT
  audience: 005e2583-91a3-4850-bd47-4bf32990fd00
EOF
```

Apply the SnapshotMetadataService manifest:
```
oc apply -f snapshotmetadataservice.yaml
```

### Create host-path CSIDriver, StorageClass, VolumeSnapshotClass

```
oc apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: CSIDriver
metadata:
  name: hostpath.csi.k8s.io
spec:
  volumeLifecycleModes:
  - Persistent
  - Ephemeral
  podInfoOnMount: true
  fsGroupPolicy: File
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-hostpath-sc
provisioner: hostpath.csi.k8s.io
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-hostpath-snapclass
driver: hostpath.csi.k8s.io
deletionPolicy: Delete
EOF
```

### Deploy host-path with external-snapshot-metadata sidecar

```
oc apply -f ${CSI_DRIVER_HOST_PATH_REPO}/deploy/kubernetes-1.30/hostpath/csi-hostpath-plugin.yaml
```

Make sure the pod is running:
```
$ oc get pods -n openshift-cluster-csi-drivers csi-hostpathplugin-0
NAME                   READY   STATUS    RESTARTS   AGE
csi-hostpathplugin-0   8/8     Running   0          17s
```


## Test Client

### Deploy snapshot-metadata-tools pod in testns namespace

Create testns Namespace:
```
oc create namespace testns
```

Create ServiceAccount, ClusterRoleBinding, and Pod:
```
oc apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: snapshot-metadata-tools-sa
  namespace: testns
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: snapshot-metadata-tools-clusterrolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-snapshot-metadata-client-runner
subjects:
- kind: ServiceAccount
  name: snapshot-metadata-tools-sa
  namespace: testns
---
apiVersion: v1
kind: Pod
metadata:
  name: snapshot-metadata-tools
  namespace: testns
spec:
  serviceAccountName: snapshot-metadata-tools-sa
  containers:
  - name: tools
    image: quay.io/jdobson/snapshot-metadata-tools:latest
    command:
    - /bin/sh
    - -c
    - "tail -f /dev/null"
    securityContext:
      allowPrivilegeEscalation: false
      privileged: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
EOF
```

Make sure the pod is running:
```
$ oc get pods -n testns
NAME                      READY   STATUS    RESTARTS   AGE
snapshot-metadata-tools   1/1     Running   0          13s
```

### Create PVC

```
oc apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-pvc
  namespace: testns
spec:
  accessModes:
  - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-hostpath-sc
EOF
```

### Write some data

Create an application pod:
```
oc apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-raw
  namespace: testns
  labels:
    name: busybox-test
spec:
  restartPolicy: Always
  containers:
    - image: gcr.io/google_containers/busybox
      command:
      - /bin/sh
      - -c
      - "tail -f /dev/null"
      name: busybox
      volumeDevices:
        - name: vol
          devicePath: /dev/loop3
      securityContext:
        allowPrivilegeEscalation: false
        privileged: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
          - ALL
  volumes:
    - name: vol
      persistentVolumeClaim:
        claimName: csi-pvc
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
EOF
```

Wait for pod to start:
```
$ oc get pods -n testns pod-raw
NAME      READY   STATUS    RESTARTS   AGE
pod-raw   1/1     Running   0          48s
```

Write some data:
```
oc exec -n testns pod-raw -- dd if=/dev/urandom of=/dev/loop3 bs=4K count=1 seek=1 conv=notrunc
oc exec -n testns pod-raw -- dd if=/dev/urandom of=/dev/loop3 bs=4K count=1 seek=3 conv=notrunc
oc exec -n testns pod-raw -- dd if=/dev/urandom of=/dev/loop3 bs=4K count=1 seek=5 conv=notrunc
oc exec -n testns pod-raw -- dd if=/dev/urandom of=/dev/loop3 bs=4K count=1 seek=7 conv=notrunc
oc exec -n testns pod-raw -- dd if=/dev/urandom of=/dev/loop3 bs=4K count=1 seek=9 conv=notrunc
```

### Create VolumeSnapshot

```
oc apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: test-snapshot1
  namespace: testns
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source:
    persistentVolumeClaimName: csi-pvc
EOF
```

Wait for it to be ready
```
$ oc get volumesnapshot -n testns
NAME             READYTOUSE   SOURCEPVC   SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS            SNAPSHOTCONTENT                                    CREATIONTIME   AGE
test-snapshot1   true         csi-pvc                             1Gi           csi-hostpath-snapclass   snapcontent-d9e502dd-aea5-4917-b6fd-7b575087a3e9   2m             2m
```

### Use snapshot-metadata-lister to see the allocated blocks

```
$ oc exec -n testns snapshot-metadata-tools -- snapshot-metadata-lister -n testns -s test-snapshot1
Record#   VolCapBytes  BlockMetadataType   ByteOffset     SizeBytes
------- -------------- ----------------- -------------- --------------
      1     1073741824      FIXED_LENGTH           4096           4096
      1     1073741824      FIXED_LENGTH          12288           4096
      1     1073741824      FIXED_LENGTH          20480           4096
      1     1073741824      FIXED_LENGTH          28672           4096
      1     1073741824      FIXED_LENGTH          36864           4096
```

### Check diff between two snapshots

Write some more data:
```
oc exec -n testns pod-raw -- dd if=/dev/urandom of=/dev/loop3 bs=4K count=5 seek=15 conv=notrunc
```

Create another snapshot:
```
oc apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: test-snapshot2
  namespace: testns
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source:
    persistentVolumeClaimName: csi-pvc
EOF
```

Wait for it to be ready:
```
$ oc get volumesnapshot -n testns
NAME             READYTOUSE   SOURCEPVC   SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS            SNAPSHOTCONTENT                                    CREATIONTIME   AGE
test-snapshot1   true         csi-pvc                             1Gi           csi-hostpath-snapclass   snapcontent-c07dccac-3031-402d-a718-8643a8abba87   3m46s          3m46s
test-snapshot2   true         csi-pvc                             1Gi           csi-hostpath-snapclass   snapcontent-61be4ca0-f51a-42e8-ae50-068afcaac614   9s             9s
```

Use snapshot-metadata-lister to see the incremental changes:
```
$ oc exec -n testns snapshot-metadata-tools -- snapshot-metadata-lister -n testns -p test-snapshot1 -s test-snapshot2
Record#   VolCapBytes  BlockMetadataType   ByteOffset     SizeBytes
------- -------------- ----------------- -------------- --------------
      1     1073741824      FIXED_LENGTH          61440           4096
      1     1073741824      FIXED_LENGTH          65536           4096
      1     1073741824      FIXED_LENGTH          69632           4096
      1     1073741824      FIXED_LENGTH          73728           4096
      1     1073741824      FIXED_LENGTH          77824           4096
```


## References

* https://github.com/kubernetes-csi/csi-driver-host-path/blob/master/docs/deploy-1.17-and-later.md
* https://github.com/kubernetes-csi/csi-driver-host-path/blob/master/docs/example-snapshot-metadata.md
* https://github.com/kubernetes-csi/external-snapshot-metadata/blob/main/deploy/README.md
* https://github.com/kubernetes-csi/external-snapshot-metadata/blob/main/deploy/example/csi-driver/README.md
* https://github.com/kubernetes-csi/external-snapshot-metadata/blob/main/deploy/example/backup-app/README.md
* https://github.com/kubernetes-csi/external-snapshot-metadata/blob/main/client/apis/snapshotmetadataservice/v1alpha1/types.go
* https://github.com/kubernetes/enhancements/blob/master/keps/sig-storage/3314-csi-changed-block-tracking/README.md

