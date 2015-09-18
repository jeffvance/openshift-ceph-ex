## OSE Environment:

* 2 RHEL-7 VMs for running the openshift enterprise (OSE) master and node hosts
* 1 Fedora 21 VM for running the ceph all-in-one (AIO) container

The RHEL-7 hosts running the OSE master and OSE nodes should have the following services enabled and running:
* selinux (*setenforce 1*)
* iptables (*systemctl start iptables*)
* firewalld (*systemctl start firewalld*) Note, if you cannot start firewalld due to the service being masked, you can do a *systemctl unmask firewalld* and then restart it
* the OSE worker nodes and the ceph host need to be running docker. Currently docker version 1.8 has storage setup issues, so see below on how to upgrade or downgrade the version of docker on your VMs.

The ceph host is not protected in these examples (ie. selinux is running in permissive mode and the firewall is disabled); however, the point here is not how to setup a secure ceph environment, but rather how to run mysql from OSE using an existing (and presumed secure) ceph cluster.

### Docker:
Fedora 21 has docker pre-installed but make sure the docker version is 1.6 or 1.7 -- not 1.8.

```
$ docker --version
Docker version 1.6.0, build 350a636/1.6.0
```

#### Downgrading Docker:
On Fedora 21 there seems to be a docker 1.8 problem with storage setup where the docker-metapool is too small. So, if your docker version is 1.8 then you should consider downgrading to 1.7 or 1.6.

```
$ yum --showduplicates  list | grep ^docker

#if you see docker 1.6 or 1.7 then...
$ yum install -y docker-1.6  #or docker-1.7
```

If the above docker downgrade fails, reporting "Error: Nothing to do", then first attempt a yum clean:

```
$ yum clean all
# redo yum install from above...
```

If the downgrade still fails then the docker target version rpm can be downloaded directly from docker.com. As of the time of this writing this link worked:

```
https://get.docker.com/rpm/1.7.1/fedora-21/RPMS/x86_64/docker-engine-1.7.1-1.fc21.x86_64.rpm
```

#### Upgrading Docker:
If docker is lower than 1.6 then:

```
yum install -y docker-1.6  #or docker-1.7
```


### Other Installations:
1. [OSE installation](OSE.md)
2. [Ceph installation](CEPH.md)
3. [MySQL installation](MYSQL.md)
