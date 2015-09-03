## Starting MySQL Using an OSE Template

Here is an example of how to, using the openshift cli (oc), create a the mysql application using ceph rbd as the persistent store -- all defined in one OSE template file.

### Environment:
* 2 RHEL 7 VMs for running the openshift enterprise (ose) master and node hosts
* 1 Fedora 21 VM for running the ceph all-in-one (AIO) container

### Setting up Openshift Enterprise (OSE):
The steps needed to seup a simple OSE cluster with 1 master and 1 worker node are described [here](link).

### Setting up Ceph:
The steps needed to setup ceph in a single container (AIO, all-in-one container) are described [here](link).

### Setting up MySQL:
Follow the instructions [here](link) to initialize and validate containerized mysql.

OSE's Security Context Constraints (SSC) need to be defined such that the *seLinuxContext* and *runAsUser* values are set to "RunAsAny". Selinux is still enabled/enforcing on the master and worker-node hosts.

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

The RHEL-7 hosts running the OSE master and OSE nodes should have the following services enabled and running:
* selinux (*setenforce 1*)
* iptables (*systemctl start iptables*)
* firewalld (*systemctl start firewalld*) Note, if you cannot start firewalld due to the service being masked, you can do a *systemctl unmask firewalld* and then restart it.

The simple example below uses the official mysql image found [here](https://hub.docker.com/_/mysql/). First, pull down the mysql image on each of your Openshift Enterprise (OES) worker nodes:
```
docker pull mysql
```

Next, test to ensure that mysql can be run from a containers:
```
$ docker run --name mysql -e MYSQL_ROOT_PASSWORD=foopass -d mysql
```
Note: if you re-run the above you will first need to remove the archived container so that the container name, "mysql", can be reused:
```
$ docker ps -a
$ docker rm <mysql-container-ID>

# to remove all containers:
$ docker rm $(docker ps -a)
```

Shell into the mysql container and run mysql:
```
$ docker exec -it <mysql-container-ID> bash
bash# mysql -p  # -p needed since a root password was supplied above
mysql> show datbases;
mysql> quit
bash# exit
```

Delete the mysql container so we can create it again, but this time from a pod
```
$ docker rm <mysql-container-ID>
```

Here is a simple pod spec which uses the same mysql image, defines the password as an environment variable, and maps the container's volume (/var/lib/mysql) to the host's volume (/opt/mysql) where the database resides:
```
apiVersion: v1
kind: Pod
metadata:
  name: mysql
  labels: 
    name: mysql
spec: 
  containers: 
    - image: mysql
      name: mysql
      volumeMounts:
        - name: varlibmysql
          mountPath: /var/lib/mysql
      env:
        - name: MYSQL_ROOT_PASSWORD
          value: foopass
  volumes:
    - name: varlibmysql
      hostPath: 
        path: /opt/mysql
```

Before we can create this pod we need to set the selinux context on the OSE host's directory (/opt/mysql) where the database lives. Selinux should remain enabled/enforcing:
```
$ chcon -Rt svirt_sandbox_file_t /opt/mysql
$ ls -dZ /opt/mysql
drwxr-xr-x. polkitd ssh_keys system_u:object_r:svirt_sandbox_file_t:s0 /opt/mysql

$ getenforce
Enforcing
```

Now we can create the mysql pod:
```
$ oc create -f mysql.yaml 
pods/mysql

$ oc get pod
NAME                      READY     STATUS          RESTARTS   AGE
mysql                     1/1       Running         0          18s
```

To see which OSE host the mysql pod has been scheduled on:
```
$ oc describe pod mysql
NAME                      READY     STATUS                                                                                               RESTARTS   AGE
docker-registry-2-223nv   0/1       Image: registry.access.redhat.com/openshift3/ose-docker-registry:v3.0.1.0 is not ready on the node   0          3d
mysql                     1/1       Running                                                                                              0          18s
[root@rhel7-ose-1 ceph]# oc describe pod mysql
Name:				mysql
Namespace:			default
Image(s):			mysql
Node:				192.168.122.254/192.168.122.254  ## <--- the hostname is often shown here
Labels:				name=mysql
Status:				Running
Reason:				
Message:			
IP:				10.1.0.41
Replication Controllers:	<none>
Containers:
  mysql:
    Image:		mysql
    State:		Running
      Started:		Thu, 03 Sep 2015 19:10:15 -0400
    Ready:		True
    Restart Count:	0
Conditions:
  Type		Status
  Ready 	True 
Events:
  FirstSeen				LastSeen			Count	From				SubobjectPath				Reason	Message
  Thu, 03 Sep 2015 19:10:07 -0400	Thu, 03 Sep 2015 19:10:07 -0400	1	{scheduler }								scheduled	Successfully assigned mysql to 192.168.122.254
...
  Thu, 03 Sep 2015 19:10:15 -0400	Thu, 03 Sep 2015 19:10:15 -0400	1	{kubelet 192.168.122.254}	spec.containers{mysql}			startedStarted with docker id 77f4af567e3d
```

On the scheduled OSE host run docker to get information about the mysql container:
```
$ docker ps
CONTAINER ID        IMAGE                         COMMAND                CREATED             STATUS              PORTS               NAMES
77f4af567e3d        mysql                         "/entrypoint.sh mysq   5 minutes ago       Up 5 minutes                            k8s_mysql.4977675e_mysql_default_ea9b64de-5290-11e5-b56b-52540039f12e_2257d0b5   
dca749fa3530        openshift3/ose-pod:v3.0.1.0   "/pod"                 5 minutes ago       Up 5 minutes                            k8s_POD.892ec37e_mysql_default_ea9b64de-5290-11e5-b56b-52540039f12e_aa534a81

$ docker inspect mysql
[
{
    "Id": "7eee2d462c8f6ffacfb908cc930559e21778f60afdb2d7e9cf0f3025274d7ea8",
    "Parent": "15a3cddfc178c4dbaa8f56142d4eebef6d22a3cd1842820844cf815992fe5a13",
    "Comment": "",
    "Created": "2015-08-24T21:55:13.704277966Z",
    "Container": "113d16e2420bb0f5e17a74f5f8c85d70572efe1720451da83e0a84fe3fcd04fd",
    "ContainerConfig": {
        "Hostname": "c6ebf900c860",
        "Domainname": "",
        "User": "",
        "AttachStdin": false,
        "AttachStdout": false,
        "AttachStderr": false,
        "PortSpecs": null,
        "ExposedPorts": {
            "3306/tcp": {}
        },
        "Tty": false,
        "OpenStdin": false,
        "StdinOnce": false,
        "Env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
            "MYSQL_MAJOR=5.6",
            "MYSQL_VERSION=5.6.26"
        ],
        "Cmd": [
            "/bin/sh",
            "-c",
            "#(nop) CMD [\"mysqld\"]"
        ],
        "Image": "15a3cddfc178c4dbaa8f56142d4eebef6d22a3cd1842820844cf815992fe5a13",
        "Volumes": {
            "/var/lib/mysql": {}
        },
        "VolumeDriver": "",
        "WorkingDir": "",
        "Entrypoint": [
            "/entrypoint.sh"
        ],
        "NetworkDisabled": false,
        "MacAddress": "",
        "OnBuild": [],
        "Labels": {},
        "Init": ""
    },
    "DockerVersion": "1.7.1",
    "Author": "",
    "Config": {
        "Hostname": "c6ebf900c860",
        "Domainname": "",
        "User": "",
        "AttachStdin": false,
        "AttachStdout": false,
        "AttachStderr": false,
        "PortSpecs": null,
        "ExposedPorts": {
            "3306/tcp": {}
        },
        "Tty": false,
        "OpenStdin": false,
        "StdinOnce": false,
        "Env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
            "MYSQL_MAJOR=5.6",
            "MYSQL_VERSION=5.6.26"
        ],
        "Cmd": [
            "mysqld"
        ],
        "Image": "15a3cddfc178c4dbaa8f56142d4eebef6d22a3cd1842820844cf815992fe5a13",
        "Volumes": {
            "/var/lib/mysql": {}
        },
        "VolumeDriver": "",
        "WorkingDir": "",
        "Entrypoint": [
            "/entrypoint.sh"
        ],
        "NetworkDisabled": false,
        "MacAddress": "",
        "OnBuild": [],
        "Labels": {},
        "Init": ""
    },
    "Architecture": "amd64",
    "Os": "linux",
    "Size": 0,
    "VirtualSize": 283575255
}
]

$ docker logs 77f4af567e3d   # <--- container ID
2015-09-03 23:10:17 0 [Note] mysqld (mysqld 5.6.26) starting as process 1 ...
2015-09-03 23:10:17 1 [Note] Plugin 'FEDERATED' is disabled.
2015-09-03 23:10:17 1 [Note] InnoDB: Using atomics to ref count buffer pool pages
2015-09-03 23:10:17 1 [Note] InnoDB: The InnoDB memory heap is disabled
2015-09-03 23:10:17 1 [Note] InnoDB: Mutexes and rw_locks use GCC atomic builtins
2015-09-03 23:10:17 1 [Note] InnoDB: Memory barrier is not used
2015-09-03 23:10:17 1 [Note] InnoDB: Compressed tables use zlib 1.2.7
2015-09-03 23:10:17 1 [Note] InnoDB: Using Linux native AIO
2015-09-03 23:10:17 1 [Note] InnoDB: Using CPU crc32 instructions
2015-09-03 23:10:17 1 [Note] InnoDB: Initializing buffer pool, size = 128.0M
2015-09-03 23:10:17 1 [Note] InnoDB: Completed initialization of buffer pool
2015-09-03 23:10:17 1 [Note] InnoDB: Highest supported file format is Barracuda.
2015-09-03 23:10:18 1 [Note] InnoDB: 128 rollback segment(s) are active.
2015-09-03 23:10:18 1 [Note] InnoDB: Waiting for purge to start
2015-09-03 23:10:18 1 [Note] InnoDB: 5.6.26 started; log sequence number 1626017
2015-09-03 23:10:18 1 [Note] Server hostname (bind-address): '*'; port: 3306
2015-09-03 23:10:18 1 [Note] IPv6 is available.
2015-09-03 23:10:18 1 [Note]   - '::' resolves to '::';
2015-09-03 23:10:18 1 [Note] Server socket created on IP: '::'.
2015-09-03 23:10:18 1 [Warning] 'proxies_priv' entry '@ root@mysql' ignored in --skip-name-resolve mode.
2015-09-03 23:10:18 1 [Note] Event Scheduler: Loaded 0 events
2015-09-03 23:10:18 1 [Note] mysqld: ready for connections.
Version: '5.6.26'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server (GPL)
```

Finally, run mysql inside the container:

```
$ docker exec -it 77f4af567e3d bash  # <--- container ID again
root@mysql:/# mysql -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
...
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> quit
Bye
root@mysql:/# exit
exit
```

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


