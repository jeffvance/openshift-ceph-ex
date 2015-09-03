# Openshift Enterprise + MySQL + Ceph Persistent Volume

This repo is an example of how to, using the openshift cli (oc), create a very simple mysql application which uses ceph rbd as the persistent store.

## Environment:
* 2 RHEL 7 VMs for running the openshift enterprise (ose) master and node hosts
* 1 Fedora 21 VM for running the ceph all-in-one (AIO) container

## Setting up Openshift Enterprise (OSE):
Please follow the steps described in [Scott Creely's document](https://github.com/screeley44/origin/tree/storage_docs/examples/storage-examples/gluster-examples). Remember we are using ceph, not glusterfs, so all gluster steps are skipped.

## Setting up Ceph:
The steps need to setup ceph in a single container are described at the beginning of [this posting](https://jeffhvance.wordpress.com/2015/07/30/containerized-ceph-kubernetes-mysql/?preview=true&preview_id=4&preview_nonce=f47ab9541e), which is similar to what we'll be doing here, except that kubernetes is used directly in that example, rather than using OSE.

## Setting up MySQL:
The [same posting above](https://jeffhvance.wordpress.com/2015/07/30/containerized-ceph-kubernetes-mysql/?preview=true&preview_id=4&preview_nonce=f47ab9541e) also shows how to set up mysql. The tutum/mysql image 
