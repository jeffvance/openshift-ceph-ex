## OpenShift Enterprise + MySQL + Ceph Persistent Volume

This repo contains examples showing how to run MySQL in a container directly from docker, via a kubernetes pod or template, both using local host storage or using ceph-rbd block storage to persist the database.

The next few sections are common across almost all of the examples and are also shown below:

### Environment:
The enviromnent used for all of the examples in this repo is described [here](ENV.md).

### Setting up Openshift Enterprise (OSE):
The steps needed to seup a simple OSE cluster with 1 master and 1 worker node are described [here](OSE.md).

### Setting up Ceph:
Most of the examples use ceph-rbd within a single container (AIO, all-in-one container), and the steps needed to setup ceph are described [here](CEPH.md).

### Setting up MySQL:
Follow the instructions [here](MYSQL.md) to initialize and validate containerized mysql.

### Specific Examples:
* [mysql + local/host storage](mysql_ceph_host) - mysql database lives on the OSE host where the pod is scheduled
* [mysql + ceph plugin](mysql_ceph_plugin) - mysql database resides in ceph, a rbd plugin is specfied
* [mysql + ceph + pvc](mysql_ceph_pvc) - mysql database resides in ceph, a Persistent Volume (PV) and Persistent Volume Claim (PVC) are used
* [mysql + ceph + template](mysql_ceph_template) -- same as the above example except the pod and pvc are defined in a single template file

