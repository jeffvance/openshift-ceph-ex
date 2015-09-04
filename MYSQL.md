## Setting Up and Validating Containerized MySQL:

First, OSE's Security Context Constraints (SSC) need to be defined such that the *seLinuxContext* and *runAsUser* values are set to "RunAsAny". Selinux is still enabled/enforcing on the master and worker-node hosts.

After logging-in to OSE as the *admin* user, edit the two scc files, as shown below:
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

Our mysql example uses the official mysql image found [here](https://hub.docker.com/_/mysql/). First, pull down the image on each of your OSE worker nodes:
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

Delete the mysql container so we can create it from a pod or re-run docker to create it later:
```
$ docker rm <mysql-container-ID>
```
