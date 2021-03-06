## Example 5: Using openshift/mysql, Ceph Persistent Volume and Claim

This example is similar to [example 3](../mysql_ceph_pvc) except that we're using the official openshift mysql image. This image has the UID hard-coded to 27 in the Dockerfile, and as a result presents challenges from a permissions/volume-mounting perspective. The solution to running the mysql container as non-root is to separate the creation of the pod from the creation of the bind mounts for the container. This is accomplished by using the *oc new-app* command to create the pod with the default emptyDir volume binding, followed by *oc volume* to replace emptyDir with the ceph rbd plugin binding.

*oc new-app* in this example creates the following:
  * a new service named "mysql-55-centos7"
  * a new replication controller named "mysql-55-centos7-1"
  * a new deployment configuration named "mysql-55-centos7"
  * a new imageStream named "mysql-55-centos7" 
  * an emptyDir volume named "mysql-55-centos7-volume-1"
  * the target mysql pod named "mysql-55-centos7-1-<random-letters-here>".

### Environment:
If the steps to install the environment, ceph, ose and mysql have not already been completed, then follow the instuctions linked-to directly below:

The enviromnent used for all of the examples in this repo is described [here](../ENV.md).

### Setting up Openshift Enterprise (OSE):
The steps needed to seup a simple OSE cluster with 1 master and 1 worker node are described [here](../OSE.md).

### Setting up Ceph:
The steps needed to setup ceph in a single container (AIO, all-in-one container) are described [here](../CEPH.md).

### Setting up MySQL:
Follow the instructions [here](../MYSQL.md) to initialize and validate containerized openshift/mysql. Be sure to *docker pull* the openshift/mysql image.

### Defining the PV and PVC Files:
The [PVC](../mysql_ceph_pvc/ceph-claim.yaml) created here is the same used in [example 3](../mysql_ceph_pvc); however, the [PV](mysql-55-centos7-volume-1.yaml) has a different name to coincide with the default naming used by the *oc new-app* command. PVs are typically created by an OSE administrator, whereas PVCs will typically be created and requested by non-admins. The example here creates both the PV and claim separate from the pod.

### Creating the PV and PVC:
*oc create -f* is execute on the OSE-master to create almost all OSE objects, and is used here to create the ceph PV and PVC.

```
#on the OSE-master:
$ oc create -f mysql-55-centos7-volume-1.yaml
persistentvolumes/mysql-55-centos7-volume-1

$ oc get pv
NAME                        LABELS    CAPACITY     ACCESSMODES   STATUS      CLAIM     REASON
mysql-55-centos7-volume-1   <none>    1073741824   RWX           Available      

$ oc create -f ../mysql_ceph_pvc/ceph-claim.yaml
persistentvolumeclaims/ceph-claim

$ oc get pvc
NAME         LABELS    STATUS    VOLUME
ceph-claim   map[]     Bound     mysql-55-centos7-volume-1
```

### Creating the openshift/mysql App:
The *oc new-app* command is used to create the openshift/mysql pod along with the other objects listed above.

```
$ oc new-app -e MYSQL_USER=mysql,MYSQL_PASSWORD=foopass,MYSQL_DATABASE=mysql openshift/mysql-55-centos7
NOTICE: Image "mysql-55-centos7" uses an EmptyDir volume. Data in EmptyDir volumes is not persisted across deployments.
imagestreams/mysql-55-centos7
deploymentconfigs/mysql-55-centos7
services/mysql-55-centos7
Service "mysql-55-centos7" created at 172.30.253.191 with port mappings 3306.
Run 'oc status' to view your app.

$ oc status
In project default
service/kubernetes - 172.30.0.1:443
service/mysql-55-centos7 - 172.30.253.191:3306
  dc/mysql-55-centos7 deploys istag/mysql-55-centos7:latest 
    #1 deployment pending 6 seconds ago
dc/docker-registry deploys registry.access.redhat.com/openshift3/ose-docker-registry:v3.0.1.0 
  #2 deployed 2 weeks ago - 1 pod
To see more, use 'oc describe <resource>/<name>'.
You can use 'oc get all' to see a list of other objects.

#list pods:
$ oc get pods
NAME                       READY     STATUS                                                                                                      RESTARTS   AGE
docker-registry-2-a57pb    0/1       Image: registry.access.redhat.com/openshift3/ose-docker-registry:v3.0.1.0 is ready, container is creating   0          1h
mysql-55-centos7-1-nmhmk   1/1       Running                                    

#list services:
$ oc get svc
NAME               LABELS                                    SELECTOR                            IP(S)            PORT(S)
kubernetes         component=apiserver,provider=kubernetes   <none>                              172.30.0.1       443/TCP
mysql-55-centos7   app=mysql-55-centos7                      deploymentconfig=mysql-55-centos7   172.30.253.191   3306/TCP

#list replication controllers:
$ oc get rc
CONTROLLER           CONTAINER(S)       IMAGE(S)                                                             SELECTOR                                                                                REPLICAS
docker-registry-2    registry           registry.access.redhat.com/openshift3/ose-docker-registry:v3.0.1.0   deployment=docker-registry-2,deploymentconfig=docker-registry,docker-registry=default   1
mysql-55-centos7-1   mysql-55-centos7   openshift/mysql-55-centos7:latest                                    deployment=mysql-55-centos7-1,deploymentconfig=mysql-55-centos7 

#list deployment configurations:
$ oc get dc
NAME               TRIGGERS                    LATEST VERSION
docker-registry    ConfigChange                2
mysql-55-centos7   ConfigChange, ImageChange   1
```

