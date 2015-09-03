## Openshift Enterprise + MySQL + Ceph Persistent Volume

This repo is an example of how to, using the openshift cli (oc), create a very simple mysql application which uses ceph rbd as the persistent store.

### Environment:
* 2 RHEL 7 VMs for running the openshift enterprise (ose) master and node hosts
* 1 Fedora 21 VM for running the ceph all-in-one (AIO) container

### Setting up Openshift Enterprise (OSE):
Please follow the steps described in [Scott Creely's document](https://github.com/screeley44/origin/tree/storage_docs/examples/storage-examples/gluster-examples). Remember we are using ceph, not glusterfs, so all gluster steps are skipped.

### Setting up Ceph:
The steps need to setup ceph in a single container are described at the beginning of [this posting](https://jeffhvance.wordpress.com/2015/07/30/containerized-ceph-kubernetes-mysql/?preview=true&preview_id=4&preview_nonce=f47ab9541e), which is similar to what we'll be doing here, except that kubernetes is used directly in that example, rather than using OSE.

### Setting up MySQL:
The [same posting above](https://jeffhvance.wordpress.com/2015/07/30/containerized-ceph-kubernetes-mysql/?preview=true&preview_id=4&preview_nonce=f47ab9541e) also shows how to set up mysql. The tutum/mysql image's *run.sh* script has two chmod's near the beginning, and these commands require that OSE's Security Context Constraints (SSC) are defined such that the *seLinuxContext* and *runAsUser* values are set to "RunAsAny". Note: selinux is still enabled on the master and node hosts.

```
$ oc login -u admin
$ oc edit scc prvilege
$ oc edit scc restricted
#change "MustRunAsRange" to "RunAsAny"

$ oc get scc
NAME         PRIV      CAPS      HOSTDIR   SELINUX    RUNASUSER
privileged   true      []        true      RunAsAny   RunAsAny
restricted   false     []        false     RunAsAny   RunAsAny
```
**Note:**
The RHEL-7 hosts running the OSE master and OSE node can and should have the following services enabled and running:
* selinux (*setenforce 1*)
* iptables (*systemctl start iptables*)
* firewalld (*systemctl start firewalld*) Note, if you cannot start firewalld due to the service being masked, you can do a *systemctl unmask firewalld* and then restart it.

###MySQL Template:
The easiest way to start this simple mysql is to use an OSE template file to define the app. The [template](link) specifies the tutum mysql image, the pod, and the ceph volume claim to be used.

```
# cat ceph-mysql-template.yaml 

apiVersion: v1
kind: Template
metadata:
  name: ceph-mysql-template
  annotations:
    description: "mysql persistent ceph application template using inline PVC"
    tags: "ceph mysql pvc"

objects:
- apiVersion: v1
  id: ceph-claim-template
  kind: PersistentVolumeClaim
  metadata:
    name: ceph-claim-template
  spec:
    accessModes:
    - ReadWriteMany
    resources:
      requests:
        storage: 1Gi

- apiVersion: v1
  id: ceph-mysql-pod
  kind: Pod
  metadata:
    name: ceph-mysql-pod
  spec:
    containers:
    - image: tutum/mysql
      name: mysql-from-template
      volumeMounts:
      - mountPath: /var/lib/mysql
        name: mysql-pv
    volumes:
    - name: mysql-pv
      persistentVolumeClaim:
        claimName: ceph-claim-template
```

note: when using the template, if the pvc already exists you'll see this error "Error: persistentvolumeclaims "ceph-claim-template" already exists", but the pod is still launched okay.


