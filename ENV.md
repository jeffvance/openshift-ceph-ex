## OSE Environment:

* 2 RHEL-7 VMs for running the openshift enterprise (OSE) master and node hosts
* 1 Fedora 21 VM for running the ceph all-in-one (AIO) container

The RHEL-7 hosts running the OSE master and OSE nodes should have the following services enabled and running:
* selinux (*setenforce 1*)
* iptables (*systemctl start iptables*)
* firewalld (*systemctl start firewalld*) Note, if you cannot start firewalld due to the service being masked, you can do a *systemctl unmask firewalld* and then restart it.

The ceph host is not protected in these examples (ie. selinux is running in permissive mode and the firewall is disabled); however, the point here is not how to setup a secure ceph environment, but rather how to run mysql from OSE using an existing (and presumed secure) ceph cluster.

### Other Installations:
1. [OSE installation](OSE.md)
2. [Ceph installation](CEPH.md)
3. [MySQL installation](MYSQL.md)
