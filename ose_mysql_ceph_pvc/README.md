## Example 5: Using openshift/mysql, Ceph Persistent Volume and Claim

This example is similar to [example 3](../mysql_ceph_pvc) except that we're using the official openshift mysql image. This image has the UID hard-coded to 27 in the Dockerfile, and as a result presents challenges from a permissions/volume-mounting perspective. The solution to running the mysql container as non-root is to separate the creation of the pod from the creation of the bind mounts for the container. This is accomplished by using the *oc new-app* command to create the pod with the default emptyDir volume binding, followed by *oc volume* to replace emptyDir with the ceph rbd plugin binding.

*oc new-app* in this example creates the following:
  * a new service named "mysql-55-centos7"
  * a new replication controller named "mysql-55-centos7-1"
  * a new deployment configuration named "mysql-55-centos7"
  * a new imageStream named "mysql-55-centos7" 
  * an emptyDir volume named ""
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
```

### Creating the Pod:
The [pod spec](ceph-mysql-pvc-pod.yaml) references the same mysql image and defines the named claim to be used for persistent storage. As with [example 2](../mysql_ceph_plugin), the mysql container needs to run privileged. *oc create* is used to create the pod:

```
#on the OSE-master:
$ oc create -f ceph-mysql-pvc-pod.yaml 
pods/ceph-mysql

$ oc get pods
NAME                      READY     STATUS       RESTARTS   AGE
ceph-mysql                1/1       Running                                      
```

Volume information is also visible on the OSE-master:

```
#on the OSE-master:
$ oc volume pod ceph-mysql --list
# pods ceph-mysql, volumes:
mysql-pv
default-token-sqef4
	# container ceph-mysql, volume mounts:
	mysql-pv /var/lib/mysql
	default-token-sqef4 /var/run/secrets/kubernetes.io/serviceaccount
```

We execute *oc describe pod* to see which OSE host the pod is running on, and to see the pod's recent events:

```
#on the OSE-master:
$ oc describe pod ceph-mysql
Name:				ceph-mysql
Namespace:			default
Image(s):			mysql
Node:				192.168.122.254/192.168.122.254  # <--- often both hostname and ip are shown here
Labels:				<none>
Status:				Running
Reason:				
Message:			
IP:				10.1.0.43
Replication Controllers:	<none>
Containers:
  ceph-mysql:
    Image:		mysql
    State:		Running
      Started:		Fri, 04 Sep 2015 14:47:31 -0400
    Ready:		True
    Restart Count:	0
Conditions:
  Type		Status
  Ready 	True 
Events:
  FirstSeen				LastSeen			Count	From				SubobjectPath				Reason	Message
  Fri, 04 Sep 2015 14:47:22 -0400	Fri, 04 Sep 2015 14:47:22 -0400	1	{scheduler }								scheduled	Successfully assigned ceph-mysql to 192.168.122.254
  Fri, 04 Sep 2015 14:47:24 -0400	Fri, 04 Sep 2015 14:47:24 -0400	1	{kubelet 192.168.122.254}	implicitly required container POD	pulled	Pod container image "openshift3/ose-pod:v3.0.1.0" already present on machine
  Fri, 04 Sep 2015 14:47:26 -0400	Fri, 04 Sep 2015 14:47:26 -0400	1	{kubelet 192.168.122.254}	implicitly required container POD	createdCreated with docker id acbee2db9a18
  Fri, 04 Sep 2015 14:47:27 -0400	Fri, 04 Sep 2015 14:47:27 -0400	1	{kubelet 192.168.122.254}	implicitly required container POD	startedStarted with docker id acbee2db9a18
  Fri, 04 Sep 2015 14:47:30 -0400	Fri, 04 Sep 2015 14:47:30 -0400	1	{kubelet 192.168.122.254}	spec.containers{ceph-mysql}		createdCreated with docker id 9a43017dbebf
  Fri, 04 Sep 2015 14:47:31 -0400	Fri, 04 Sep 2015 14:47:31 -0400	1	{kubelet 192.168.122.254}	spec.containers{ceph-mysql}		startedStarted with docker id 9a43017dbebf
```

We see that the pod was scheduled on OSE host 192.168.122.254 (often there is a hostname visible too). On the target OSE node verify that the mysql container is running and that it's using the ceph-rbd volume:

```
#on the target/scheduled OSE-node:
$ docker ps
CONTAINER ID        IMAGE                         COMMAND                CREATED             STATUS              PORTS               NAMES
9a43017dbebf        mysql                         "/entrypoint.sh mysq   4 minutes ago       Up 4 minutes                            k8s_ceph-mysql.d8cb6e3e_ceph-mysql_default_608af6aa-5335-11e5-b56b-52540039f12e_b0163abc   
acbee2db9a18        openshift3/ose-pod:v3.0.1.0   "/pod"                 4 minutes ago       Up 4 minutes                            k8s_POD.dbbbe7c7_ceph-mysql_default_608af6aa-5335-11e5-b56b-52540039f12e_d030f08f 
```

The mysql container ID is 9a43017dbebf. More details on this container are availble via *docker inspect container-ID-or-name*. Log information is shown via *docker logs container-ID*, and via the *systemctl status openshift-node -l* and *journalctl -xe -u openshift-node docker* commands.

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

