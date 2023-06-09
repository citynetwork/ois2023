# Some issues


## LoadBalancer type services

<!-- Note -->

In a recent incident, we lost access to our learning platforms. It was because the region in our cloud infra where we host our learning platform primary site had a meltdown and we had a temporary loss of control and compute nodes. This incident led to the unavailability of Kubernetes nodes and the loss of all load balancers and the Kubernetes API service, which are load-balanced behind Octavia.

So we lost accessibility to services that expose our Open edX platforms which are load-balanced behind Octavia, and also the Kubernetes API service. As a result, we can't execute any commands against the cluster. Therefore we can't initiate failover to our disaster recovery site as it requires some scripts to run against the cluster.

This incident highlighted flaws in our DR policy and our failover process. Our systems will only work once Octavia is fixed. And Octavia always gets fixed last, and there's no way around that: for Octavia to work, Nova and Neutron must work, and so that means after a complete site meltdown, you must always get controllers up, and then Nova and Neutron, and only then can you look into fixing Octavia and broken load balancers. 

Even after these issues are solved and we can communicate with the Octavia API, the load balancers are often stuck in an ERROR state, which we can't fix on our own.
We have to wait for someone with admin privileges to fix it, which adds up to our downtime.

To mitigate the downtime during the outage, we reconsidered our DR policy and looked for an alternative solution to expose our Open edX platform services other than the load balancer. For background, We use Caddy as a web server as Tutor uses it as a web proxy and for generating SSL/TLS certificates at runtime.


### Solutions?

`ClusterIP`

`NodePort`

`ClusterIP` with `externalIPs`

<!-- Note -->

We were looking for a solution to expose the caddy service besides the load balancer. As per the Kubernetes documentation, there are many ways to expose services other than the Loadbalancer type.

* The first approach, ClusterIP, allows the service to be accessed only via an internal IP within the cluster. However, this type of exposure is not suitable for a production environment as it restricts accessibility within the cluster.

* The second option, NodePort, is also unsuitable for our needs since our Caddy service operates on ports 80 and 443, while the NodePort range is from 30000 to 32767.

* There is a way not widely used among the Kubernetes community because many use cloud providers' Load balancers which is using ClusterIP with external IP
Customers can expose their Kubernetes cluster to external traffic using ClusterIP and an external IP address. This solution requires that the customer maps the external IP address of a service to their private IP addresses in the cluster.
Unfortunately, we encountered issues in utilizing a Floating IP address as an external IP address. It was because our Floating IP address was getting mapped to a private address within a different subnet than the one used by the ClusterIP.

We could not figure out any way to expose our services without Loadbalancer. But we settled with an approach where rather than doing anything with the Kubernetes APIs, we would operate on a single DNS record that we could flip, without using any API calls, to the backup site. That allows us to have a rudimentary service on the backup site until load balancer functionality in the failed primary site becomes available again.


## Orphaned Pods

<!-- Note -->

What are Orphaned pods in a Kubernetes cluster?

Orphaned pods can occur in Kubernetes when their owner objects, such as a deployment, replica set, or replication controller, is deleted or modified.
By default, Kubernetes deletes these dependent objects. Therefore, the responsibility of cleaning up these objects lies with the kubelet.

What is kubelet?

The kubelet is the primary "node agent" that runs on each node.
Kubelet is responsible for applying, creating, updating, and destroying containers on a Kubernetes node. It performs garbage collection on unused images and unused containers on the nodes.

When kubelet fails to delete these dependent objects, orphaned pods remain dangling in the environment and interfering with the deployment process.

We are facing this issue while performing a rolling update in our deployment.

What happens in a rolling deployment?
During a rolling deployment, the controller deletes and recreates pods with the updated configuration, one by one, without causing downtime to the cluster.

However, our rolling deployment gets stuck when the pod with the mounted persistent volumes, running the old version remains in the Terminating state, and the new version that needs to be recreated gets stuck in the ContainerCreating state. This scenario is encountered with ReadWriteOnce access mode for Persistent Volumes. If we could use ReadWriteMany access mode,  we did not face this issue.

When we check kubelet logs, we notice massive log entries stating "Orphaned pod found <pod UID>, but volumes are still present on the disk." for the pods with mounted Persistent Volumes So we dig deeper into the logs and see when the pod is getting deleted, it tries to unmount the CSI volumes. The unmount operation unmounts the volume, but fails to remove stale volume paths from the node.

This problem of pods getting stuck in the "Terminating" state due to the inability to clean volume subPath mounts is a known issue in Kubernetes. Several users have reported similar issues and there is no definite cause what arises the issue.

The workaround suggested in the bug reports was to manually delete the orphaned pod directory and restart the kubelet service, which works for us also. But we have a CI driven deployment and it's very complicated to manually shell into nodes and deleting stale directories when the pipeline is waiting on terminating (orphaned) pods, which will be stuck forever if stale directories are not manually deleted.


### Solutions?

**Maybe** fixed in a recent Kubernetes release?

(We don't know for sure.)

<!-- Note -->
Ideally, kubelet should intelligently deal with the orphaned pods, cleaning a stale directory manually should not be required.

As per the bug reports the orphaned pods issue is present in Kubernetes versions up to atleast 1.24. It is *possibly* fixed in later releases, which brings us to our next issue.


## Kubernetes versions

<!-- Note -->
One other issue is that Kubernetes releases available in Magnum frequently are quite bit behind upstream. For example, the current most recent Kubernetes version supported in Xena is 1.21.

Even after we upgrade from OpenStack Xena to Openstack Antelope, we get official support for Kubernetes version v1.24 — but this probably won't fix our Orphaned pod issue.

So with Openstack Magnum we are running the limitation of running older Kubernetes versions, or even versions that are beyond their end-of-life.
