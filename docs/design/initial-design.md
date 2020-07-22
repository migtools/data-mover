# Data Mover design

The primary goal of the Data Mover is to provide a pluggable means of
backing up and restoring OpenShift Persistent Volumes, and it forms
one component of a larger backup and restore strategy. One of the main
motivations for the Data Mover is that it combines the minimum
downtime of snapshot-based backups with the restore flexibility of
filesystem-based strategies. If the user will always restore to the
same volume type (and, where relevent, to the same region), then
velero backup and restore using the Velero Plugin for CSI will handle
it. If the limited downtime of snapshot-based backup isn't needed,
then Velero backup and restore via using restic copy will work. The
Data Mover combines the advantages of both approaches. In addition, it
will allow an alternate Data Mover plugin to be used instead of the
Restic plugin to meet additional user requirements.

## Data Mover as related to the Velero CSI Plugin

Velero (and the Velero CSI plugin) will not be a requirement to use
the Data Mover. However, creating a Velero backup with CSI
snapshotting enabled is one way to provide the per-PVC CSI snapshots
required to use the Data Mover.  The Velero backup of the snapshot
won't be used. Alternatively, the user can manually create the
VolumeSnapshot resource or make use of some other infrastructure
automation tools to generate the snapshot.

Here is a summary of the actions taken by the Velero CSI plugin on backup:

- The Velero `VCBackupItemAction` creates the `VolumeSnapshot`. The
  (Kubernetes) Snapshot controller (along with the csi-snapshotter)
  then creates `VolumeSnapshotContent`.
- The Velero `VolumeSnapshotBackupItemAction` annotates the
  `VolumeSnapshot`, includes the associated `VolumeSnapshotContent`
  and `VolumeSnapshotClass` as `additionalItems` in the backup.
- the Velero `VolumeSnapshotContentBackupItemAction` and
  `VolumeSnapshotClassBackupItemAction` include relevant secrets as
  `additionalItems` in the backup

## Data Mover CRDs

The following CRDs will be needed by the Data Mover:

### `DataMoverConfig`

This could be implemented as a ConfigMap instead of a separate
configuration CRD. In the absence of an explicit Backup resource, this
will contain configuration necessary to tell the DataMover how to back
up VolumeSnapshots as they are created. This will need to be either a
well-known default name/namespace, or the Data Mover Controller will
need an environment variable with a reference. In addition to
configuring the plugin and storage location, this resource will also
contain the selection specification for VolumeSnapshots to back up --
back up all snapshots, or by namespace, or by label, etc. There may be
plugin-specific metadata needed as well, although that may be better
placed in the StorageLocation.

Since other plugins may require additional parameters, we may need a
map-based `config` element as well. Parameters listed here that relate
explicitly to PVC configuration can probably be kept as explicit CRD
fields, though, since any data mover implementation will need to deal
with them.

Open question: Do we need a separate StorageLocation CRD, or can that
be merged into this Configuration resource?

apiVersion: oadp.openshift.io/v1alpha1
kind: DataMoverConfig
metadata:
  name: example-config
  namespace: oadp-example
spec:
  storageLocationRef:
    name: storage-example
    namespace: oadp-example
  plugin: restic # the restic reference plugin
  config:
    # plugin-specific metadata fields would go here.
  includeAllSnapshots: false
  namespaces:
  - example-project
  labelSelector:
    matchLabels:
      app: mysql

### `DataMoverStorageLocation`

We need a way to model a storage location for PVC storage. The
simplest thing to do here would be just to reuse either Velero's
`BackupStorageLocation` model, or CAM's `MigStorage` model. We could
define our own with its own model, or we could even have a more
flexible system where a `DataMoverStorageLocation` could optionally
reference a Velero `BackupStorageLocation`, a CAM `MigStorage`, or a
third data-mover-internal model. Regardless of the choice here, the
metadata needs are approximately the same as the pre-existing Velero
and CAM CRDs. In subsequent examples, `StorageLocation` is used as a
stand-in for whatever is decided on here.

### `DataMoverVolumeBackup`

The `DataMoverVolumeBackup` represents the backup operation for a
single volume. This resource will be created by the Data Mover
Controller when it responds to a completed
VolumeSnapshot. StorageClassName, AccessModes, and Storage are pulled
from the original PVC.