### Volume Binding:
Now that the mysql pod has been successfully created (using the *emptyDir* volume), it's time to update the volume bind mount to reference the ceph volume claim rather than emptyDir.

Volume information is visible on the OSE-master:

```
#on the OSE-master:
#list all volumes based on pod selector:
$ oc volume pod/mysql-55-centos7-1-nmhmk --list
# pods mysql-55-centos7-1-nmhmk, volumes:
mysql-55-centos7-volume-1
default-token-1qip2
	# container mysql-55-centos7, volume mounts:
	mysql-55-centos7-volume-1 /var/lib/mysql/data
	default-token-1qip2 /var/run/secrets/kubernetes.io/serviceaccount

#list the volume base on dc selector:
$ oc volume dc/mysql-55-centos7 --list
# deploymentconfigs mysql-55-centos7, volumes:
mysql-55-centos7-volume-1
	# container mysql-55-centos7, volume mounts:
	mysql-55-centos7-volume-1 /var/lib/mysql/data

#list more details about the volume:
$ oc describe dc mysql-55-centos7
Name:		mysql-55-centos7
Created:	31 minutes ago
Labels:		app=mysql-55-centos7
Latest Version:	1
Triggers:	Config, Image(mysql-55-centos7@latest, auto=true)
Strategy:	Rolling
Template:
  Selector:	deploymentconfig=mysql-55-centos7
  Replicas:	1
  Containers:
  NAME			IMAGE					ENV
  mysql-55-centos7	openshift/mysql-55-centos7:latest	MYSQL_DATABASE=mysql,MYSQL_PASSWORD=pass,MYSQL_USER=mysql
Deployment #1 (latest):
	Name:		mysql-55-centos7-1
	Created:	31 minutes ago
	Status:		Complete
	Replicas:	1 current / 1 desired
	Selector:	deployment=mysql-55-centos7-1,deploymentconfig=mysql-55-centos7
	Labels:		app=mysql-55-centos7,openshift.io/deployment-config.name=mysql-55-centos7
	Pods Status:	1 Running / 0 Waiting / 0 Succeeded / 0 Failed
...
```

Notice that the default volume name is the image name with "-volume-*N*" appended.

Use the *oc volume* command to bind the running pod (whose container has already run its entrypoint initialization script) to the ceph volume:

```
$ oc volume dc/mysql-55-centos7 --add --name=mysql-55-centos7-volume-1 -t persistentVolumeClaim --claim-name=ceph-claim --mount-path=/var/lib/mysql/data --overwrite
deploymentconfigs/mysql-55-centos7
```


The container's rbd mounts are visible directly from the host and from within the container itself. On the OSE host:

```
#on the target/scheduled OSE-node:
$ mount | grep rbd
/dev/rbd0 on /var/lib/openshift/openshift.local.volumes/plugins/kubernetes.io/rbd/rbd/rbd-image-foo type ext4 (rw,relatime,seclabel,stripe=1024,data=ordered)
/dev/rbd0 on /var/lib/openshift/openshift.local.volumes/pods/608af6aa-5335-11e5-b56b-52540039f12e/volumes/kubernetes.io~rbd/ceph-pv type ext4 (rw,relatime,seclabel,stripe=1024,data=ordered)
```

And, shelling into the container:

```
#on the target/scheduled OSE-node:
$ docker exec -it 9a43017dbebf bash
root@ceph-mysql:/# mount | grep rbd
/dev/rbd0 on /var/lib/mysql type ext4 (rw,relatime,seclabel,stripe=1024,data=ordered)
root@ceph-mysql:/# exit
exit
```

Mysql can also be run in the container as follows:

```
#on the target/scheduled OSE-node:
$ docker exec -it 9a43017dbebf bash
root@ceph-mysql:/# mysql                                                       
Welcome to the MySQL monitor.  Commands end with ; or \g.
...
mysql> show databases;
+---------------------+
| Database            |
+---------------------+
| information_schema  |
| #mysql50#lost+found |
| mysql               |
| performance_schema  |
| us_states           |
+---------------------+
5 rows in set (0.00 sec)

mysql> use us_states;
Reading table information for completion of table and column names
...
mysql> select * from states;
+----+---------+------------+
| id | state   | population |
+----+---------+------------+
|  1 | Alabama |    4822023 |
+----+---------+------------+
1 row in set (0.00 sec)

mysql> quit
Bye
root@ceph-mysql:/# exit
exit
```

