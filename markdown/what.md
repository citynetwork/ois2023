# What's this about? <!-- .element class="hidden" -->
A major release of a platform we run forced us to change our entire stack.

**_Here's what we did._**


## Cleura logo <!-- .element class="hidden" -->
![Cleura logo](images/cleura-logo.svg)

<!-- Note -->
I work in Cleura in the education team, and what we do is that we run the learning platforms. Our platforms are based on Open edX, a free open source learning management system.
Open edX has two releases a year, just like OpenStack, and Open edX releases follow a naming convention of botanical tree names. So when I mention the names "Maple", "Lilac", "Nutmeg" or "Olive",  I am reffering to specific releases of the Open edX platform.


### Open edX Maple
... changed everything.

<!-- Note -->
Starting from the Maple release of Open edX, upstream deprecated its prior method of deploying the platform using Ansible playbooks. Instead, they decided to move towards a containerized deployment approach.


### Heat → Kubernetes

<!-- Note -->
To adapt to this change, we made the switch from OpenStack Heat to Kubernetes on OpenStack Magnum.


### VMs and Ansible → Containers

<!-- Note -->
This shift allowed us to deploy the Open edX platform in containers, and *that* makes things more scalable and flexible for us.


### GitLab CI → Zuul

<!-- Note -->
Alongside the deployment changes, we also had a transition in our CI-driven deployment approach. We moved from Gitlab CI to Zuul.
 
This is a summary of the architecture before and after, which also discusses some of the issues we ran into along the way.


![Open edX logo](images/open-edx-logo-color.svg)
## Open edX <!-- .element class="hidden" -->

<!-- Note -->
As mentioned earlier, our platform is built on Open edX. Now let me provide you with a concise overview of what Open edX is and its fundamental components.

Open edX is an open-source learning management system (LMS) an online course platform that allows individuals and organizations to create, deliver, and manage online educational content. 	


<!-- .slide: data-background-image="images/lms-screenshot.png" data-background-size="contain" -->
### Learning Management System <!-- .element class="hidden" -->

<!-- Note -->
The edX platform consists of three main components. Firstly, the Learning Management System (LMS) serves as the application through which learners access and engage with the course materials. The LMS provides a user-friendly interface for learners to view and interact with the educational content.


<!-- .slide: data-background-image="images/studio-screenshot.png" data-background-size="contain" -->
### Open edX Studio <!-- .element class="hidden" -->

<!-- Note -->
Secondly, the platform includes Open edX Studio, an advanced course creation tool designed for instructors. It is the content management system for creating courses and course libraries in the Open edX platform.


<!-- .slide: data-background-image="images/django-admin-screenshot.png" data-background-size="contain" -->
### Django Admin Panel <!-- .element class="hidden" -->

<!-- Note -->
Lastly, Django Admin panel, which allows the administrators to handle tasks such as magnaging data and setting permissions.


## How we used to run Open edX

<!-- Note -->
When we started building our learning platform on Open edX in 2015, we adopted a deployment strategy that aligned with the community's recommended approach.
We opted to deploy the platform on cloud server instances using Ansible playbooks.


<!-- .slide: data-background-image="images/old-method.svg" data-background-size="contain" -->
## How we used to run Open edX (image) <!-- .element class="hidden" -->

<!-- Note -->
To achieve a cloud-centric and image-driven deployment, we incorporated OpenStack Heat into our workflow and managed our clusters as Heat stacks. We used Gitlab CI to invoke Heat to deploy the platform.


## What we migrated to


## Tutor <!-- .element class="hidden" -->
![Tutor logo](https://overhang.io/static/img/tutor-logo.svg)

<!-- Note -->
Tutor is a community-supported Docker-based distribution that simplifies the deployment, customization, debugging, upgrading, and scaling of Open edX platforms, starting from the Lilac release. It completely replaced the Open edX native method of using Ansible playbooks for Open edX installation since the Maple release.

So why Tutor came into the picture?

As per the community, it was difficult to install Open edX with native installation scripts. For instance, there are no official instructions for upgrading an existing platform to the new release. There is a recommended approach to backup the data, trash the existing server and create a new one. As a consequence, people keep running deprecated releases and are hesitant to upgrade their platforms.


## What Tutor provides <!-- .element class="hidden" -->
Application Isolation

CLI

Plugins

Kubernetes integration

<!-- Note -->
Tutor aims to simplify the installation and upgrade process of Open edX platforms. Here's how Tutor simplifies Open edX deployment:

* Application Isolation: Tutor runs Open edX processes inside Docker containers, providing isolation and encapsulation of the application components.

* Command Line Interface (CLI): Tutor offers a CLI for common administrative tasks, making it easier to manage and control the Open edX platform.

* Plugin System: Tutor uses a plugin-driven system that allows users to extend or customize Open edX without modifying the core codebase.

* Kubernetes integration: Tutor comes with Kubernetes integration and facilitates running the Open edX platform on a Kubernetes cluster. Under the hood, Tutor wraps `kubectl` commands to interact with the cluster.

Tutor provides comprehensive documentation that offers detailed information on how to use the distribution effectively. If you want to explore further, you can refer to the official Tutor documentation for more insights.


## Our goals

<!-- Note -->
In order to adopt the new Tutor deployment method, our aim was to identify a solution that aligned with the following key goals:


## Our goals, enumerated <!-- .element class="hidden" -->
CI-driven deployment

CI-driven configuration and changes <!-- .element class="fragment" -->

Data replication & redundancy <!-- .element class="fragment" -->

Stateful configuration management <!-- .element class="fragment" -->

Zuul/Gerrit integration <!-- .element class="fragment" -->

<!-- Note -->
* CI driven Open edX deployment: Our objective was to achieve a fully automated deployment of the Open edX platform to a Magnum managed Kubernetes cluster, similar to our previous setup, through a CI/CD pipeline.

* Configuration and setup changes via Gerrit/Zuul: All configuration and setup changes to Open edX environment are to be done through our Gerrit/Zuul toolchain.

* Data Replication and redundancy: We wanted to have a deployment model that utilized clusters distributed over two different regions, while ensuring data replication between them, as we had done in our previous setup. This would provide redundancy and failover to a diffrent region for uninterrupted service in case of any incidents.

* Stateful Configuration Management: We sought to maintain the ability to manage our Open edX platform through a stateful configuration engine. In our previous setup, we used Heat stacks, and now we wanted to use Kubernetes deployments as the orchestration tool.

* Zuul/Gerrit Integration: Instead of GitLab CI, we aimed to utilize Zuul as our preferred CI/CD tool. Zuul/Gerrit's integration would provide the necessary capabilities for code review, job orchestration, and maintaining a smooth development and deployment workflow.