We also need to preserve VolumeSnapshot labels (also PVC labels?) and
VolumeSnapshot creation time in plugin-accessible metadata, in
addition to the namespaced names of originalPVC and volumeSnapshot

```
apiVersion: oadp.openshift.io/v1alpha1
kind: DataMoverVolumeBackup
metadata:
  name: example-backup-68b9q
  namespace: oadp-example
    ownerReferences: # should this reference the VolumeSnapshot, or
                     #    should it be empty?
spec:
  volumeSnapshot:
    name: mysql
    namespace: mysql-persistent
  originalPVC:
    name: mysql
    namespace: mysql-persistent
  dataMoverPVC:
    name: mysql-q3e4q
    namespace: mysql-persistent
  storageClassName: csi-ceph
  accessModes:
  - ReadWriteMany
  storage: 10Gi
status:
  phase: Completed

```

### `DataMoverRestore`

The `DataMoverRestore` is the resource created to initiate a Restore
operation in the Data Mover controller. The main things this needs to
contain is a StorageLocation reference and an identification of the
PVCs/snapshots to restore. We should support multiple ways of
identification.

Listing namespaces will restore every PVC in those namespaces:
- Do we allow remapping of namespaces on restore?
- If there are multiple snapshot backups for a PVC, with
  namespace-level restore we take the latest.

Specify a label to restore. This will be similar to a labelSelector,
but it won't actually be a label selector, since the backups are not
kubernetes resources. When backing up VolumeSnapshots, VolumeSnapshot
labels need to be accessible to the plugin.
- Do we also allow labels set on the PVC prior to backing up, or
  should the infrastructure that creates the VolumeSnapshot be
  responsible for copying labels from PVC to VolumeSnapshot?
- Will label-based restore allow for changing namespace or name? This
  would be difficult to implement a reasonable API for this part.
- If there are multiple snapshot backups for a PVC with the matching
  we take the latest.
- If there are multiple backups for a snapshot, with namespace-level
  restore we take the latest.

Specify individual PVCs to restore by namespaced name.
- Do we allow remapping of namespaces or names on restore? At this
  level, it would be possible to allow it.
- If there are multiple snapshot backups for a given PVC  we take the
  latest. 

Specify individual VolumeSnapshots to restore:
- What key is indexed here? Namespaced name of VolumeSnapshot?
  Something else?
- Do we allow remapping of namespaces or names on restore? At this
  level, it would be possible to allow it.
- If there are multiple snapshot backups selected which map to a given
  PVC (either using the prior namespaced name, or with remapping),
  there is the potential for failure. Do we just ignore the second
  one? Do we detect a clash and fail before attempting?

General question: Can the StorageLocation reference be omitted if the
DataMoverConfig StorageLocation is to be used? This goes away entirely
if the StorageLocation is combined with DataMoverConfig (i.e. only one
StorageLocation per cluster is supported)

Since other plugins may require additional parameters, we may need a
map-based `config` element as well. Parameters listed here that relate
explicitly to PVC configuration can probably be kept as explicit CRD
fields, though, since any data mover implementation will need to deal
with them.

per-PVC spec parameters
- `originalPVC` (namespaced name)
- `targetPVC` (namespaced name)
  Optional: if not specified, use `originalPVC` values
- `targetStorageClassName`
  Optional: if not specified, use storageClassName from backup
- `targetAccessModes`
  Optional: if not specified, use accessModes from backup
- `targetStorage`
  Optional: Used for setting `Requests.Storage` for the new PVC. If
  omitted, the value from `Capacity.Storage` from the PVC backed up is used

When the DataMoverVolumeRestores are created, omitted values
from the Spec field will be filled in with the from-backup defaults.

Open question: Is the status.pvcs field below needed? Essentially it
provides the full set of all snapshots/PVCs to restore. It will be the
union of the following (if more than one match a given target
namespaced name, the most recent is used):
- all Snapshots in the namespaces listed
- all Snapshots matching the label selector
- latest snapshot for each pvc in Spec.Pvcs
- all Snapshots listed in Spec.Snapshots

For each of the above, the controller should create one
DataMoverVolumeRestore. Status.Pvcs is a redundant bit of data, but it
might help the controller if it's here as well.

Here is an example of a `DataMoverRestore` resource.

