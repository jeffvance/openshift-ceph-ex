## Setting Up and Validating Containerized MySQL:

First, OSE's Security Context Constraints (SSC) need to be defined such that the *seLinuxContext* and *runAsUser* values are set to "RunAsAny". Selinux is still enabled/enforcing on the master and worker-node hosts, as decribed in the [OSE environment readme](ENV.md).

Our mysql example uses the official mysql image found [here](https://hub.docker.com/_/mysql/). First, pull down the image on *each* of your OSE worker nodes:

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
$ docker stop <mysql-container-ID>
$ docker rm <mysql-container-ID>

#NOTE: to remove all containers:
$ docker rm $(docker ps -a)
```

Shell into the mysql container and run mysql:

```
$ docker exec -it <mysql-container-ID> bash
bash# mysql -p  # -p needed since a root password was supplied above
mysql> show databases;
mysql> quit
bash# exit
```

Delete the mysql container so we can create it from a pod or re-run docker to create it later:

```
$ docker stop <mysql-container-ID>
$ docker rm <mysql-container-ID>
```

### Security:
If the mysql container is not privileged then it will fail with this error:

```
chown: cannot read directory `/var/lib/mysql/': Permission denied
```

The above error is found by looking at the docker logs on the target OSE-node:

```
$ docker ps -a  #need -a since the container start fails
$ docker logs <mysql-id>
```

Even setting the selinxu context for /opt/mysql (which is the host directory being bound to /var/lib/mysql inside the container) on the OSE-node does not fix the issue. Eg. doing this:

```
$ chcon -Rt svirt_sandbox_file_t /opt/mysql
```
still fails with the same permissions error.

The only solution that seems to allow the mysql container to start correctly is to set the mysql pod to be privileged, which is done with the yaml fragment:

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


