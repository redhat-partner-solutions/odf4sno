ODF4SNO
=======

This repository assists in running
[OpenShift Data Foundation (ODF)](https://www.redhat.com/en/technologies/cloud-computing/openshift-data-foundation)
on Single-Node OpenShift (SNO). It contains scripts, manifests, and a detailed guide.

Note that this repository is not necessarily the definitive or officially supported approach. It is but one approach.

The starting point is a single machine with more than one disk. The result is CoreOS installed on one disk with
OpenShift on top and with [Ceph](https://ceph.io/en/) as one of its workloads. Ceph would be managing the rest of
the disks and can 1) dynamically provision persistent volumes to other workloads on OpenShift using
[CSI](https://kubernetes-csi.github.io/docs/), and 2) export its storage to the network at large (filesystem,
object, and block).

Use Cases
---------

Out-of-the-box SNO does not come with support for dynamically provisioned persistent volumes, so adding ODF nicely
fills in that gap. That said, running Ceph on a single node is not generally recommended. Indeed, doing so disables
some of Ceph's powerful redundancy, replication, and failover features.

Another use case is deploying a flexible single-box NAS. Though you can install Ceph directly on the machine,
running it on top of CoreOS and OpenShift allows for robust management, and of course enables running many other
Kubernetes workloads side-by-side with Ceph, too.

Alternatives to ODF
-------------------

SNO *does* come with the [Local Storage Operator](https://github.com/openshift/local-storage-operator), which
allows for *non*-dynamic persistent volumes on local drives. Similarly, it does not provide a distributed
filesystem so the pods must be located on the nodes with the drives.

It might be good enough for your use case, and indeed it would offer the best possible raw performance as it allows
pods to directly access the drive. You can use it to create your persistent volumes manually or enable "Local Volume
Discovery" (not enabled by default), which will automatically create persistent volumes with the name prefix
`local-pv-` for all unused disks. (It will also populate the "Disks" tab for Nodes in the OpenShift web dashboard.)

Also check out these alternatives, which are similar to the Local Storage Operator but add the ability to dynamically
provision persistent volumes:

* [TopoLVM](https://github.com/topolvm/topolvm) (Cybozu): provisions LVM partitions per volume
* [Open-Local](https://github.com/alibaba/open-local) (Alibaba): provisions LVM partitions per volume
* [Local Path Provisioner](https://github.com/rancher/local-path-provisioner) (Rancher): provisions hostPath directories per volume

Guide
-----

### 1. Preparation

Start by installing [SNO](https://cloud.redhat.com/blog/deploy-openshift-at-the-edge-with-single-node-openshift).

Clone this repository and edit the `env` file with the name of your node and its local disk name.

### 2. Install Operators

Now install the [OpenShift Container Storage (OCS) operator](https://github.com/red-hat-storage/ocs-operator).
You can do this via the web dashboard under Operators -> Operator Hub, or directly:

    kubectl create namespace openshift-storage
    kubectl apply --filename=ocs-subscription.yaml

OCS is a high-level operator on top of the [Rook](https://rook.io/) Ceph operator and [NooBaa](https://www.noobaa.io/)
data federation operator. In this guide we will only be using Rook, and using it directly without OCS, because
currently OCS does not support SNO.

### 3. Create Ceph Cluster

We will now deploy our Ceph cluster via Rook:

    kubectl apply --filename=ceph-cluster.yaml

You'll see the cluster's pods created here with the prefix `rook-ceph-`:

    kubectl get pods --namespace=openshift-storage

The Ceph cluster comprises the following component types:

* **mon (monitor)**: These manage the high-level Ceph cluster's connectivity and status. Each instance manages a database about
  the cluster stored by default at `/var/lib/rook` on its host. Note that if you run multiple Ceph clusters then you must set
  each to use a different directory or otherwise make sure that their mon instances are not running on the same nodes. A quorum
  of at least 3 mon instances is required per cluster. The default for multi-node clusters it to place them on separate nodes, so
  for SNO we need to explicitly enable them to run on the same node.
* **mgr (manager)**: These provide user access to the Ceph cluster. This includes responding to the `ceph` command line tool,
  serving the web dashboard, and providing Prometheus telemetry. They can be configured with various optional modules. The Ceph
  cluster can scale these out as need. For SNO a single instance will be automatically deployed.
* **osd (object storage daemon)**: These handle low-level local disk I/O, which they expose to clients over the network. They are
  deployed on-demand for every node that is adding its local disk to the cluster and are indeed the only components that need to
  run on the storage nodes. By default multiple instances are deployed per node for redunancy but for SNO this is unnecessary,
  so we configure it to use just one.
* **mds (metadata server daemon)**: These handle a high-level distributed filesystem on top of the raw storage exposed by the osd
  instances. Internally this is achieved by creating a separate storage block for filesystem metadata. They are only deployed if
  a filesystem is explicitly created for the cluster. Note that since Ceph 16 it is possible to create multiple filesystems per
  cluster. Rook always creates double the number of mds instances we request for redundancy. For SNO it is enough to request
  one, thus we will have two running mds instances.
* **crashcollector**: These are deployed per node to collect crash information from other components. For SNO this means that
  only one will be deployed.

At this point we should have mon, mgr, osd, and crashcollector instances. We have not yet created a filesystem so there will be
no mds nodes.

You can use our shortcut to run the `ceph` command line tool, e.g.:

    ./ceph status

#### Troubleshooting: Only mon instances

If you only see mon instances and nothing else then the cluster is not up. To find out what went wrong try the operator logs:

    kubectl logs deployment/rook-ceph-operator --namespace=openshift-storage

If the mon instances are unable to form a quorom, and this is not your first time trying to create the cluster, it is likely that
the `/var/lib/rook` directory has some data from previous attempts. Rook intentionally does not wipe this directory when deleting
a cluster, so you need to do it manually. Delete the cluster and then use our script:

    kubectl delete --filename=ceph-cluster.yaml
    ./cleanup-cluster

You can then try to create the cluster again.

Note that deleting a failed cluster may result in the resources getting stuck in a "terminating" state. This is due to Rook's use
of Kubernetes finalizers to manage cleaning up of resources such as CephCluster and CephFilesystem, which are unfortunately quite
fragile. To force the deletion of the resource you may have to manually edit the resource and remove the finalizer. You might also
have to delete the current rook-ceph-operator pod (a new one will be automatically created by the ReplicaSet).

Another issue could be networking, specifically that the operator cannot reach the mon instances. This might happen if you are
using a special SDN configuration. To check that connectivity use our script:

    ./curl-mon

If successful it should show "ceph v2".

#### Troubleshooting: No osd instances

We've configured our cluster to use all our drives. Ceph will specifically look for unmounted empty, unused drives or drives that
have already been structured for Ceph. Note that "unused" means no MBR, too! If no such drives are found then you will not see any
osd instances. You can check various logs to find out what went wrong:

    kubectl logs --selector=app=rook-ceph-osd-prepare --namespace=openshift-storage
    kubectl logs deployment/rook-ceph-operator --namespace=openshift-storage
    kubectl logs deployment/rook-ceph-mon-a --namespace=openshift-storage --container=mon

Often the issue is that the drive is not truly "unused". To erase it completely see "zapping devices" in the
[documentation](https://github.com/rook/rook/blob/master/Documentation/ceph-teardown.md). For an HDD use our script:

    ./cleanup-hdd

### 4. Access Dashboard

This step is not required, but it could be useful to see the Ceph web dashboard. To enable external access to it:

    kubectl apply --filename=dashboard.yaml

Then use our script to show the URL and login credentials:

    ./dashboard

The dashboard will show you any alerts and allow you to see the general health of the Ceph cluster.

### 5. Create Filesystem

We will now create our filesystem connected to an associated storage class via CSI:

    kubectl apply --filename=filesystem.yaml

If all goes well we should finally see the mds instances starting up.

#### Troubleshooting: HEALTH_ERR: 1 filesystem is online with fewer MDS than max_md

If the `ceph status` command or Ceph dashboard shows this error then it likely means that none of the mds instances
was made "active" and all are in "standby". The complex reasons are explained
[here](https://lists.ceph.io/hyperkitty/list/ceph-users@ceph.io/thread/KQ5A5OWRIUEOJBC7VILBGDIKPQGJQIWN/), for but
for now you can use our script to fix the issue:

    ./fix-mds

After this the HEALTH_ERR should go away and the filesystem will be created.

### 6. Quick Test

To check that our `ceph` storage class works let's create a simple pod with a persistent volume claim for 600GB in
the current namespace:

    kubectl apply --filename=test.yaml

If the pod successfully starts then we are done.
