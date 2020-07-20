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

## `DataMoverStorageLocation`

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

### `DataMoverBackup`

The `DataMoverBackup` is the resource created to initiate a Backup
operation in the Data Mover controller. The main things that this must
contain are a list of PVCs to back up and a `StorageLocation`
reference.

The main open question here is whether an explicit list of PVCs be the
only way to identify what to back up.  Will we also allow a list of
namespaces (including all PVCs in each namespace)? What about allowing
the use of label selectors? If these alternate means are supported, do
we support all 3 at once and take the union of the matches, or do we
require that if one is specified the others are left blank?

Since other plugins may require additional parameters, we may need a
map-based `config` element as well. Parameters listed here that relate
explicitly to PVC configuration can probably be kept as explicit CRD
fields, though, since any data mover implementation will need to deal
with them.

Here is an example of a `DataMoverBackup` resource.

```
apiVersion: oadp.openshift.io/v1alpha1
kind: DataMoverBackup
metadata:
  name: example-backup
  namespace: oadp-example
spec:
  storageLocationRef:
    name: storage-example
    namespace: oadp-example
  plugin: restic # the restic reference plugin
  config:
    # plugin-specific metadata fields would go here.
  pvcs:
  - name: mysql
    namespace: mysql-persistent
  - name: mysql2
    namespace: mysql-persistent
  namespaces:
  - example-project
  labelSelector:
    matchLabels:
      app: mysql
status:
  pvcs:
  - originalPVC:
      name: mysql
      namespace: mysql-persistent
    dataMoverPVC:
      name: example-backup-q3e4q
      namespace: mysql-persistent
    storageClassName: csi-ceph
    accessModes:
    - ReadWriteMany
    storage: 10Gi
  - originalPVC:
      name: mysql2
      namespace: mysql-persistent
    dataMoverPVC:
      name: example-backup-6er3q
      namespace: mysql-persistent
    storageClassName: csi-ceph
    accessModes:
    - ReadWriteMany
    storage: 10Gi
  phase: Completed

```

### `DataMoverVolumeBackup`

