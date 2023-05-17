# Our stack today


## OpenStack

<!-- Note -->


## Magnum

<!-- Note -->

In order to transition to the Docker based deployment model with Kubernetes, we required a container orchestration service. During that period, we had two options forcontainer orchestration services (Rancher and Magnum) within our cloud infrastructure. We ultimately chose OpenStack Magnum because it provided all the necessary components for a production-ready setup for deploying Tutor.


### Private registry

<!-- Note -->

We needed a container orchestration service that could accommodate custom-built Docker images, and thus we decided to utilize the private Docker registry provided by Magnum. The built-in Docker registry runs locally on each worker nodes localhost and does not require authentication, making it easy to publish and pull images without any configuration changes to Tutor.

The Docker registry supported by Magnum utilizes Swift for storage and does not maintain any local state. This allows us to launch it as needed in any CI environment,as long as we can provide the necessary Swift credentials.


## Kubernetes

<!-- Note -->

We deploy our kubernetes cluster using Magnum cluster templates.
In our production environment, we have 6 node kubernetes cluster 3 control plane nodes and 3 worker nodes.

We deploy our cluster with kubernetes version as v.1.21.10 on Fedora CoreO 34 with Cinder csi driver enabled so that we can provision OpenStack cinder volumes dynamically. Openstack cinder is a block storage service that allows you to create persistent volumes for your kubernetes cluster so when you enable cinder csi driver on your magnum cluster you will be able to create persistent volumes backed by cinder block storage.


## Ceph

<!-- Note -->

To achieve automatic replication between the primary and DR site, we use Backup and Restore policy using the Ceph object gateway S3 API. 

In our kubernetes environment, the backup and restore process is implemented using cron jobs. The cron job is responsible for capturing MySQL and MongoDB dumps from the primary site, uploading them to S3 buckets, and subsequently restoring them from these S3 buckets at the DR site.  

So, in case of any incident, we can do a failover to the DR site for uniterruped services.


## Zuul

<!-- Note -->

Company-wide, we adopted Gerrit as a code review tool and Zuul for CI/CD to align with OpenStack upstream. So we migrated from Gitlab talking to OpenStack APIs to Gerrit/Zuul talking to the Kubernetes API.

We make all configuration and setup changes to an Open edX environment directly from our Gerrit/Zuul review toolchain and don't do any manual changes to the Kubernetes environment.

So, we have an end-to-end Tutor deployment, to a Magnum-managed Kubernetes cluster, from Zuul.

Let me explain our workflow in more detail:

* We run multiple platforms, so we have different topic branches for every platform that uses a separate Kubernetes namespace.
  That's how we isolate the platforms from each other.
* When making changes to a platform, a Zuul job is triggered to build the corresponding Docker images.
* Subsequently, another  Zuul job pushes this image to a Swift backed registry that Magnum makes available to Kubernetes.
* Lastly, a Zuul job that wrappers tutor commands to deploy a Kubernetes cluster.


