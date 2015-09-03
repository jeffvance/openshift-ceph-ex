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
The easiest way to start this simple mysql example is to use an OSE template file to define the app. The [template](ceph-mysql-template.yaml) specifies the tutum/mysql image, the pod, and the ceph volume claim to be used.

Note: when re-using this template, if the pvc already exists you'll see this error "Error: persistentvolumeclaims "ceph-claim-template" already exists". This error can be ignored since the pod is still started correctly.

Create the mysql app as follows:

```
$ oc new-app ceph-mysql-template
pods/ceph-mysql-pod

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

Above, you can see that the pod was scheduled to run on node 192.168.122.254 (often the hostname is shown here). 
On the target openshift-node you can see the container being run as follows:

```
$ docker ps
CONTAINER ID        IMAGE                         COMMAND             CREATED             STATUS            PORTS              NAMES
bcd54ae963c2        tutum/mysql                   "/run.sh"           4 seconds ago       Up 2 seconds                        k8s_mysql-from-template.55d85000_ceph-mysql-pod_default_24320626-525a-11e5-9d3b-52540039f12e_612d72e5
```

Using the above container ID, we can inspect the docker logs and then run a shell inside this container and access a simple database (previously created), as follows:

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





```