```
apiVersion: oadp.openshift.io/v1alpha1
kind: DataMoverRestore
metadata:
  name: example-restore
  namespace: oadp-example
spec:
  dataMoverBackup: example-backup
# if UIDs are used:
# dataMoverBackup: fbd0eeeb-6ea7-11ea-912c-0274cdd9b69e
  # plugin might come from active DataMoverConfig
  plugin: restic # the restic reference plugin
  config:
    # plugin-specific metadata fields would go here.
  # StorageLocation might come from active DataMoverConfig
  storageLocationRef:
    name: storage-example
    namespace: oadp-example
  namespaces:
  - example-project
  labelSelector:
    matchLabels:
      app: mysql
  snapshots: # instead of namespaced name, possibly other key?
  - volumeSnapshot:
      name: mysql-snapshot
      namespace: mysql-persistent
    targetPVC:
      name: mysql3-moved
      namespace: mysql-persistent
    targetStorageClassName: csi-ceph
    targetAccessModes:
    - ReadWriteMany
  - volumeSnapshot:
      name: mysql-snapshot
      namespace: mysql-persistent
    targetStorage: 20Gi
  pvcs:
  - originalPVC:
      name: mysql
      namespace: mysql-persistent
    targetPVC:
      name: mysql-moved
      namespace: mysql-persistent
    targetStorageClassName: csi-ceph
    targetAccessModes:
    - ReadWriteMany
  - originalPVC:
      name: mysql2
      namespace: mysql-persistent
    targetStorage: 20Gi
status:
  pvcs:
  - volumeSnapshot:
      name: mysql-snapshot
      namespace: mysql-persistent
    originalPVC:
      name: mysql
      namespace: mysql-persistent
    targetPVC:
      name: mysql-moved
      namespace: mysql-persistent
    targetStorageClassName: csi-ceph
    targetAccessModes:
    - ReadWriteMany
    targetStorage: 20Gi
  - volumeSnapshot:
      name: mysql-snapshot
      namespace: mysql-persistent
    originalPVC:
      name: mysql2
      namespace: mysql-persistent
    targetPVC:
      name: mysql2
      namespace: mysql-persistent
    targetStorageClassName: csi-ceph
    targetAccessModes:
    - ReadWriteMany
    targetStorage: 20Gi
  phase: Completed
```

### `DataMoverVolumeRestore`

