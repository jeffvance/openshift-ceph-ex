## MySQL + Ceph + Persistent Volume (PV) + PV Claim (PVC)

Here is an example of how to create the mysql application, using ceph rbd as the persistent store, where the ceph rbd is defined in a PV, and the pod uses a claim rather than defining the rbd volume inline. The kubernetetes PVClaimBinder matches the pod's claim against PVs and binds the claim to the PV that has the best match. Today, the matching criteria is very simple -- capacity and sharing attributes -- but, hopefully richer PV and PVC definitions will be coming in the future.  See  the [kubernetes persistent storage](https://github.com/kubernetes/kubernetes/blob/master/docs/design/persistent-storage.md) document for more information on PVs and PVCs.

### Environment:
The enviromnent used for all of the examples in this repo is described [here](../ENV.md).

### Setting up Openshift Enterprise (OSE):
The steps needed to seup a simple OSE cluster with 1 master and 1 worker node are described [here](../OSE.md).

### Setting up Ceph:
The steps needed to setup ceph in a single container (AIO, all-in-one container) are described [here](../CEPH.md).

### Setting up MySQL:
Follow the instructions [here](../MYSQL.md) to initialize and validate containerized mysql.

### Defining the PV and PVC Files:
A persistent volume is created from a file defining the name, capacity, and access methods for persistent storage. The PV spec used in this example is [here](ceph-mysql-pv.yaml), and the PVC is [here](ceph-mysql-claim.yaml).

PVs are typically created by an OSE administrator, whereas PVCs will typically be created and requested by non-admins. The example here creates both the PV and claim separate from the pod. There is also a [template](../mysql_ceph_template) example which defines the PVC in the same file used to define the pod.

uses the same mysql image, defines the password as an environment variable, and maps the container's volume (/var/lib/mysql) to the host's volume (/opt/mysql) where the database resides:


