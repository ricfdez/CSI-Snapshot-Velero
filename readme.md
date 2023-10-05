# CSI Snapshotter

This tool is a feature of CSI that allows to take point-in-time snapshots of a volume for restoring it later. This component manages these snapshots in the cluster, and it provides a way to create/delete/and manage snapshots of volumes attached to containers.

- [CSI Snapshotter](#csi-snapshotter)
  - [Key points:](#key-points)
    - [Prerequisites:](#prerequisites)
  - [Prod implementation:](#prod-implementation)
    - [Webhook](#webhook)
    - [Scheduling](#scheduling)
      - [Test case:](#test-case)
      - [Deploy application writing data to a PVC, using a DB. (Ghost blog application)](#deploy-application-writing-data-to-a-pvc-using-a-db-ghost-blog-application)
      - [CSI Snapshot plugin and Velero.](#csi-snapshot-plugin-and-velero)
      - [Caveats](#caveats)
    - [References:](#references)




## Key points:

- **Point-in-time-snapshots:** CSI snapshots are copies in time of a volume. They capture state of a volume at specific moment.
- **Volume Snapshot Classes:** You can define deifferent classes for colume snapshots with different characteristics (how long to keep/stora providers..)
- **Volume restore:** Allows to restore volumes to a previous state from a snapshot. Also useful for data cloning.
- **CRDS:** To use it, it is necessary to implement its CRDS, like VOlumeSnapshot CR to specify how to configure the snapshots
- **Validation Webhook** The validation webhook is an extra mechanism that validates snapshots. This feature has been on the BETA stage in k8s since 1.17 in which a gap in validation when CR's where created. The gap was resolved using this webhook. For demo purposes is not necessary, but it is for prod, day to day operations.

### Prerequisites:
    - EKS 1.20+
    - kustomize
    - EBS-CSI storage driver.
    - External Snapshotter installed:
      - CRDS
    - patch velero to use CSI snapshot ```kubectl patch deployment velero-cluster-backup -n nite-velero --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--features=EnableCSI"}]'
```

## Quick Start Guide:

Clone the snapshotter repo https://github.com/kubernetes-csi/external-snapshotter

kubectl kustomize client/config/crd | kubectl create -f -
https://github.com/kubernetes-csi/external-snapshotter/tree/master/client/config/crd


### Setup simple test application

```
cat << EOF | kubectl apply -f -
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: ebs-sc-test
provisioner: ebs.csi.aws.com
volumeBindingMode: Immediate
EOF
```
``` 
cat <<EOF | kubectl apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-test-vsc
driver: ebs.csi.aws.com
deletionPolicy: Delete
EOF
```
- We deploy a simple pod that writes into a txt file along a pvc for it
```
cat <<EOF | kubectl apply -f - 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:k 
  name: ebs-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc-test
  resources:
    requests:
      storage: 4Gi
EOF
```
```
cat <<EOF | kubectl apply -f - 
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: centos
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(date -u) >> /data/out.txt; sleep 5; done"]
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: ebs-claim
EOF
```
At this point we have a basic pod that is writing data over to a PVC we designed.

- Next, we create a volume snapshot referencing the persistenvolumeclaim we provisioned earlier:
 
 ```
cat <<EOF | kubectl apply -f - 
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: ebs-volume-snapshot
spec:
  volumeSnapshotClassName: csi-test-vsc
  source:
    persistentVolumeClaimName: ebs-claim
EOF
```
This volume snapshot should show reflect while checking the CR volume snapshot ```kubectl get volumesnapshot``` This can also be checked at storage level in AWS.

### Validation:

- Remove the pod ```kubectl delete pod app```
- Remove the pvc ``` kubectl delete pvc ebs-claim ``` after this command the application will be gone from your cluster.

Restoring the application will require us to issue a ```PersistentVolumeClaim``` referencing the ```VolumeSnapshot``` in its datasource key:

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-snapshot-restored-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc-test
  resources:
    requests:
      storage: 10Gi
  dataSource:
    name: ebs-volume-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
EOF
```

And a pod with the new claim

``` yaml

cat << EOF | kubectl apply -f - 
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: centos
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(date -u) >> /data/out.txt; sleep 5; done"]
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: ebs-snapshot-restored-claim
EOF
```

Once the pod starts, it should have picked the EBS voume created from the snapshot in the preivou step. In the Amazon EC2 volumes tab it should display. Finally we can verify the pod is accesing the data from the volume attached to the pod:

```kubectl exec app -- cat /data/out.txt | head -n 10 ```

## Prod implementation:

Part of the downsides of using the snapshotter alone it is that it only provides the concept of a volume snapshot in the cluster, and there is no method to schedule a backup or use the snapshot feature as a restore point object to help keeping consistency in larger backups.
A part from this a production implementation provides the following features:

- Snapshot automation based on backup schedule
- Customize restore points
- Ensure data consistency.

To address the first 2 points and last, we can use Velero and the snapshotter plugin to incorporate snapshot request into backup schedules, in backup objects in velero CRs.

Ensuring data consistency, requires the use of a webhook that validates the volumesnapshot objects upon creation and before there is any write operation to disks.
The Helm chart takes care of installing a certificate, and a validation webhook object that takes care of making sure no bad snapshots are created.

### Webhook

Example of webhook denying incorrect volume snapshot:


```
cat <<EOF | kubectl apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: new-snapshot-demo-v1
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass-v1
  source: # Only one of the two fields should be set for a snapshot. Therefore this snapshot is invalid.
    persistentVolumeClaimName: pvc 
    volumeSnapshotContentName: vsc 
pipe heredoc> EOF
The VolumeSnapshot "new-snapshot-demo-v1" is invalid: <nil>: Invalid value: "": "spec.source" must validate one and only one schema (oneOf). Found 2 valid alternatives

```
### Scheduling
To introduce Velero into CSI snapshots, we need the following prerequisites to be met:

1. Kubernetes cluster version 1.20+
2. CSI drivers enabled cluster (aws-ebs-cs-driver)
3. In cross cluster scenarios it is necessary that on the source cluster, the CSI driver is named like in the destination so snapshots can be portable

#### Test case:

We will create an applicaton that relies on a DB in the backend that we will backup using ebs snapshots, and Velero as our DR solution.
To start we will provision a Volume snapshot class for our driver (in this case - AWS EBS CSI) needs to be provisioned (like in our previous example), setting this volume snapshot class would create a default behavior:


``` bash
cat <<EOF | kubectl apply -f -

apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: backup-snapshotclass-velero
  namespace: kube-system
  labels:
    velero.io/csi-volumesnapshot-class: "true"
driver: ebs.csi.aws.com
deletionPolicy: Delete
EOF
```
#### Deploy application writing data to a PVC, using a DB. (Ghost blog application)

1. Add Bitnami app
```Bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```
2. Create a values file for the app chart 

```bash
#we lay out our config file for the app
cat <<EOF > ghost-values.yaml
ghostUsername: admin
ghostPassword: "0123456789"
ghostEmail: admin@example.com

mariadb:
    auth:
        rootPassword: "password_of_your_choice"
        password: "your_db_password_of_choice"
EOF
```

3. Install chart:

```bash
helm install ghost bitnami/ghost -n ghost \
--version 19.5.9 \
--values ./ghost-values.yaml \
--namespace ghost \
--create-namespace
```

4. Prepare External LB IP and DB 

```bash
# Set the APP_HOST to the ghost service's EXTERNAL-IP. You could also copy-paste
export APP_HOST=$(kubectl get svc --namespace ghost ghost \--template "{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}")
export MYSQL_PASSWORD=$(kubectl get secret --namespace "ghost" ghost-mysql -o jsonpath="{.data.mysql-password}" | base64 -d)
export MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace "ghost" ghost-mysql -o jsonpath="{.data.mysql-root-password}" | base64 -d)     

# Add ghostHost to ghost-values.yaml
echo "ghostHost: $APP_HOST" >> ghost-values.yaml
# Check the file to see it got added and isn't blank
cat ghost-values.yaml

# To complete the Ghost deployment, we will run upgrade with the ghostHost
helm upgrade ghost bitnami/ghost -n ghost \
    --version 19.5.9 \
    --values ./ghost-values.yaml \
    --set mysql.auth.rootPassword=$MYSQL_ROOT_PASSWORD \
    --set mysql.auth.password=$MYSQL_PASSWORD

```
5. Finally, go to the admin URL and make changes to the application. Creating a sample post, or upload a image.
The url can be located at the app host address, ```echo $APP_HOST``` /ghost. Make sure the pods are running in the newly created ghost namespce ```kubectl get pods -n ghost```

6. We take a backup of our app using Velero, also, make sure in the backup to tell the csi velero plugin to issue a snapshot ```velero backup create my-app-backup --include-namespaces ghost --exclude-resources customresourcedefinitions.apiextensions.k8s.io```. use ```velero backup get``` to make sure you took your backup. Here is a sample yaml using the csi plugin:

``` yaml
cat <<EOF | kubectl apply -f -
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: my-backup-snapshot-with-velero
  namespace: nite-velero
  annotations:
    velero.io/csi-volumesnapshot-class: "backup-snapshotclass-velero" 
spec:
    includedNamespaces:
    - ghost
    excludedResources:
    - customresourcedefinitions.apiextensions.k8s.io
    - backups.velero.io
    - ciliumnodes.cilium.io
    - ciliumnetworkpolicies.cilium.io
    - ciliumidenties.cilium.io
    - ciliumendpoints.cilium.io

EOF
```
   

7. We can easily remove now our application ```helm delete -n ghost ghost``` and ```kubectl delete pvc data-ghost-mysql-0```. Our app should be gone at this point.
8. To restore, we use ```velero restore create  my-app-db-restore --from-backup my-backup-snapshot-with-velero --exclude-resources customresourcedefinitions.apiextensions.k8s.io```. This would create a normal backup, however we can use the plugins to modify the behavior, for instance by adding an annotation to specify the type of volumeSnapshotClass you defined earlier:

Also, you can annotate pvcs, to select which class to use:

``` yaml
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  annotations:
    velero.io/csi-volumesnapshot-class: "backup-snapshotclass-velero" #When scheduled, the snapshot will use this class, overriding any annotation in the schedule.
spec:
    accessModes:
    - ReadWriteOnce
    resources:
        requests:
        storage: 1Gi
    storageClassName: aws-ebs-csi-driver
EOF
```

9.  Examine the resources came up with ```kubectl get all -n ghost``` Consider your external IP might change, so you might want to run the helm upgrade command again. 

#### CSI Snapshot plugin and Velero.

While using snapshots, the feature do not rely on the VolumeSnapshotter plugin interface. It actually uses a collection of backcupaction plugins that act against PVCs, giving the option to incorporate snapshots that are handled by the CSI driver instead of velero with the benefit of the schedule features Velero offers, and additional features to use both tools, below a summary:

| Plugin                    | Type             | Purpose                                                                                                                  |
|---------------------------|------------------|--------------------------------------------------------------------------------------------------------------------------|
| **PVCBackupItemAction**   | BackupItemAction | Backs up PersistentVolumeClaims backed by CSI volumes. Creates a CSI VolumeSnapshot, triggering the volume snapshot operation. |
| **VolumeSnapshotBackupItemAction** | BackupItemAction | Backs up volumesnapshots.snapshot.storage.k8s.io. Captures info about underlying volumesnapshotcontent. Also backs up volumesnapshotcontent and associated volumesnapshotclasses. |
| **VolumeSnapshotContentBackupItemAction** | BackupItemAction | Backs up volumesnapshotcontent.snapshot.storage.k8s.io. Looks for snapshot delete operation secrets in annotations. |
| **VolumeSnapshotClassBackupItemAction** | BackupItemAction | Backs up snapshot.storage.k8s.io.volumesnapshotclasses. Looks for snapshot list operation secrets in annotations. |
| **PVCRestoreItemAction** | RestoreItemAction | Restores PersistentVolumeClaims backed up by PVCBackupItemAction. Modifies the claim's spec to use the VolumeSnapshot as the data source. |
| **VolumeSnapshotRestoreItemAction** | RestoreItemAction | Restores volumesnapshots.snapshot.storage.k8s.io. Uses annotations from backup to create volumesnapshotcontent. Sets necessary annotations if original VolumeSnapshotContent had deletion secrets. |
| **VolumeSnapshotClassRestoreItemAction** | RestoreItemAction | Restores snapshot.storage.k8s.io.volumesnapshotclasses. Uses annotations to return any associated snapshot lister secrets. |

The plugins are used based on the annotations, and how the backup is provisioned.

#### Caveats
- Using this solution for DBS should consider mixing velero hooks to pause data ingestion, so in flight data is reduced
- For I/O intensive apps, this might require a combination of the above and multiple snapshot issues with short TTL.

### References:
https://kubernetes-csi.github.io/docs/external-snapshotter.html
https://github.com/kubernetes-csi/external-snapshotter
https://github.com/kubernetes/enhancements/tree/master/keps/sig-storage/1900-volume-snapshot-validation-webhook
https://github.com/kubernetes-csi/external-snapshotter/blob/master/deploy/kubernetes/webhook-example/README.md
https://github.com/vmware-tanzu/velero-plugin-for-csi/blob/main/README.md
https://github.com/kubernetes-csi/external-snapshotter/blob/master/deploy/kubernetes/webhook-example/README.md
https://github.com/kubernetes-csi/external-snapshotter/tree/master#validating-webhook-command-line-options
https://velero.io/docs/v1.12/csi/
