---
summary: Overview of the infrastructure architecture of Project Neo.
tags: 
---

# Cloud-native architecture of Project Neo

<div class="info" markdown="1">

Project Neo documentation is under construction. It's frequently updated and expanded. Leave your feedback and help us build the most useful content.

</div>

Project Neo is cloud-native. This means that the infrastructure of both the development **Platform**, for building and deploying applications, and the independent **Runtime**, for hosting and running the deployed applications, live in the cloud.

## OutSystems cloud platform

In addition to access to **Service Studio**, each customer is granted access to an **OutSystems cloud platform**. This consists of the following:

* Access to the [**Project Neo Portal**](../neo-differences.md#neo-portal).
* Access to multi-tenant development **Platform** services.
* A default Runtime setup of three stages: a **Development** stage, a **Test** stage, and a **Production** stage.
* A set of isolated, encrypted, and scalable databases and data stores for the Platform services data.
* An isolated, encrypted, and scalable relational database for each Runtime stage.
* An **Identity Service** to keep [user identities secure](../manage-users.md).

The following diagram shows the high-level architecture of the OutSystems cloud platform.

![OutSystems cloud platform](images/cloud-architecture-diag.png)

All internal requests between the Platform and Runtime stages are made through NATS, a secure messaging system. All external requests to both the Platform and each of the Runtime stages go through a Web Application Firewall (WAF) and Content Delivery Network (CDN). All internal and external requests are fully encrypted using Transport Layer Security (TLS). See [Cloud-native network architecture and security of Project Neo](networking.md) to learn more.

#### Platform { #platform }

The development **Platform** comprises multiple services, each responsible for specific functions that facilitate the building and deployment of applications. All the Platform services benefit from a resilient microservices design with a REST API web service interface. Developers, DevOps engineers, and architects interact with these services using tools such as Service Studio and the Project Neo Portal. 

The Platform **Load Balancer** handles all requests to the services. 

An example of a service is the Build Service. Triggered by a developer clicking the 1-Click Publish button in Service Studio, the Build Service takes the visual language model developed in Service Studio (.oml file) and turns it into a compiled application to deploy. 

All the Platform services are multi-tenant and benefit from automatic recoveries and continuous upgrades.

The following diagram shows the high-level architecture of the development Platform.

![Architecture of the development Platform](images/cloud-architecture-platform-diag.png "Architecture of the development Platform") 

#### Runtime { #runtime }

In Project Neo, the **Runtime** is independent of the Platform and comprises multiple **stages**, each independent of the other, that serve to host and run the deployed applications. The default Runtime setup is a Development stage, a Test stage, and a Production stage. Staging lets multiple teams deliver independently and in parallel, a foundational part of the **continuous integration** approach to software development.

The Runtime **Load Balancer** handles all requests to the applications.

The following diagram shows the high-level architecture of the Runtime.

![Architecture of the Runtime](images/cloud-architecture-runtime-diag.png "Architecture of the Runtime") 

## Key technologies of the cloud-native infrastructure

The following is an overview of the cloud technologies that Project Neo uses.

### Kubernetes

The core of both the Platform and each of the Runtime stages is the **Kubernetes cluster**. 

Powered by AWS Elastic Kubernetes Service (EKS), the Platform and each of the Runtime stages use a cluster: an isolated, scalable, and self-healing compute capacity. 

#### Platform cluster { #platform-cluster }

To run on a Kubernetes cluster, each Platform service into packaged into a **container**—a lightweight, standalone, executable package of software that includes everything needed to run an application: code, runtime, system tools, system libraries, and settings. 

##### Auto scaling

The compute capacity for each running Platform service is scalable, which means multiple developers can use the Build Service or any other service concurrently without any performance degradation of the Platform. This lets multiple teams rapidly scale the development process independently of the deployed applications.

The compute capacity is adjusted in real-time, with no user interaction required.

The following diagram shows how auto scaling works inside the Platform cluster.

![Autoscaling of the development Platform](images/cloud-architecture-platform-k8s-diag.png "Autoscaling of the development Platform") 

The **auto scale controller** monitors the CPU and RAM metrics of each running service. It continuously checks these metrics against the cluster compute capacity allocated to each service and can:

* Replicate the running service to optimize the use of the allocated compute capacity.
* Allocate additional cluster compute capacity to the running service if the CPU and RAM metrics for the service exceed a threshold.

Because the overall compute capacity for the isolated Platform cluster is resourced from a multi-tenant pool, it's scalable.

#### Runtime cluster

In the example of the Build Service in the [previous section](#platform), the compiled application generated is a **container image**. An instance of a container image is a container.

Each application is packaged into a separate container, making the infrastructure resilient to individual resource-intensive application(s) that degrade the performance of other applications.

Application containers running in each cluster of each of the Runtime stages are replicated across multiple availability zones (AZs) to ensure **high availability (HA)**.

##### Auto scaling

The compute capacity for each application container running in each Runtime stage is scalable. This lets each of your applications scale independently.

The compute capacity is adjusted in real-time, with no user interaction required.

The following diagram shows how auto scaling works inside the Runtime cluster.

![Autoscaling of the runtime apps](images/cloud-architecture-runtime-scale-diag.png "Autoscaling of the runtime apps") 

The **auto scale controller** monitors the CPU and RAM metrics of each application container. It continuously checks these metrics against the cluster compute capacity allocated to each application container and can: 

* Replicate the application container to optimize the use of the allocated compute capacity and distribution across AZs.
* Allocate additional cluster compute capacity to the application container if the CPU and RAM metrics for the application container exceed a threshold.

The overall compute capacity for the isolated Runtime stage cluster is scalable because it's resourced from a multi-tenant pool.

### Databases and data stores

#### Platform data

Each Platform service make calls to the databases and data stores.

The following table lists and describes the Platform databases and data stores.

| Data stored | Service used | Service description | Availability |
| - | - | - | - |
| Application revisions and dependency information. | Amazon Aurora | A PostgreSQL-compatible relational database built for the cloud. | High availability and high data durability by default (Aurora Serverless). |
| Current and historic application revisions, in the form of .oml files, stored as blob data. | S3 | An object storage service offering industry-leading scalability, data availability, security, and performance. | HA by default. |
| Configuration and metadata from the Platform Build Service. | DynamoDB | A fully managed, serverless, key-value NoSQL database designed to run high-performance applications at any scale. | HA by default. |
| Current and historic application container images. | Elastic Container Registry (ECR) | A fully-managed Docker container registry that makes it easy to store, share, and deploy container images. | HA by default. |

#### Runtime data

Each Runtime stage has an isolated Amazon Aurora database that scales for both compute and storage and has HA through instance replication across multiple AZs. High data durability is ensured through data replication across multiple AZs.

The following diagram shows how this is achieved.

![Database autoscaling](images/cloud-architecture-db-scale-diag.png "Database autoscaling") 

With the Amazon Aurora database architecture, compute and storage is decoupled.

Cluster storage volumes automatically scale as the amount of data stored increases.

#### Platform to Runtime

In addition to storing the application container image in the Elastic Container Registry (ECR), the Build Service passes it to the specified Runtime stage for deployment.

The idea of "Build once, deploy anywhere"—the build process not making strong assumptions about the environment the application is to be deployed into—is a foundational part of the **continuous delivery** approach to software development.

## Logging, monitoring, and analytics

Logs and metrics are collected from each of the application containers running in each Runtime stage cluster. Logs can be filtered on the Project Neo Portal.

Automatic monitoring by EKS replaces unhealthy application containers running in each Runtime stage cluster with a replica.

Site Reliability Engineering on the Platform is supported by automated monitoring.