The `DataMoverVolumeBackup` represents the backup operation for a
single volume. This resource will be created by the Data Mover
Controller, one for each volume being backed up in the
`DataMoverBackup` operation. StorageClassName, AccessModes, and
Storage are pulled from the original PVC (if not available, can this
be inferred from the CSI snapshot? If not, figure out default -- in
any case this matches what's used by the DataMoverPVC)

```
apiVersion: oadp.openshift.io/v1alpha1
kind: DataMoverVolumeBackup
metadata:
  name: example-backup-68b9q
  namespace: oadp-example
    ownerReferences:
    - apiVersion: oadp.openshift.io/v1alpha1
      kind: DataMoverBackup
      name: example-backup
      uid: 2968fe09-aa94-11ea-85ad-0274cdd9b69e
spec:
  originalPVC:
    name: mysql
    namespace: mysql-persistent
  dataMoverPVC:
    name: example-backup-q3e4q
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
operation in the Data Mover controller. The main things that this must
contain are the `DataMoverBackup` name, a list of PVCs to restore and
a `StorageLocation` reference.

One open question is what form to provide the Backup reference. We
can't assume that the `DataMoverBackup` resource will exist in the
cluster, but we still need an identifying name to use to find the
backups in the storage location. Whatever identifier is used here must
be the same one used (possibly as a containing directory in the
storage location) when backing up the volumes into object storage. We
could use the Backup name, although this requires Backup names to be
unique among all clusters using the same storage location. Reusing a
name in a different cluster could result in data loss. Alternatively,
we could use the uid from the `DataMoverBackup`, which should
eliminate that problem. Whatever element is used to identify the
backup is the one which would be specified here.

Also, regarding the PVCs to restore. Do we require an explicit list,
or would an empty PVC list mean "restore all PVCs from the backup". An
empty PVC list also means that the other per-PVC parameters will be
the same as if the PVC list was included but the other parameters were
omitted.

Another question: if a listed PVC matching `originalPVC` is not found
in the referenced backup, is it ignored, listed with a warning, or an
error which aborts the restore?

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
  plugin: restic # the restic reference plugin
  config:
    # plugin-specific metadata fields would go here.
  storageLocationRef:
    name: storage-example
    namespace: oadp-example
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
`Deployment`) which watches `DataMoverBackup` and
`DataMoverRestore` resources.

TBD: The Backup and Restore CRs will have a status/phase field with a
to-be-determined workflow. Possible states to deal with include:

- New (might not be needed since once Status is no longer empty, we've
  moved beyond this status
- Processing (sub-states? progress reporting?)
- WaitingOnDataMoverActionController
- DeletingPVCs (Backup only)
- Completed
- Failed (sub-states? FailedProcessing, FailedDataMoverActionController, etc.)

### Backups

For each new `DataMoverBackup`:
- For each PVC to be backed up, find a VolumeSnapshot
  - If no snapshot, do we fail? skip? create one?
  - If multiple snapshots, take most recent?
  - Do we add an optional `VolumeSnapshot` parameter to the
    `DataMoverBackup` PVC element to allow non-recent choices?
  - If VolumeSnapshot found, wait for snapshot contents to be ready:
    - If snapshot.Status is empty or Status.BoundVolumeSnapshotContentName is empty, keep waiting
    - Find VolumeSnapshotContent based on Status.BoundVolumeSnapshotContentName
    - If snapshotContent.Status is empty or Status.SnapshotHandle is empty, keep waiting
    - Any other Status fields to check?

  - Create new PVC with VolumeSnapshot DataSource, using VolumeSnapshot from above
    - request.Storage from source PVC Status.Capacity.Storage
    - StorageClassName from source PVC
    - update `DataMoverBackup` `Status` with Spec PVC fields and
      additional metadata to include in backup (storageClassName,
      accessModes, storage)
  - For each new PVC, create a `DataMoverVolumeBackup` with
    appropriate owner references and metadata (see above section).
  - Open question: do we require source PVC to exist, or should we
    allow an alternate method of specifying just a CSI snapshot? If
    so, do we also need to require the user to specify
    StorageClass, capcity, etc. or can that be inferred from the
    CSI snapshot?

- Once the PVCs are created, create a Deployment for a
  DataMoverActionController, including a separate InitContainer for
  the selected data mover plugin:
  - Where is the line between Data Mover core and the plugin?
  - New PVCs created above are all mounted in the
    DataMoverActionController deployment
  - `DataMoverBackup` reference will be passed to Deployment in the
    DATA_MOVER_BACKUP environment variable.

### Restores

For each new `DataMoverRestore`:
- For each PVC to be restored:
  - Create new PVC using Namespace, Name, Storageclass, size,
    AccessMode from DataMoverRestore
  - Create `DataMoverVolumeRestore` with appropriate fields (see above
    CRD section).

- Once the empty PVCs are created, create a Deployment for a
  - DataMoverActionController, including a separate InitContainer for
  - the selected data mover plugin
  - New PVCs created above are all mounted in the deployment
  - `DataMoverRestore` reference will be passed to Deployment in the
    DATA_MOVER_RESTORE environment variable.

## Data Mover Action Controller

This is a short-lived Deployment created as a `Job` resource that's
run once per `DataMoverBackup` or `DataMoverRestore` resource. We can
probably use the same image for both operations, but it's still an
open question. There's no reason we can't use separate images if it
results in a cleaner implementation. The plugin will be included as an
initContainer similar to Velero plugins. The Data Mover Controller
will be responsible for launching and terminating this deployment as
needed.

### Backups

A `DataMoverBackup` reference is passed into the pod via the
DATA_MOVER_BACKUP environment variable. For the PVC referenced by each
`DataMoverVolumeBackup` owned by the `DataMoverBackup` (all should be
mounted), perform a backup using the plugin (restic in the reference
implementation) from the mounted PVC into the referenced
`StorageLocation`.

#### Reference (restic) plugin backup

The reference plugin will do the following:
- Perform a restic backup
- Store appropriate per-PVC Status metadata into the
  `StorageLocation`:
  - Original PVC namespaced name
  - PVC capacity
  - StorageClassName
  - Restic Snapshot ID

The metadata and all snapshots should be stored under a top level
directory which uniquely identifies the `DataMoverBackup` (name, UID,
etc.)

### Restores

`DataMoverRestore` is passed into the pod via the DATA_MOVER_BACKUP
environment variable. For the PVC referenced by each
`DataMoverVolumeRestore` owned by the `DataMoverRestore` (all should
be mounted), perform a restore using the plugin (restic in the
reference implementation) from the the referenced `StorageLocation` to
the mounted PVC.

## Other Open issues to address:

- Possible alternate design approach: Instead of watching for
  `DataMoverBackup` resources, the Data Mover controller could watch
  CSI snapshots instead. It depends on whether we want a
  "backup-centric" approach, or a "PV/snapshot-centric" approach. The
  explicit "back this up" approach allows the user to group PVCs and
  explicitly call out storage locations. It also allows users to be
  selective about which PVs get backed up into the defined
  `StorageLocation`. The alternative would be to configure the
  controller with a storage location to use for all snapshots, and
  then back up one PVC at a time (for every PVC in the cluster for
  which CSI snapshots are being taken) from the controller. This will
  complicate restores, though, as there's no longer a backup ID to use
  to pull volumes out to restore. We could key restores off namespaced
  name of PVC, although this could be a problem with multiple clusters
  (and multiple backups per PVC). We could key it off "PVC namespaced
  name+source cluster ID", but I think this ultimately comes down to
  what does the restore use case look like. Is it better for a user to
  choose a backup operation and (optionally) backed-up volumes within
  it, or is better to choose volumes by some other means and drop the
  notion of a `DataMoverBackup` -- I assume we'll still need the
  `DataMoverRestore`, but in the "no backup resource" case, the
  restore would just take a list of "backed up volume"
  references. This goes back to how we identify the backups at backup
  time, and how we locate them at restore time. The other challenge is
  the question of plugins. With an explicit `DataMoverBackup`
  resource, the way of selecting PVCs for restore is plugin-agnostic,
  but if the only artifact of backing up is what the plugin creates in
  the `BackupLocation`, then we're relying on the actions of the
  plugin for the definition of the identifier that the main controller
  takes as an argument for restore. In any case, we have a few options
  for this identifier:
  - Namespaced name of PVC (doesn't account for multiple backups)
  - Namespaced name of CSI snapshot
￼ - UID of the above
  - some combination
- ceph/ocs API concerns: change block tracking, incremental
  snapshotting (if relevant). This may be outside the Data Mover scope
  if we assume snapshots already exist.
