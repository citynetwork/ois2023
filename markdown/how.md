# Our stack today


## OpenStack <!-- .element class="hidden" -->
![OpenStack logo](images/OpenStack-Logo-Vertical.svg)

<!-- Note -->

Our cloud infrastructure runs on OpenStack, and we use it for both the infrastructure of the learning platform and our lab environments.

In our previous setup, we used the Open edX native installation method, which was deploying Open edX platform onto cloud VMs or baremetal servers using Ansible roles or playbooks (It was called "native" because it deployed directly onto cloud VMs or baremetal servers, rather than into Vagrant-managed VMs). We used Openstack Heat and Ansible to deploy our learning platforms to OpenStack VMs. After transitioning to Docker-based deployment, i.e., containerizing the services and running them on Kubernetes, we use OpenStack Magnum to deploy our Kubernetes cluster.

Additionally, we run a self-paced, interactive learning platform, meaning we provide the learners with access to complex, realistic distributed environments on demand where they can learn things by actually doing it and OpenStack Heat comes in handy.

While both the deployment of labs and the deployment of the Open edX infrastructure involve the use of Heat, their approaches are completely different.
Although we have replaced the Heat-driven approach to deploy the platform with one that interacts with Kubernetes, we have not replaced the Heat-driven approach to deploy our interactive labs.


## Magnum <!-- .element class="hidden" -->
![OpenStack Magnum logo](images/magnum-logo.svg)

<!-- Note -->

In order to transition to the Docker-based deployment model with Kubernetes, we required a container orchestration service and a container registry offering for storing private container images.
Within our cloud infrastructure, we had two options available for container orchestration services: Rancher and Magnum. We chose OpenStack Magnum because it provided all the necessary components for a production-ready setup for deploying Tutor.


### Private registry
<!-- Note -->

We needed a container orchestration service that could accommodate custom-built Docker images, and thus we decided to utilize the private Docker registry provided by Magnum. The built-in Docker registry runs locally on each Kubernetes's worker nodes localhost.

The Docker registry supported by Magnum is just the official Docker registry image with the OpenStack Swift storage driver. When invoked by Magnum, it is Magnum that automatically populates the registry's storage configuration with the necessary parameters to match the Swift endpoint in the region it's running in. In our CI, we duplicate this for a locally-running registry instance, backed by the same storage parameters, that we push our images to. Once we do, any thus-registered image becomes available to our Magnum registries.

In our case it's not native OpenStack Swift that backs the registry, but Ceph radosgw with Keystone integration.

We don't just rely on a single registry, but instead, we have two registries -one for each region. All images are available in both regions, ensuring redundancy and availability across our infrastructure.


## Kubernetes <!-- .element class="hidden" -->
![Kubernetes logo](images/kubernetes-logo.svg)
<!-- Note -->

Magnum lets you create clusters via the OpenStack APIs. To do that, you base your configuration on a cluster template.
The template defines paramters how the cluster will be constructed.

In our production environment, we have six node Kubernetes clusters, three control plane nodes, and three worker nodes.

We deploy our cluster with Cinder CSI driver enabled. So that we can create persistent volumes for our Kubernetes cluster backed by Cinder block storage.

And, we use Octavia (OpenStack load balancing service) to expose our Kubernetes cluster to the outside world via LoadBlancer type service.


## Ceph <!-- .element class="hidden" -->
![Ceph logo](images/ceph-logo.svg)
<!-- Note -->

We use the Ceph object gateway S3 API to achieve automatic replication between the primary and DR site. We don't use the multi-site replication as provided by Ceph natively.

In our Kubernetes environment, the backup and restore process is implemented using cron jobs.

In Kubernetes, you can achieve similar functionality to the cron job utility in Linux by using CronJobs. CronJobs allow you to schedule and run recurring tasks or jobs within your Kubernetes cluster.

With CronJobs, you can define a schedule using a cron-like syntax, specifying the desired frequency and timing for your tasks. Kubernetes will then automatically create and manage pods to execute those tasks based on the defined schedule.
In our environment, the cron job is responsible for capturing MySQL and MongoDB dumps from the primary site, uploading them to S3 buckets, and subsequently restoring them from these S3 buckets at the DR site.  

So, in case of any incident, we can do a failover to the DR site for uniterrupted services.


## Zuul <!-- .element class="hidden" -->
![Gerrit and Zuul logos](images/gerrit-zuul-logo.svg)
<!-- Note -->

Company-wide, we adopted Gerrit as a code review tool and Zuul for CI/CD to align with OpenStack upstream. So we migrated from Gitlab talking to OpenStack APIs to Gerrit/Zuul talking to the Kubernetes API.

We make all configuration and setup changes to an Open edX environment directly from our Gerrit/Zuul review toolchain and don't do any manual changes to the Kubernetes environment.

So, we have an end-to-end Tutor deployment, to a Magnum-managed Kubernetes cluster, from Zuul.


### Zuul branches <!-- .element class="hidden" -->

```plain
main   # (we build container images here)
 |
 +-- platform-A → namespace-A
     + platform-A-site-1
     + platform-A-site-2
 |
 +-- platform-B → namespace-B
     + platform-B-site-1
     + platform-B-site-2
 |
 +-- platform-C → namespace-C
     + platform-C-site-1
     + platform-C-site-2
```

<!-- Note -->
Let me explain our workflow in more detail:

* We run multiple platforms, so we have different topic branches for every platform that uses a separate Kubernetes namespace.
  That's how we isolate the platforms from each other.
* When making changes to a platform, a Zuul job is triggered to build the corresponding Docker images.
* Subsequently, another  Zuul job pushes this image to a Swift backed registry that Magnum makes available to Kubernetes.
* Lastly, a Zuul job that deploys the changes to Kubernetes cluster.
