# What's this about?

<!-- Note -->

I work in Cleura in the education team, and what we do is that we run the learning platforms. Our platforms are based on Open edX, a free open source learning management system.
Open edX platforms have biannual releases like OpenStack, following a naming convention of botanical tree names. So when I mention the names "Maple", "Lilac" I am reffering to specific releases of the Open edX platform.

Starting from the Maple release of Open edX, the community-supported method of deploying the platform using Ansible playbooks was deprecated. Instead, the upstream decided to transition towards a containerized deployment approach.

To adapt to this change, we made the switch from OpenStack Heat to Kubernetes on OpenStack Magnum. This shift allowed us to embrace containerization for deploying the Open edX platform, leveraging the benefits it provides in terms of scalability, flexibility, and ease of management.

Alongside the deployment changes, we also had a transition in our CI-driven deployment approach. We moved from Gitlab CI to Zuul.
 
This is a summary of the architecture before and after, which also discusses some of the issues we ran into along the way.


## Open edX

<!-- Note -->

As mentioned earlier, our platform is built on Open edX. Now let me provide you with a concise overview of what Open edX is and its fundamental components.

Harvard University and MIT collaborated to establish edX, a non-profit organization with a mission to promote an open-source vision for the future of education. In 2013, edX made its code open source, enabling universities, corporations, NGOs, government organizations, and entrepreneurs to utilize this powerful tool.

The edX platform comprises of three main components. Firstly, the Learning Management System (LMS) serves as the application through which learners access and engage with the course materials. The LMS provides a user-friendly interface for learners to view and interact with the educational content.

Secondly, the platform includes Studio, an advanced course creation tool designed for instructors. Studio empowers instructors to develop and design their courses with ease. This sophisticated tool simplifies the process of creating engaging and interactive learning experiences.

Lastly, Django Admin panel, the control house for adminstration and management tasks. It facilitates platform-level changes such as setting permissions and managing data, providing administrators with the tools needed to effectively manage the platform.

All the components within the edX platform are meticulously designed and offer intuitive interfaces, ensuring that users can navigate and utilize the tools effortlessly.


## How we used to run Open edX

<!-- Note -->

When we started building our learning platform on Open edX in 2015, we adopted a deployment strategy that aligned with the community's recommended approach.
We opted to deploy the platform on cloud server instances using Ansible playbooks.

To achieve a cloud-centric and image-driven deployment, we incorporated OpenStack Heat into our workflow and managed our clusters as Heat stacks. Additionally, we effectively managed the entire system using GitLab CI, leveraging its capabilities to orchestrate the deployment process.


## Tutor

<!-- Note -->

Tutor is a community-supported Docker-based distribution that simplifies the deployment, customization, debugging, upgrading, and scaling of Open edX platforms, starting from the Lilac release. It completely replaced the Open edX native method of using Ansible playbooks for Open edX installation since the Maple release.

So why Tutor came into the picture?

As per the community, it was difficult to install Open edX with native installation scripts. For instance, there are no official instructions for upgrading an existing platform to the new release. There is a recommended approach to backup the data, trash the existing server and create a new one. As a consequence, people keep running deprecated releases and are hesitant to upgrade their platforms.

Tutor aims to simplify the installation and upgrade process of Open edX platforms. Here's how Tutor simplifies Open edX deployment:

* Application Isolation: Tutor runs Open edX processes inside Docker containers, providing isolation and encapsulation of the application components.

* Command Line Interface (CLI): Tutor offers a CLI for common administrative tasks, making it easier to manage and control the Open edX platform.

* Plugin System: Tutor utilizes a plugin-driven system that allows users to extend or customize Open edX without modifying the core codebase.

Tutor provides comprehensive documentation that offers detailed information on how to use the distribution effectively. If you want to explore further, you can refer to the official Tutor documentation for more insights.

## Our goals

<!-- Note -->

In order to adopt the new Tutor deployment method, our aim was to identify a solution that aligned with the following key goals:

* CI driven Open edX deployment: Our objective was to achieve a fully automated deployment of the Open edX platform to a Magnum managed Kubernetes cluster, similar to our previous setup, through a CI/CD pipeline.

* Configuration and setup changes via Gerrit/Zuul: All configuration and setup changes to Open edX environment are to be done through our Gerrit/Zuul toolchain.

* High Availability Clusters across Two Regions with Data Replication: We wanted to have a deployment model that utilized high availability clusters distributed over two different regions, while ensuring data replication between them, as we had done in our previous setup. This would provide redundancy and fault tolerance for uninterrupted service.

* Stateful Configuration Management: We sought to maintain the ability to manage our Open edX platform through a stateful configuration engine. In our previous setup, we utilized Heat stacks, and now we wanted to leverage Kubernetes deployments as the orchestration tool.

* Zuul/Gerrit Integration: Instead of GitLab CI, we aimed to utilize Zuul as our preferred CI/CD tool. Zuul/Gerrit's integration would provide the necessary capabilities for code review, job orchestration, and maintaining a smooth development and deployment workflow.

