## Setting up Ceph in a Single Container

Here is an example of how to set up and run ceph-rbd in a single container. .

### Environment:
The enviromnent used for all of the examples in this repo is described [here](../ENV.md).

### Ceph:
I set up a Fedora 21 VM on which to run ceph. It’s important to create an additional disk on your ceph VM so that you can map a ceph image to this extra disk device. I created an extra 8GB disk which shows up as /dev/vdb I installed ceph-common (client libraries) so that the pod running mysql can do the ceph RBD mount .



