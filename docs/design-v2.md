# Azure Disk CSI Driver V2 (Project Mandalay)

Project Mandalay enchances the Azure Disk CSI Driver to improve scalability and reduce pod 
failover latency. It uses shared disks to provision replica mounts on multiple cluster nodes 
and integrates with the pod scheduler to ensure a node with a replica mount is chosen on pod 
failover.

## Architecture and Components

The diagram below shows the components in the Kubernetes control plane (CCP) and cluster 
nodes, and the Azure services the Mandalay driver uses.

![Mandalay Architecture](images/mandalay_arch.png)

### Controller Plug-in

In addition to the CSI Controller API Server, this plug-in hosts several controllers for 
custom resources used to orchestrate persistent volume management and node placement.

The controller plug-in is deployed as a [ReplicaSet](https://kubernetes.io/docs/concepts/
workloads/controllers/replicaset/) through [Deployment](https://kubernetes.io/docs/concepts/
workloads/controllers/deployment/) with leader election.

### Node Plug-in

In addition to the CSI Node API Server, this plug-in also provides feedback for pod placement 
used by the scheduler extender described below.

The node plug-in is deployed on each node in the cluster as a [DaemonSet](https://kubernetes.
io/docs/concepts/workloads/controllers/daemonset/).

### Scheduler Extender

Mandalay provides a [scheduler extender](https://github.com/kubernetes/community/blob/master/
contributors/design-proposals/scheduling/scheduler_extender.md) that is responsible for 
influencing pod placements.

Like the controller plug-in, the scheduler extender is deployed as a [ReplicaSet](https://
kubernetes.io/docs/concepts/workloads/controllers/replicaset/) through [Deployment](https://
kubernetes.io/docs/concepts/workloads/controllers/deployment/) with leader election.

## Implementation

This section describes the changes and high-level implementation details of the different 
components in Mandalay.

### StorageClass

In addition to the existing StorageClass parameters described in [driver-parameters]
(driver-parameters.md), Mandalay supports the following:

| Name                                        | Meaning                                   | Available Value                                        | Mandatory | Default value                                                                                     |
| ------------------------------------------- | ----------------------------------------- | ------------------------------------------------------ | --------- | ------------------------------------------------------------------------------------------------- |
| `maxMountReplicaCount`                      | The number of mount replicas to maintain. | This value must be in the range `[0..(maxShares - 1)]` | No        | If `accessMode` is `ReadWriteMany`, the default is `0`. Otherwise, the default is `maxShares - 1` |
| `enableFileSystemConsistencyCheck`          | ???                                       | `true` or `false`                                      | No        | `false`                                                                                           |
| `enableMountReplicaReductionOnDiskPressure` | ???                                       | `true` or `false`                                      | No        | `false`                                                                                           |

### Custom Resources and Controllers

Mandalay uses 3 different custom resources defined in the `azure-disk-csi` namespace to 
orchestrate disk management and facilitate pod failover to nodes with mount replicas. A 
controller for each resource watches for and responds to changes to its custom resource 
instances.

#### `AzDriverNode` Resource and Controller

The `AzDriverNode` custom resource represents a node in the cluster where the Mandalay node 
plug-in runs. An instance of `AzDriverNode` is created when the node plug-in starts. The node 
plug-in periodically updates the heartbeat in the `Status` field.

The controller for `AzDriverNode` runs in the controller plug-in. It is responsible for 
deleting `AzDriverNode` instances that no longer have corresponding nodes in the cluster.

#### `AzVolume` Resource and Controller

The `AzVolume` custom resource represents the managed disk of a `PersistentVolume`. The 
controller for `AzVolume` runs in the controller plug-in. It watches for and reconciles 
changes in the `AzVolume` instances.

An `AzVolume` instance is created by the `CreateVolume` API to the CSI Controller plug-in. The 
`AzVolume` controller responds to the new instance by creating a managed disk using the 
parameters in the referenced `StorageClass`. When disk creation is complete, the controller 
sets the `.Status.State` field to `Created` or suitable error state on failure. It also adds a 
finalizer if creation was successful. The `CreateVolume` request completes when the `AzVolume` 
state is updated.

To delete a managed disk, the `DeleteVolume` API in the CSI Controller plug-in deletes the 
corresponding `AzVolume` instance. The controller responds to deletion by garbage collecting 
the managed disk based on the `reclaimPolicy` set in the `StorageClass`. When the reclaim 
policy has been applied, the controller removes the finalizer from the `AzVolume` instance. 
The `DeleteVolume` request completes once the `AzVolume` instance has been removed from the 
object store.

#### `AzVolumeAttachment` Resource and Controller

The `AzVolumeAttachment` custom resource represents the attachment of a managed disk to a 
specific node. The controller for this custom resource runs in the controller plug-in and 
watches for changes in the `AzVolumeAttachment` instances.

An `AzVolumeAttachment` instance representing the primary node attachment is created by the 
`ControllerPublishVolume` API in the CSI Controller plug-in. If an instance for the current 
node already exists, it is updated to represent the primary node attachment. The 
`AzVolumeAttachment` controller then attaches the manage disk to the primary node (downgrading 
to mount replica or detaching any previous primary attachment if one exists). It then creates 
additional `AzVolumeAttachment` instances representing mount replicas and attaches the shared 
manage disk to a number of backup nodes as specified by the `maxMountReplicaCount` parameter 
in the `StorageClass` instance of the `PersistentVolumeClaim` of the managed disk. The 
controller also sets the `.Status.State` field to `Attached` and a finalizer to each 
`AzVolumeAttachment` instance as each attachment completes. The `ControllerPublishVolume` 
request is complete when the state of the primary node attachment is complete.

When the `ControllerUnpublishVolume` API in the CSI Controller plug-in is called, it deletes 
the `AzVolumeAttachment` instance for the current (primary) node. The controller responds by 
detaching the managed disk from the primary and mount replica nodes. It removes the finalizer 
from the `AzVolumeAttachment` instances as each detach operation completes. The 
`ControllerUnpublishVolume` request is complete when all detach operations have completed and 
corresponding `AzVolumeAttachments` have been removed from the object store.

### Scheduler Extender

The Manadalay scheduler extender influences pod placement by prioritizing healthy nodes where 
mount replicas for the requried persisten volume(s) already exist (i.e. node(s) to which the 
managed disk(s) is(are) already attached). It relies on the `AzVolumeAttachment` instances to 
determine which nodes have mount replicas, and the heartbeat information in the `AzDriverNode` 
to determine health. If no mount replicas for the specified persistent volume currently exist, 
the scheduler extender will weight all nodes equally.

### Provisioner Library

The Provisioner Library is a common library to abstract the underlying platform for all 
Mandalay plugins, services and controllers. It handles the platform-specific details of 
performing volume operations such as (but not necessarily limited to) create, delete, attach, 
detach, snapshot, stage, unstage, mount, unmount, etc.
