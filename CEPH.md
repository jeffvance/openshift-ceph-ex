## Setting up Ceph in a Single Container

Here is an example of how to set up and run ceph-rbd in a single container. .

### Environment:
The enviromnent used for all of the examples in this repo is described [here](ENV.md).

### Ceph:
A Fedora 21 VM was used to run containerized ceph. It’s important to create an additional disk on your ceph VM in order to map the ceph image (not to be confused with a docker image) to this extra disk device. Create an extra 8GB disk which shows up as */dev/vdb*. Install ceph-common (client libraries) so that the OSE pod running mysql can do the ceph RBD mount .

Fedora 21 has docker pre-installed but make sure the docker version is 1.6+.

```
$ docker --version
Docker version 1.6.0, build 350a636/1.6.0
```

On Fedora 21 there seems to be a docker 1.8 problem with storage setup where the docker-metapool is too small. So, if your docker version is 1.8 then you should consider:

```
$ yum --showduplicates  list | grep ^docker

#if you see docker 1.6 or 1.7 then...
$ yum install -y docker-1.6  #or docker-1.7
```

If docker is lower than 1.6 then:

```
yum install -y docker-1.6  #or docker-1.7
```

Now install the ceph-common library:

```
$ yum install -y ceph-common
 
# for now there is a ceph packaging bug where the ceph-rbdnamer
# code is missing from ceph-common, therefore ceph also needs to
# be installed on the node
$ yum install -y ceph
```

Pull ceph-docker and run all of the ceph processes inside a single Docker container:

```
$ docker pull ceph/demo
#stash the image locally. At some point I will need to use the ceph/daemon image to
#create the monitor, osd, and rgw as separate containers
 
$ docker run --net=host -v /etc/ceph:/etc/ceph -v /var/lib/ceph:/var/lib/ceph \
  -e MON_IP=192.168.122.133 -e CEPH_NETWORK=192.168.122.0/24 ceph/demo
 
$ ps -e | grep ceph
29242 ?        00:00:30 ceph
29282 ?        00:02:48 ceph-mon
29452 ?        00:14:00 ceph-osd
29664 ?        00:03:01 ceph-mds
29725 ?        00:00:00 ceph-rest-api defunct
29993 ?        00:00:00 ceph-msgr
29997 ?        00:00:00 ceph-watch-noti
```

192.168.122.133 is the eth0 IP address for my F21 VM. This container runs in the foreground and logs to the terminal from which it was launched. If you get stuck on this step be sure to rm -rf /etc/ceph/* and potentially rm -rf /var/lib/ceph/* before retrying.

From a different terminal window on the same F21 ceph VM, create an rbd image (not be be confused with container images) and map it to a /dev/rbdX device file:

```
# Note: sometimes the command "modprobe rbd" is required
$ rbd create foo -s 1024 #creates a rbd image named "foo", 1GB in size
$ rbd map foo
 
$ rbd showmapped   #get the rbd dev, here it's /dev/rbd0
id pool image snap device 
0 rbd foo - /dev/rbd0
 
# create a file system on the block dev
$ mkfs.ext4 /dev/rbd0 #same rbdX as above
 
$ rbd --image foo -p rbd info
     rbd image 'foo':
         size 1024 MB in 256 objects
         order 22 (4096 kB objects)
         block_name_prefix: rb.0.100f.74b0dc51
         format: 1
```

### Ceph Secret:
The ceph-rbd storage plugin uses a ceph secret for authorization. This is a short yaml file created as follows, on the ceph monitor host:

```
$ ceph auth get-key client.admin
AQDva7JVEuVJBBAAc8e1ZBWhqUB9K/zNZdOHoQ==

$ echo "AQDva7JVEuVJBBAAc8e1ZBWhqUB9K/zNZdOHoQ=="| base64
QVFEdmE3SlZFdVZKQkJBQWM4ZTFaQldocVVCOUsvek5aZE9Ib1E9PQo=
# copy the above output
```

Create the [ceph-secret file](ceph-secret.yaml):

```
$ oc create -f ceph-secret.yaml 
secrets/ceph-secret
 
$ oc get secret
NAME                  TYPE                                  DATA
ceph-secret           Opaque                                1
...
```
