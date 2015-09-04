## OSE Environment:

* 2 RHEL-7 VMs for running the openshift enterprise (ose) master and node hosts
* 1 Fedora 21 VM for running the ceph all-in-one (AIO) container

The RHEL-7 hosts running the OSE master and OSE nodes should have the following services enabled and running:
* selinux (*setenforce 1*)
* iptables (*systemctl start iptables*)
* firewalld (*systemctl start firewalld*) Note, if you cannot start firewalld due to the service being masked, you can do a *systemctl unmask firewalld* and then restart it.