The `DataMoverVolumeRestore` represents the restore operation for a
single volume. This resource will be created by the Data Mover
Controller, one for each volume being restored in the
`DataMoverRestore` operation. StorageClassNme, AccessModes, and
Storage are pulled from the original PVC (if not available, can this
be inferred from the CSI snapshot? If not, figure out default -- in
any case this matches what's used by the DataMoverPVC)

```
apiVersion: oadp.openshift.io/v1alpha1
kind: DataMoverVolumeRestore
metadata:
  name: example-restore-68b9q
  namespace: oadp-example
    ownerReferences:
    - apiVersion: oadp.openshift.io/v1alpha1
      kind: DataMoverRestore
      name: example-restore
      uid: 2968fe09-aa94-11ea-85ad-0274cdd9b69e
spec:
  config:
    resticSnapshotId: b2493502 # plugin-specific
    # if rsync-based, some plugin-defined identifier (possibly just
    #    using something from the VolumeSnapshot originally)
  volumeSnapshot: # instead of namespaced name, possibly other key?
    name: mysql-snapshot
    namespace: mysql-persistent
  originalPVC:
    name: mysql
    namespace: mysql-persistent
  targetPVC:
    name: mysql-moved
    namespace: mysql-persistent
  targetStorageClassName: csi-ceph
  targetAccessModes:
  - ReadWriteMany
  targetStorage: 20Gi
status:
  phase: Completed
---
apiVersion: oadp.openshift.io/v1alpha1
kind: DataMoverVolumeRestore
metadata:
  name: example-restore-68b9r
  namespace: oadp-example
    ownerReferences:
    - apiVersion: oadp.openshift.io/v1alpha1
      kind: DataMoverRestore
      name: example-restore
      uid: 2968fe09-aa94-11ea-85ad-0274cdd9b69e
spec:
  config:
    resticSnapshotId: b249350a # plugin-specific
  volumeSnapshot: # instead of namespaced name, possibly other key?
    name: mysql-snapshot2
    namespace: mysql-persistent
  originalPVC:
    name: mysql2
    namespace: mysql-persistent
  targetPVC:
    name: mysql2
    namespace: mysql-persistent
  targetStorageClassName: csi-ebs
  targetAccessModes:
  - ReadWriteOnce
  targetStorage: 20Gi
status:
  phase: Completed
```

## Data Mover Controller

The Data Mover Controller is a long-running pod (implemented as a
`Deployment`) which watches `VolumeSnapshot` and
`DataMoverRestore` resources.

TBD: The DataMoverVolumeBackup, DataMoverRestore, and
DataMoverVolumeRestore CRs will have a status/phase field with a 
to-be-determined workflow. Possible states to deal with include:

- New (might not be needed since once Status is no longer empty, we've
  moved beyond this status
- Processing (sub-states? progress reporting?)
- WaitingOnDataMoverActionController (Restore)
- DeletingPVCs (VolumeBackup)
- Completed
- Failed (sub-states? FailedProcessing, FailedDataMoverActionController, etc.)

### Backups

For each new `VolumeSnapshot`:
- Wait for snapshot contents to be ready:
  - If snapshot.Status is empty or Status.BoundVolumeSnapshotContentName is empty, keep waiting
  - Find VolumeSnapshotContent based on Status.BoundVolumeSnapshotContentName
  - If snapshotContent.Status is empty or Status.SnapshotHandle is empty, keep waiting
  - Any other Status fields to check?

- Create new PVC with VolumeSnapshot DataSource, using VolumeSnapshot
  from above
  - Find PVC via spec.VolumeSnapshotRef.
    - For dynamically provisioned VolumeSnapshots this is required.
    - For pre-provisioned snapshots, it is not. Do we support
      pre-provisioned snapshots? If so, if there any reliable way to
      get PVC metadata? name, namespace, size, StorageClass, AccessMode
  - request.Storage from source PVC Status.Capacity.Storage
  - StorageClassName from source PVC
  - update `DataMoverBackup` `Status` with Spec PVC fields and
    additional metadata to include in backup (storageClassName,
    accessModes, storage)
- Create a `DataMoverVolumeBackup` with
  appropriate owner references and metadata (see above section).
- Open Question: See above, related to supporting pre-provisioned snapshots?

- Once the PVC is created, create a Deployment for a
  DataMoverActionController, including a separate InitContainer for
  the selected data mover plugin:
  - Where is the line between Data Mover core and the plugin?
  - Will the plugin be a go container similar to velero storage
    plugins which have several implemented API calls, all in go, or
    will there be a small go plugin for queries only (mostly for
    restore), and a separate binary only container where the API
    consists of a CLI with arguments specified as environment variables?
  - New PVC created above is mounted in the
    DataMoverActionController (or vendor-supplied binary) deployment
  - `DataMoverVolumeBackup` reference will be passed to Deployment in the
    DATA_MOVER_BACKUP environment variable (or explicit args to vendor
    binary set in TBD environment variable).

### Restores

For each new `DataMoverRestore`:
- Determine list of Snapshots/PVCs to restore. Plugin interface needs
  to be generated to allow querying for a list of snapshots by:
  - Namespace
  - Label (foo="bar")
  - VolumeSnapshot key (namespaced name, UID, or something else)
  - PVC namespaced name
  Plugin interface (either for the full vendor plugin, if implemented
  in go, or for the more specialized vendor query plugin, if the
  actual backup/restore is done via binary image) needs the following
  API calls (note: if go plugin is limited to query interface, then
  we'll use `DataMoverQuery` instead of `DataMoverPluginAction` for this):
  ```
  type DataMoverSnapshot struct {
       # Stored at backup time. Either "what time the controller
       # processed snapshot" or pulled from VolumeSnapshot
       # creationTimestamp. Used for comparison to determine which
       # snapshot to use if more than one are chosen for a given target PVC
       CreationTimestamp metav1.Time        
       VolumeSnapshot    NamespacedName // instead of of namespaced name, possibly other key?
       PVC               NamespacedName
       StorageClassName  string
       AccessModes	 []PersistentVolumeAccessMode
       # is status.Capacity sufficient, or do we need to store
       # Spec.Request and Spec.Limit?
       Storage		 resource.Quantity 
  }
  type DataMoverPluginAction interface {
       FindByNamespace(namespace string) (DataMoverSnapshot[], error)
       # should we use an actual LabelSelector or just a key/value pair?
       FindByLabel(labelKey, labelValue string) (DataMoverSnapshot[], error)
       FindBySnapshot(namespace, name string) (DataMoverSnapshot[], error)
       FindByPVC(namespace, name string) (DataMoverSnapshot[], error)
  }
  ```
  From the above calls, we need to merge the list so that only one
  snapshot (the one with the latest CreationTimestamp) will be
  restored for each targetPVC namespaced name.  
- For each PVC to be restored:
  - Update `DataMoverRestore.Status` with appropriate metadata
  - Create new PVC using Namespace, Name, Storageclass, size,
    AccessMode from output above
  - Create `DataMoverVolumeRestore` with appropriate fields (see above
    CRD section).

- Once the empty PVCs are created, create a Deployment for a
  - DataMoverActionController, including a separate InitContainer for
    the selected data mover plugin
  - See above on the open question of whether this is a go plugin or a
    binary image with args in env vars
  - Open question 2: One deployment per `DataMoverVolumeRestore`
    (i.e. one per volume) or one per `DataMoverRestore` (one for all
    volumes)
  - New PVCs created above are all mounted in the deployment (or one
    per deployment)
  - `DataMoverRestore` reference will be passed to Deployment in the
    DATA_MOVER_RESTORE environment variable (or all args via env vars).

## Data Mover Action Controller

This is a short-lived Deployment  that's
run once per `DataMoverVolumeBackup` or `DataMoverRestore` (or
`DataMoverVolumeRestore`) resource. We can probably use the same image
for both operations. The main open question is whether this is a go
plugin or whether it's vendor-supplied binary image that takes all
required args via environment variables. If it's the former, then this
Deployment will manage the
DataMoverVolumeBackup/DataMoverVolumeRestore resources. If it's the
latter, then we will need to determine the appropriate way for the
DataMoverController to wait for output from the vendor binary and
manage the DataMoverVolumeBackup/DataMoverVolumeRestore resources itself.
The Data Mover Controller will be responsible for launching and
terminating this deployment as needed.

### Backups

A `VolumeSnapshot` reference is passed into the pod via the
DATA_MOVER_VOLUME_BACKUP environment variable. For the PVC referenced
by the `DataMoverVolumeBackup` (already mounted), perform a backup
using the plugin (restic or rsync in the reference implementation)
from the mounted PVC into the referenced
`StorageLocation`. Alternately, pass all `DataMoverVolumeBackup`
metadata to the binary image via environment variables. Method of
return status is still TBD.

#### Reference plugin backup

For the reference plugin, we have the option of implementing using
restic, or using rsync, or writing two reference plugins, one for
each.

The reference plugin will do the following:
- Perform a restic/rsync backup
- Store appropriate per-PVC Status metadata into the
  `StorageLocation`:
  - CreationTimestamp
  - VolumeSnapshot namespaced name
  - Original PVC namespaced name
  - StorageClassName
  - PVC access modes
  - PVC capacity
  - label k/v pairs
  - Restic Snapshot ID (or appropriate identifier to be able to copy
    back out via rsync)

The plugin must be able to query and identify lists of snapshots via
VolumeSnapshot namespaced name, PVC namespaced name, namespace only
(PVC? VolumeSnapshot? Will these always be the same?), or a label k/v
pair. If more than one snapshot for a given PVC namespaced name match
for a namespace or label query, should only the latest should
returned, or should the DataMover controller filter that?)

The plugin must be able to find the appropriate snapshot data for
restore based on the VolumeSnapshot's namespaced name. If multiple
Snapshots (sequentially) reuse the same namespaced name, newer ones
will overwrite older ones. Is that OK? Do we need to use some other
form of UID instead to identify the snapshot in all of our CRs instead
of NamespacedName?

### Restores

`DataMoverRestore` is passed into the pod via the DATA_MOVER_BACKUP
environment variable. For the PVC referenced by each
`DataMoverVolumeRestore` owned by the `DataMoverRestore` (all should
be mounted), perform a restore using the plugin (restic or rsync in the
reference implementation) from the the referenced `StorageLocation` to
the mounted PVC. Alternately, pass all `DataMoverVolumeBackup`
metadata to the binary image via environment variables. Method of
return status is still TBD.
Open question: One Deployment per PVC, or one deployment for all PVCs?

## Other Open issues to address:

- ceph/ocs API concerns: change block tracking, incremental
  snapshotting (if relevant). This may be outside the Data Mover scope
  if we assume snapshots already exist.
