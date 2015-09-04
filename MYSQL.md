## Setting Up and Validating Containerized MySQL:

First, OSE's Security Context Constraints (SSC) need to be defined such that the *seLinuxContext* and *runAsUser* values are set to "RunAsAny". Selinux is still enabled/enforcing on the master and worker-node hosts.

After logging-in to OSE and the *admin* user, edit the two scc files, as shown below:
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
