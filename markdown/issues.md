# Some issues


## LoadBalancer type services

<!-- Note -->

Recently, we experienced a temporary loss of control and compute nodes in one of our regions within our Cloud infrastructure. The region had our learning platform's primary(active) site. That incident led to not only the unavailability of Kubernetes nodes but also the loss of all load balancers and also the API service, which are both behind Octavia.

Although we had a disaster recovery setup in another region, we couldn't initiate failover because the Kubernetes API services, load-balanced by Octavia, were inaccessible. As a result, we were unable to access the cluster or execute any commands, hindering the Primary/DR switch that required running scripts against the Kubernetes cluster. This incident exposed our lack of preparedness for such an outage, highlighting flaws in our failover process. None of our systems will function until Octavia is fixed.

Unfortunately, Octavia is always addressed last, as its functionality depends on the proper functioning of Nova and Neutron. Therefore, after a complete site failure, the controllers, Nova and Neutron, must be restored before investigating and fixing Octavia and the broken load balancers.

Even after these issues are solved and we can communicate  with the Octavia API, often the loadbalancers are stuck in an ERROR state, which we can't fix it on 
our own and need admin privileges to fix.


### Solutions?

`ClusterIP`

`NodePort`

`LoadBalancer` with `externalIPs`

<!-- Note -->
To mitigate such downtime during outages, we explored an alternative solution to expose our Kubernetes service without relying on Octavia.
We use Caddy as a web server as it is used in Tutor both as a web proxy and for generating SSL/TLS certificates at runtime.

We were looking for a solution to expose the caddy service other than the loadbalancer and make our platform Octavia free. According to the Kubernetes documentation, several ways exist to expose a service in Kubernetes.

* The first approach, ClusterIP, allows the service to be accessed only via an internal IP within the cluster. However, this type of exposure is not suitable for a production environment as it restricts accessibility within the cluster.

* The second option, NodePort, is also unsuitable for our needs since our Caddy service operates on ports 80 and 443, while the NodePort range is from 30000 to 32767.

* The third option is a loadbalancer, which we are trying to eliminate.
* Lastly, there is a way not widely used among the kubernetes community because many are using cloud provider's Load balancers.
Using ClusterIP with external IP

Customers can expose their Kubernetes cluster to external traffic using ClusterIP and an external IP address. This solution requires that the customer maps the external IP address of a service to their private IP addresses in the cluster. 

Unfortunately, we encountered issues in utilizing a Floating IP address as an external IP address. It was due to the fact that our Floating IP address was mapped to a private address within a different subnet than the one used by the ClusterIP.

So, we were not able to figure out any way to expose our services without Loadbalancer. But we settled with an approach where rather than doing anything with the Kubernetes APIs, we would just operate on a single DNS record that we could flip, without using any API calls, to the backup site. That allows us to have at least a rudimentary service on the backup site, until load balancer functionality in the failed primary site becomes available again.


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

However, our rolling deployment gets stuck when the pod running the old version remains in the Terminating state, and the new version that needs to be recreated gets stuck in the ContainerCreating state. This scenario is encountered with ReadWriteOnce access mode for Persistent Volumes. If we could use ReadWriteMany access mode,  we did not face this issue.

When we check kubelet logs, we notice massive log entries stating "Orphaned pod found <pod UID>, but volumes are still present on the disk." So we dig deeper into the logs and see when the pod is getting deleted, it tries to unmount the CSI volumes. The unmount operation unmounts the volume, but fails to remove stale volume paths from the node.

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

In our cloud infrastructure, we currently have OpenStack Xena deployed. This version of OpenStack only supports Kubernetes up to version 1.21.x, and "officially" only supports Fedora CoreOS 31 (although we at least got it to run on Fedora CoreOS 34). As we are experiencing issues with orphaned pods and the solutions available in newer Kubernetes versions can't be deployed and we are using Fedora CoreOS 34 which has reached its EOL,  which means it is no longer supported and may not receive critical updates and security patches. 

If we upgrade from OpenStack Xena to Openstack Antelope also, we get official support for Kubernetes version v1.24 which will not fix the Orphaned pod issue and will also be end of life in few days (2023-07-28).

So with Openstack Magnum we are running the limitation of running older or EOL kubernetes versions.
