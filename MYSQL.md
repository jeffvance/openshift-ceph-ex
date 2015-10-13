## Setting Up and Validating Containerized MySQL:

First, OSE's Security Context Constraints (SSC) need to be defined such that the *seLinuxContext* and *runAsUser* values are set to "RunAsAny". Selinux is still enabled/enforcing on the master and worker-node hosts, as decribed in the [OSE environment readme](ENV.md).

Our mysql example uses the docker mysql image found [here](https://hub.docker.com/_/mysql/). There is also an "official" [openshift/mysql](https://hub.docker.com/r/openshift/mysql-55-centos7/) image available ([repo](https://github.com/openshift/mysql)), which is more restrictive in that the container's UID is hardcoded to 27 (arbitrary). Almost all of the examples in this repo use the docker mysql image, but, to get a feel for some of the issues when the container's UID is not root (0), the openshift/docker image provides insights.

First, pull down the desired mysql image on *each* of your OSE worker nodes:

```
#on *each* OSE-node:
$ docker pull mysql
```

Next, test to ensure that mysql can be run in a container:

```
$ docker run --name mysql -e MYSQL_ROOT_PASSWORD=foopass -d mysql
```

Note: if you re-run the above you will first need to remove the archived container so that the container name, "mysql", can be reused:

```
$ docker ps -a
$ docker stop <mysql-container-ID>
$ docker rm <mysql-container-ID>

#NOTE: to remove all containers:
$ docker rm $(docker ps -a)
```

Shell into the mysql container, run mysql -p and create a simple "us_states" database:

```
$ docker exec -it <mysql-container-ID> bash
bash# mysql -p
mysql> show databases;
+---------------------+
| Database            |
+---------------------+
| information_schema  |
| #mysql50#lost+found |
| mysql               |
| performance_schema  |
+---------------------+
4 rows in set (0.12 sec)

# create a simple database for us states:
mysql> CREATE DATABASE us_states;
Query OK, 1 row affected (0.03 sec)

mysql> USE us_states;
Database changed

mysql> CREATE TABLE states (id INT NOT NULL PRIMARY KEY AUTO_INCREMENT, state CHAR(25), population INT(9));
Query OK, 0 rows affected (1.93 sec)

mysql>  INSERT INTO states (id, state, population) VALUES (NULL, 'Alabama', '4822023');
Query OK, 1 row affected (0.17 sec)

mysql> SELECT * FROM states;
+----+---------+------------+
| id | state   | population |
+----+---------+------------+
|  1 | Alabama |    4822023 |
+----+---------+------------+
1 row in set (0.00 sec)

mysql> quit
bash# exit
```

Note: *-p* causes mysql to prompt for the user's password, even if the password has been provided via the MYSQL_ROOT_PASSWORD env variable. In some cases, as seen here, *-p* is required for the mysql command to succeed. Omitting *-p* (even when the root password is specified in the environment), in my case, causes the error below:

```
#On the host running the mysql container we see the password is provided:
$ docker inspect 9501299ac215 | grep PASSWORD
            "MYSQL_ROOT_PASSWORD=foopass",
            
#inside the mysql container we get an error unless we use -p and supply the password:
root@mysql:/# mysql  #no -p
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: NO)
```

Delete the mysql container so we can create it from a pod later:

```
$ docker stop <mysql-container-ID>
$ docker rm <mysql-container-ID>
```

### Security:
When using the ceph-rbd plugin, if the mysql container is not privileged then it will fail with this error:

```
chown: cannot read directory `/var/lib/mysql/': Permission denied
```

The above error is found by looking at the docker logs on the target OSE-node:

```
$ docker ps -a  #need -a since the container start fails
$ docker logs <mysql-id>
```

The "quick" solution that seems to allow the mysql container to start correctly is to set the mysql pod to be privileged, which is done with this yaml fragment:

```
spec:
  containers:
    - image: mysql
     ...
      securityContext:
        capabilities: {}
        privileged: true
```

Note that selinux remains enabled/enforcing.

```
$ getenforce
Enforcing
```


