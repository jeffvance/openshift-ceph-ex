## MySQL + PVClaim in Single OSE Template:

Here is an example of how to create the mysql application using ceph rbd as the persistent store -- all defined in one OSE template file.

### Environment:
The enviromnent used for all of the examples in this repo is described [here](../ENV.md).

### Setting up Openshift Enterprise (OSE):
The steps needed to seup a simple OSE cluster with 1 master and 1 worker node are described [here](../OSE.md).

### Setting up Ceph:
The steps needed to setup ceph in a single container (AIO, all-in-one container) are described [here](../CEPH.md).

### Setting up MySQL:
Follow the instructions [here](../MYSQL.md) to initialize and validate containerized mysql.

### Defining the Template File:
Here is the [pod spec](ceph-mysql-template.yaml) which defines both the persistent volume claim (PVC) and the pod. The actual persistent volume (PV) has already been created -- [example 3](../mysql_ceph_pvc) -- and is verified here.

```
$ oc get pv
NAME                 LABELS    CAPACITY     ACCESSMODES   STATUS    CLAIM                   REASON
ceph-pv              <none>    2147483648   RWX           Bound     default/ceph-claim 
```

### Create the Template Object:

```
$ oc create -f ceph-mysql-template.yaml 
templates/ceph-mysql-template

# oc get templates
NAME                  DESCRIPTION                                                   PARAMETERS    OBJECTS
ceph-mysql-template   mysql persistent ceph application template using inline PVC   0 (all set)   2
```

Note: when re-using this template, if the pvc already exists, you'll see this error: *Error: persistentvolumeclaims "ceph-claim-template" already exists*. This error can be ignored since the pod is still started correctly.

### PVC Cleanup:
Before the mysql app can be created from the above template, we need to make sure there is a PV available for the defined template claim. The *oc get pv* command above shows that the only PV we have is Bound to the "ceph-claim", which we created in [example 3](../mysql_ceph_pvc). Therefore, we first need to delete the current PVC:

```
$ oc delete pvc ceph-claim
persistentvolumeclaims/ceph-claim

# oc get pvc
NAME            LABELS    STATUS    VOLUME

# oc get pv
NAME                 LABELS    CAPACITY     ACCESSMODES   STATUS     CLAIM                   REASON
ceph-pv              <none>    2147483648   RWX           Released   default/ceph-claim 
```

Notice that the "ceph-claim" is gone and that the "ceph-pv" is Released. This may be a bug, but currently a new claim cannot bind to a Released PV; therefore, we have to also delete and recreate the PV before we can start mysql from the tmeplate.

```
$ oc delete pv 
persistentvolumes/ceph-pv

$ oc create -f ceph-pv.yaml
persistentvolumes/ceph-pv

$ oc get pv
NAME                 LABELS    CAPACITY     ACCESSMODES   STATUS      CLAIM                   REASON
ceph-pv              <none>    2147483648   RWX           Available                           

```
The pv file used above is defined [here](../mysql_ceph_pvc/ceph-pv.yaml).

### Instantiate the Mysql Pod:
The *oc new-app* command accepts a template object (or a template file can be specified) and creates the mysql app from this template.

```
$ oc new-app ceph-mysql-template
persistentvolumeclaims/ceph-claim-template
pods/ceph-mysql-pod
Run 'oc status' to view your app.

$ oc get pods
NAME                      READY     STATUS  RESTARTS   AGE
ceph-mysql-pod            1/1       Running 0          25s

$ oc describe pod ceph-mysql-pod
Name:				ceph-mysql-pod
Namespace:			default
Image(s):			tutum/mysql
Node:				192.168.122.254/192.168.122.254
Labels:				<none>
Status:				Running
Reason:				
Message:			
IP:				10.1.0.19
Replication Controllers:	<none>
Containers:
  mysql-from-template:
    Image:		tutum/mysql
    State:		Running
      Started:		Thu, 03 Sep 2015 12:39:14 -0400
    Ready:		True
    Restart Count:	1
Conditions:
  Type		Status
  Ready 	True 
Events:
  FirstSeen				LastSeen			Count	From				SubobjectPath				Reason	Message
  Thu, 03 Sep 2015 12:38:01 -0400	Thu, 03 Sep 2015 12:38:01 -0400	1	{scheduler }								scheduled	Successfully assigned ceph-mysql-pod to 192.168.122.254
  Thu, 03 Sep 2015 12:38:03 -0400	Thu, 03 Sep 2015 12:38:03 -0400	1	{kubelet 192.168.122.254}	implicitly required container POD	pulled	Pod container image "openshift3/ose-pod:v3.0.1.0" already present on machine
...
```

Above, you can see that the pod was scheduled to run on node 192.168.122.254 (often the hostname is shown here). On the target openshift-node you can see the container being run as follows:

```
$ docker ps
CONTAINER ID        IMAGE                         COMMAND             CREATED             STATUS            PORTS              NAMES
bcd54ae963c2        tutum/mysql                   "/run.sh"           4 seconds ago       Up 2 seconds                        k8s_mysql-from-template.55d85000_ceph-mysql-pod_default_24320626-525a-11e5-9d3b-52540039f12e_612d72e5
```

Using the above container ID, we can inspect the docker logs, and then run a shell inside this container to show the ceph/rbd mount and to access a simple database (previously created), as follows:

```
$ docker logs bcd54ae963c2
=> Using an existing volume of MySQL
=> Starting MySQL ...
=> Waiting for confirmation of MySQL service startup, trying 0/60 ...
=> Waiting for confirmation of MySQL service startup, trying 1/60 ...
tail -F $LOG
150903 16:48:39  InnoDB: Waiting for the background threads to start
150903 16:48:40 InnoDB: 5.5.44 started; log sequence number 1599201
150903 16:48:40 [Note] Server hostname (bind-address): '0.0.0.0'; port: 3306
150903 16:48:40 [Note]   - '0.0.0.0' resolves to '0.0.0.0';
150903 16:48:40 [Note] Server socket created on IP: '0.0.0.0'.
150903 16:48:40 [Warning] 'user' entry 'root@ceph-mysql' ignored in --skip-name-resolve mode.
150903 16:48:40 [Warning] 'proxies_priv' entry '@ root@ceph-mysql' ignored in --skip-name-resolve mode.
150903 16:48:40 [Note] Event Scheduler: Loaded 0 events
150903 16:48:40 [Note] /usr/sbin/mysqld: ready for connections.
Version: '5.5.44-0ubuntu0.14.04.1'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  (Ubuntu)

$ docker exec -it bcd54ae963c2 bash
root@ceph-mysql-pod:/# ls /var/lib/mysql/
ib_logfile0  ibdata1     mysql               us_states
ib_logfile1  lost+found  performance_schema

root@ceph-mysql-pod:/# mount | grep rbd
/dev/rbd0 on /var/lib/mysql type ext4 (rw,relatime,seclabel,stripe=1024,data=ordered)

root@ceph-mysql-pod:/# mysql
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.5.44-0ubuntu0.14.04.1 (Ubuntu)
...
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

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

mysql> show tables;
+---------------------+
| Tables_in_us_states |
+---------------------+
| states              |
+---------------------+
1 row in set (0.00 sec)

mysql> select * from states;
+----+---------+------------+
| id | state   | population |
+----+---------+------------+
|  1 | Alabama |    4822023 |
+----+---------+------------+
1 row in set (0.00 sec)

mysql> quit
Bye
root@ceph-mysql-pod:/# exit
exit
```

