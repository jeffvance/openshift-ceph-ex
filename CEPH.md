## Setting up Ceph in a Single Container

Here is an example of how to set up and run ceph-rbd in a single container. Note: this document describes setting up ceph in a simple and insecure manner which is absolutely not appropriate for any type of production environment, but it is an easy way to get ceph up and running.

### Environment:
The enviromnent used for all of the examples in this repo is described [here](ENV.md). A Fedora 21 VM is used to run containerized ceph. It’s important to create an additional disk on your ceph VM in order to map the ceph image (not to be confused with a docker image) to this extra disk device. Create an extra 8GB disk which, when using KVM, shows up as */dev/vdb* (other VMs may name the extra disk */dev/sdb*).

Below we show docker considerations, how to install ceph-common (client libraries), security and authorization suggestions, so that the OSE pod running mysql can do the ceph RBD mount.

### Docker:
Fedora 21 has docker pre-installed but make sure the docker version is 1.6 or 1.7 -- not 1.8. See the [ENV readme](ENV.md) for information on downgrading or upgrading the installed version of docker.

### Ceph:
Now install the ceph-common library:

```
$ yum install -y ceph-common
 
# for now there is a ceph packaging bug where the ceph-rbdnamer
# code is missing from ceph-common, therefore ceph also needs to
# be installed on the node
$ yum install -y ceph
```

### SELinux and Security:
Though the customer will determine ceph access policies, for the purpose of getting mysql persisted on ceph via openshift, we have left the ceph host vm wide open. So, selinux is set to permissive (*setenforce 0*), iptables rules have been flushed (*iptables -F*), and firewalld is stopped (*systemctl stop firewalld*).

If, for example, selinux is set to enforcing (default) then the ceph/demo container will not start: 

```
$ docker run --net=host -v /etc/ceph:/etc/ceph -v /var/lib/ceph:/var/lib/ceph \
     -e MON_IP=192.168.122.133 -e CEPH_NETWORK=192.168.122.0/24 ceph/demo 
creating /tmp/ceph.mon.keyring
importing contents of /etc/ceph/ceph.client.admin.keyring into /tmp/ceph.mon.keyring
mkdir: cannot create directory '/var/lib/ceph/mon/ceph-ceph-f21-deploy': Permission denied
```

It is likely that certain selinux context labels can be set (see the *chcon* and *semanage* commands) but that is not the purpose of this repo, therefore, the ceph vm is wide open. A note about *semanage* on f21:
```
semanage is not part of the installed f21 utilities so it has to be installed independently.
To find the package containing selinux:
   $ yum provides /usr/sbin/semanage
   # yum provides /usr/sbin/semanage
   Loaded plugins: langpacks
   policycoreutils-python-2.3-7.1.fc21.x86_64 : SELinux policy core python
                                           : utilities
   Repo        : fedora
   Matched from:
   Filename    : /usr/sbin/semanage

   policycoreutils-python-2.3-8.fc21.x86_64 : SELinux policy core python utilities
   Repo        : updates
   Matched from:
   Filename    : /usr/sbin/semanage
   ...

So, policycoreutils-python contains semanage:
   $ yum -y install policycoreutils-python

Then semanage can be used to allow contexts to run.
For example, to allow nginx to have access to it's /usr/share/nginx/* files, simply do:
   $ semanage permissive -a httpd_t
```

### Ceph/demo:
After setting selinux to permissive mode (*setenforce 0*) the ceph/demo AIO container can be run. Pull ceph/demo and run all of the ceph processes inside an all-in-one (AIO) Docker container:

```
$ setenforce 0

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
The ceph-rbd storage plugin uses a ceph secret for authorization. This is a simple yaml file residing on the OSE-master host. The encoded secret value is generated on the ceph host, as follows:

```
#on the ceph monitor host (or the AIO ceph container's VM):
$ ceph auth get-key client.admin
AQDva7JVEuVJBBAAc8e1ZBWhqUB9K/zNZdOHoQ==

#copy the above value and paste it in the echo command below:
$ echo "AQDva7JVEuVJBBAAc8e1ZBWhqUB9K/zNZdOHoQ=="| base64
QVFEdmE3SlZFdVZKQkJBQWM4ZTFaQldocVVCOUsvek5aZE9Ib1E9PQo=
# copy the above base64 output
```
Note: if the base64 conversion step is omitted and a pod using the non-base64 ceph secret is created, it will fail with an error like this:

```
...
-0400        8        {kubelet ose2.rhs}                        failedSync        Error syncing pod, skipping: fork/exec /usr/bin/rbd: invalid argument
```

So, the base64 conversion is critical.

See the [OSE setup readme](OSE.md) for details on how to use this required feature.

### Firewall:
Port 6789 needs to be reachable. An iptables rule accomplishes this, or iptables rules can be flushed (*iptables -F*), or firewalld can be stopped (*systemctl stop firewalld*).

On the ceph host, see if port 6789 is reachable (i.e., not blocked by the firewall):

```
# assume gateway ip on the ceph host is 192.168.122.133
$ nc 192.168.122.133 6789 </dev/null
$ echo $?  #must be 0
 
#or
$ telnet 192.168.122.133 6789
# error if refused, else CTRL-C to exit telnet
```

Repeat the above *telnet* or *nc* commands on some (or all) of the OSE-nodes to test that whichever node the mysql container is scheduled on is capable of reaching the ceph AIO's container's host node.

### Additional information:
Additional information can be found in the [ceph documentation](http://ceph.com/docs/master/start/quick-rbd/).
