## How to Setup a Simple OpenShift Enterprise (OSE) Cluster

### Environment:
The enviromnent used for all of the examples in this repo is described [here](../ENV.md).

### Setting up OSE:
OpenShift is installed following the [OpenShift Admin Guide – quick installation](https://docs.openshift.com/enterprise/3.0/admin_guide/install/quick_install.html).

To consider OSE installed and working the OSE master server must be running, all OSE worker-nodes are running, you are able to login to the OpenShift Console via the GUI - https://master-host:8443/console), and you have access to the OSE cli, oc, and can login to the console, eg:

```
$ oc login -u admin
Password:
Using project "default".

$ oc get projects
NAME              DISPLAY NAME   STATUS
default                          Active
openshift                        Active
openshift-infra                  Active
```

### Security:
OSE Security Context Constraints (SCC) are described in [this OSE document](https://docs.openshift.com/enterprise/3.0/architecture/additional_concepts/authorization.html#security-context-constraints). The *privileged* and *restricted* SCCs are added as defaults by OSE and need to be modified in order for mysql and ceph to have sufficient privileges

After logging-in to OSE as the *admin* user, edit the two SCC files, as shown below:

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

### Additional Verification/Validation:

On the OSE master host:
```
$ systemctl status openshift-master -l
openshift-master.service - OpenShift Master
   Loaded: loaded (/usr/lib/systemd/system/openshift-master.service; enabled)
   Active: active (running) since Thu 2015-09-03 16:35:56 EDT; 8h ago
     Docs: https://github.com/openshift/origin
 Main PID: 49702 (openshift)
   CGroup: /system.slice/openshift-master.service
           └─49702 /usr/bin/openshift start master --config=/etc/openshift/master/master-config.yaml --loglevel=4
...
DeploymentConfigs for trigger on ImageStream openshift/mysql
Sep 04 00:46:02 rhel7-ose-1 openshift-master[49702]: I0904 00:46:02.803183   49702 controller.go:38] Detecting changed images for DeploymentConfig default/docker-registry:2
...
Sep 04 00:46:03 rhel7-ose-1 openshift-master[49702]: I0904 00:46:03.001061   49702 controller.go:85] Ignoring DeploymentConfig change for default/docker-registry:2 (latestVersion=2); same as Deployment default/docker-registry-2
```

On an OSE worker-node:
```
$ systemctl status openshift-node -l
openshift-node.service - OpenShift Node
   Loaded: loaded (/usr/lib/systemd/system/openshift-node.service; enabled)
  Drop-In: /usr/lib/systemd/system/openshift-node.service.d
           └─openshift-sdn-ovs.conf
   Active: active (running) since Tue 2015-09-01 18:58:27 EDT; 2 days ago
     Docs: https://github.com/openshift/origin
 Main PID: 94526 (openshift)
   CGroup: /system.slice/openshift-node.service
           └─94526 /usr/bin/openshift start node --config=/etc/openshift/node/node-config.yaml --loglevel=4


Sep 04 00:48:09 rhel7-ose-2 openshift-node[94526]: I0904 00:48:09.198143   94526 manager.go:1388] Pod infra container looks good, keep it "mysql_default"
Sep 04 00:48:09 rhel7-ose-2 openshift-node[94526]: I0904 00:48:09.198195   94526 manager.go:1411] pod "mysql_default" container "mysql" exists as 77f4af567e3dd3b10656ad5ee38a39600174a87c519d6a23735e96cf0ee4208a
Sep 04 00:48:09 rhel7-ose-2 openshift-node[94526]: I0904 00:48:09.198214   94526 prober.go:180] Readiness probe for "mysql_default:mysql" succeeded
Sep 04 00:48:09 rhel7-ose-2 openshift-node[94526]: I0904 00:48:09.198224   94526 manager.go:1442] probe success: "mysql"
Sep 04 00:48:09 rhel7-ose-2 openshift-node[94526]: I0904 00:48:09.198239   94526 manager.go:1515] Got container changes for pod "mysql_default": {StartInfraContainer:false InfraContainerId:dca749fa3530d552a643a836051d4b00ef4e3a69c69ebc8ede059848b3b27569 ContainersToStart:map[] ContainersToKeep:map[dca749fa3530d552a643a836051d4b00ef4e3a69c69ebc8ede059848b3b27569:-1 77f4af567e3dd3b10656ad5ee38a39600174a87c519d6a23735e96cf0ee4208a:0]}
Sep 04 00:48:09 rhel7-ose-2 openshift-node[94526]: I0904 00:48:09.198268   94526 kubelet.go:2245] Generating status for "mysql_default"
...
Sep 04 00:48:09 rhel7-ose-2 openshift-node[94526]: I0904 00:48:09.202366   94526 status_manager.go:129] Ignoring same status for pod "mysql_default", status: {Phase:Running Conditions:[{Type:Ready Status:True}] Message: Reason: HostIP:192.168.122.254 PodIP:10.1.0.41 StartTime:2015-09-03 19:10:07.598461546 -0400 EDT ContainerStatuses:[{Name:mysql State:{Waiting:<nil> Running:0xc213e6d0c0 Terminated:<nil>} LastTerminationState:{Waiting:<nil> Running:<nil> Terminated:<nil>} Ready:true RestartCount:0 Image:mysql ImageID:docker://7eee2d462c8f6ffacfb908cc930559e21778f60afdb2d7e9cf0f3025274d7ea8 ContainerID:docker://77f4af567e3dd3b10656ad5ee38a39600174a87c519d6a23735e96cf0ee4208a}]}
```

And some docker checks on the OSE worker-node:

```
$ docker ps   # make sure docker is running

# if docker is not running then start it:
$ systemctl start docker
$ systemctl status docker -l
```

### Log Files
*journalctl* and *systemctl status* are the main ways to view OSE log files. The *systemctl status* command is shown above. Here are some *journalctl* examples:

```
$ journalctl -xe -u openshift-master  # and on the OSE node:
$ journalctl -xe -u openshift-node
```

It's often necessary to scroll right (right arrow) to see pertinent info.
