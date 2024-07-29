# G2S Infrastructure Proposal

DevOps strategy for infrastructure, CI/CD, cloud solutions, operational and monitoring. 
The aims is to establish a robust, scalable, and efficient IT environment tailored for current needs of the company and allowing scalable and maintainable infrastructure.

- [Deployment of Current Game Engine](#deployment-of-current-game-engine)
- [Version Control System](#version-control-system)
  - [Repository Structure](#repository-structure)
  - [Workflow Strategy: Git-Flow](#workflow-strategy-git-flow)
- [Continuous Integration and Continuous Deployment (CI/CD)](#continuous-integration-and-continuous-deployment-cicd)
  - [Pipeline Architecture](#pipeline-architecture)
  - [Infrastructure Pipelines](#infrastructure-pipelines)
  - [GitOps Pipelines](#gitops-pipelines)
  - [CI/CD Runners Comparison](#cicd-runners-comparison)
- [Kubernetes and Container Management](#kubernetes-and-container-management)
  - [Managed Kubernetes (AWS/GCP)](#managed-kubernetes-awsgcp)
  - [Self-Hosted Solutions](#self-hosted-solutions)
  - [Scaling Strategies](#scaling-strategies)
  - [Multi-Zone and Multi-Cloud Deployments](#multi-zone-and-multi-cloud-deployments)
- [Secrets Management](#secrets-management)
  - [Password Managers](#password-managers)
  - [HashiCorp Vault](#hashicorp-vault)
- [Backup and Recovery](#backup-and-recovery)
  - [Multi-provider Backup Strategy](#multi-provider-backup-strategy)
  - [Log Management Tools](#log-management-tools)

## Deployment of current game engine

To deploy the game service which diagram was sent to me.
I will use simple representation with ascii, asumming that we will use GitHub Actions and AWS managed services.

```bash
                                   ┌────────────────┐                                                     
                                   │                │                                                     
                                   │     GitHub     │                                                     
                                   │                │                                                     
                                   │   GH Actions   │                                                     
                                   │                │                                                     
                                   └────────▲───────┘                                                     
                                            │                                                             
                                            │                                                             
                                            │                                                             
                                            │                                                             
┌───────────────────────────────────────────┼────AWS─────────────────────────────────────────────────────┐
│                                           │                                                            │
│                                           │                                                            │
│                                     ┌─────┤Managed EKS─────────────────────────────┐                   │
│    ┌──────────────┐                 │┌────┴────┐           ┌────────────────┐      │   ┌───────────┐   │
│    │   Casiono    │                 ││         │           │                │      │   │           │   │
│    │   Operators  │                 ││ ArgoCD  ├───────────►   BackOffice   ├──────+───►DocumentDB │   │
│    │      LB      ◄──────┐          ││         │           │                ◄──┐   │   │           │   │
│    │              │      │          │└────┬────┘           └────────────────┘  │   │   └───────────┘   │
│    └──────────────┘      │          │     │                                    │   │   ┌───────────┐   │
│                          │          │     │                 ┌─────────────┐    └───┼───┤           │   │
│                          │          │     └─────────────────►             │        │   │   Kafka   │   │
│    ┌──────────────┐      │          │                       │Game Service─┼────────+───►           │   │
│    │   Public     │      └──────────+───────────────────────┤             │        │   └───────────┘   │
│    │   Load       │                 │   ┌────────────────┐  └─┬────┬──────┘        │                   │
│    │   Balancer   │                 │   │                │    │    │               │   ┌───────────┐   │
│    │              ◄─────────────────+───┤  FrontEnd      ◄────┘    │               │   │           │   │
│    └──────────────┘                 │   │                │         └───────────────+───► Postgres  │   │
│                                     │   └────────────────┘                         │   │    RDS    │   │
│                                     └──────────────────────────────────────────────┘   └───────────┘   │
│                                                                                                        │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

The proposed schema use as much as possible managed services, making maintenance really easy and the whole infrastructure in single cloud provider ensuing compatibility between services.

As we discussed, probably we will use Mongo Atlas rather than DocumentDB, still this is just example on how i plan to deploy system as ilustrated in the provided schema.
Here I dont assume more than a single instance nor any backups/logs outside what AWS gives out-of-the-box.


## Version Control System

### Repository Structure
Proposed structure for maintaining repositories on platforms like GitLab, GitHub, or Bitbucket, which supports varied integrations but lacks subgroup functionality (noted for GitHub). Each category serves specific operational or development needs:
```bash
git/
│
├── DevOps/
│   ├── CI-CD-templates/
│   ├── Container-images/
│   │   ├── fe-builder/
│   │   ├── be-builder/
│   │   ├── ansible-base/
│   │   ├── terraform-base/
│   │   └── ...
│   ├── Ansible/
│   │   ├── common/
│   │   ├── logs-rotation/
│   │   ├── logstash/
│   │   ├── backup/
│   │   ├── grafana/
│   │   ├── node-exporter/
│   │   └── ...
│   └── ...
│
├── IT/
│   ├── IAM/
│   |   ├── CloudIAM/
│   |   ├── AD/
│   |   └── Google/
│   ├── VDI/
│   └── .../
|
├── GitOps/
│   ├── Dev/
│   ├── Stg/
│   ├── Tooling/
│   └── Prod/
│
├── FrontEnd/
│   ├── FE1/
│   ├── FE2/
│   └── ...
│
└── BackEnd/
    ├── BE1/
    ├── BE2/
    └── ...

```

### Workflow Strategy: Git-Flow
```bash
master (main)
├── develop/...
│   ├── feature/...
│   └── feature/...
│   └── ...
```

-   **master (main):** A primary branch that always contains production-quality code, protected for direct push.
-   **develop:** In the main developer branch, we merge all new features before testing in staging, merging, and pushing to the master (main).
-   **feature/:** Branches for each new feature, are created from the develop branch and merged back.

This simple Git-Flow strategy will give us control over who and where can push, merge, and deploy.
With MR (PR) rules set for approvals, and change management we can easily manage who should review code, where to deploy it, how to test and revert in case of failed tests (automatic or manual), and still keep the code base clean with clear rules from where we can deploy each version. 

The strategy will help with retention policies for built artefacts, for example, we can keep any artefact from the master (main) for several years, but for anything from develop/feature branches, we can set it to auto-delete after a week.
This will reduce the cost of maintaining storage and keeping all production artefacts for legal/audit purposes.

As for the GitOps strategy, we can gain a clear representation of environments and an easy auditable git history for our SST (single source of truth).

## Continuous Integration and Continuous Deployment (CI/CD)

### Pipeline Architecture

Continuous Integration (CI) and Continuous Deployment (CD) serve as critical components ensuring our code and applications are tested, buildable, and controlled by an automation system that prevents unauthorized and unapproved changes and deployment of code to production. 

Making proper CI/CD with security and change management is a must for almost all businesses operating with finances.

```bash
            ┌────────┐           ┌─────────┐                                                                                   
            │        │           │         │                                                                                   
      ┌─────►  Git   ◄──────┐    │  JIRA   │                                                                                   
      │     │        │      │    │         │                                                                                   
      │     └────┬───┘      │    └────▲────┘                                                                                   
      │          │          │         │                                                                                        
      │          │          │         │                                                                                        
      │          │          │         │                                                                                        
      │     ┌────▼────┐     │         │                                                                                        
      │     │         │     │         │                                                                                        
      │     │ MR (PR) │     └─────────+────────────────┐                                                                       
      │     │         │               │                │                                                                       
      │     └────┬────┘               │                │                                                                       
      │          │                    │                │                                                                       
      │          │                    │                │                                                                       
      │          │                    │                │                                                                       
      │          │                    │                │                                                                       
      │          │                    │                │                                                                       
      │    ┌─────▼────────────────────┼────────────────┼──────────────PIPELINE────────────────────────────────────────────────┐
      │    │                          │                │                                                                      │
      │    │   ┌─────────┐       ┌────┴────┐       ┌───┴─────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    │
      │    │   │  Clone  │       │ Change  │       │  Clone  │    │         │    │         │    │ Static  │    │         │    │
      └────+───┤  Repo   ├───────►Managment├───────►  Repo   ├────► Build   ├────► Test    ├────► Code    ├────► Upload  │    │
           │   │         │       │         │       │         │    │         ├─┬  │         │    │ Analysys│    │         │    │
           │   └─────────┘       └─────────┘       └─────────┘    └────┬──┬─┘ │  └─────┬───┘    └─┬──┬────┘    └────┬────┘    │
           │                                                           │  │   │        │          │  │              │         │
           │                                                           │  │   │        │          │  │              │         │
           └───────────────────────────────────────────────────────────+──+───+────────+──────────+──+──────────────+─────────┘
                                                                       │  │   │        │          │  │              │          
                                                                       │  │   │        │          │  │              │          
                                                                       │  │   │        │          │  │              │          
                                                               ┌───────▼──+───+────────▼──────────+──┘              │          
                                                               │          │   │                   │                 │          
                   ┌─────────────┬─────────────┬───────────┐   │          │   │ ┌─────────────────┘                 │          
                   │             │             │           │   │          │   │ │                                   │          
┌─────────┐   ┌────▼────┐   ┌────▼────┐   ┌────▼────┐    ┌─┴───▼────┐    ┌▼───▼─▼──┐    ┌─────────┐     ┌─────────┐ │          
│  Calls  │   │         │   │  SMTP   │   │         │    │  Alarm   │    │  Nexus  │    │Object   │     │ Registry│ │          
│    /    ◄───┤PagerDuty│   │    /    │   │  Slack  │    │Aggregator│    │    /    ├────►Storage  │     │    /    ◄─┘          
│   SMS   │   │         │   │ Emails  │   │         │    │          │    │  JFrog  │    │S3 / Dav │     │   ECR   │            
└─────────┘   └─────────┘   └─────────┘   └─────────┘    └──────────┘    └─────────┘    └─────────┘     └─────────┘                    
```

This is an example pipeline we should have.
For example, for (code build), it's just generic logic, depending on the pipeline we should not have some elements or add more.

Deployment, Infrastructure, and GitOps pipelines will be more or less the same depending on what we use for IaC (Infrastructure as Code) and GitOps.

#### Infrastructure Pipelines

Infrastructure pipelines handle the deployments of the infrastructure-as-code (IaC) configurations using tools like Terraform, CloudFormation, Ansible, and Salt.

Most of the example pipelines above will be the same, there will be additional steps like cloud cost report, uploading state files, and logic for handling secrets (pulling from Git provider or encrypted storage).

#### GitOps Pipelines

As GitOps pipelines, it all depends on which solution we adopt.
As with everything else, there are several solutions, every one has its pros and cons both in terms of security and maintenance. 

Our best options are:
* Runner with helm -> directly use CI/CD to connect to the k8s cluster and install/update applications with helm
* ArgoCD -> out-of-the-box GitOps solution, great web UI, but do not deploy helm directly
* FluxCD ->  out-of-the-box GitOps solution, works directly with Helm

Only for the first solution do we need our pipeline, still, to add change management and additional approvals, we will need to make a pipeline that merges IaC changes to master and restrict any account to be able to merge.

### CI/CD Runners Comparison

CI/CD runners play a crucial role in automating the processes of building, testing, and deploying code. Most major Git hosting services like GitLab, GitHub, and Bitbucket offer their integrated runners.

Relying exclusively on integrated solutions can and will lead to vendor lock-in, limiting flexibility and long-term costs. 
The alternative is standalone CI/CD like Jenkins and many other open-source solutions.

---

#### GitHub Actions

-   **Pros:**
    -   **Seamless Integration**: Works natively with GitHub repositories, issues, and pull requests.
    -   **Marketplace**: Access to store with predefined templates.
    -   **Free Minutes**: Has free runtime (probably, we will need more).
    -   **Self Hosted Runner**: Option to self-host GitHub Actions Runner, to have full control of code and cost.
-   **Cons:**
    -   **Complex Pricing**: For private repositories, the cost can escalate quickly based on usage.
    -   **Limited by GitHub**: Vendor lock-in.

---

#### GitLab Runners

-   **Pros:**
    -   **Highly Customizable**: Supports a variety of executors such as Docker, Shell, Kubernetes, and more, that allow running jobs in tailored environments.
    -   **Integrated CI/CD**: Built-in CI/CD capabilities that are deeply integrated with GitLab features like merge requests, issue tracking, docker registry, terraform state repo, and status pages.
    -  **Free Minutes**: Has free runtime (probably, we will need more).
    -  **Self Hosted Runner**: Option to self-host GitHub Actions Runner, to have full control of code and cost.
    -   **Scalability**: It can be scaled by adding more runners. Alternatively, we can use Kubernetes for dynamic runner management.
-   **Cons:**
    -   **Resource Intensive**: Self-hosted runners require significant resources and maintenance. 
    -   **Limited by GitLab**: Vendor lock-in.

#### Bitbucket Pipelines

-   **Pros:**
    -   **Simplicity**: Native integration within Bitbucket with all basic functions we need. 
-   **Cons:**
    -   **Less Extensive Marketplace**: Fewer integrations and third-party tools compared to GitHub Actions.
    -  **Limited by GitLab**: Vendor lock-in.

#### Jenkins

-   **Pros:**
    -   **Flexibility and Customization**: Highly customizable with plugins to support virtually any job type, environment, or integration.
    -   **Platform Agnostic**: It can be used with any Git provider, cloud environment, or local setup.
    -   **Strong Community**: Large and active community, extensive documentation, and plugins for everything.
-   **Cons:**
    -   **Complex Setup and Maintenance**: It requires more effort to set up and maintain, especially with complex pipelines and large-scale operations.
    -   **UI/UX**: Has no fancy single-page application with animations and chart flow diagrams.


## Kubernetes and Container Management

Kubernetes (K8s) is now a de facto standard for orchestrating and containerized applications for production and zero-time deployments, it's robust with features like high availability, scalability, and microservice friendly.

Kubernetes can help avoid vendor lock-in at the application orchestration level, however, while Kubernetes standardizes application deployment, the infrastructure components like load balancers and storage remain provider-specific, which is still vendor lock-in and making a migration from one cloud provider to another not so simple task.

### Managed Kubernetes (AWS/GCP)

Managed Kubernetes services like Amazon EKS (Elastic Kubernetes Service) and Google GKE (Google Kubernetes Engine) offer significant benefits and less maintenance (man-hours).

#### Pros
-   **Simplicity and Speed of Deployment**: Both, AWS and the GCP, do the heavy lifting of setting up a Kubernetes cluster and come with integrated control panels and access controls.
-   **High Availability**: Designed for high availability across multiple zones, reducing the risk of a single point of failure.
-   **Automatic Scaling**: Both, AWS and GCP services, support auto-scaling at both the pod and node level, allowing the cluster to automatically adjust based on load, ensuring our applications handle all requests.
- **Managed Updates/Backup**: Managed core updates and full backups with schedule.

#### Cons

-   **Cost**: Managed solutions are more (way more) expensive than self-hosted ones.
-   **Less Control**: Almost no control over the infrastructure.

### Self-Hosted Solutions

Self-hosted solutions seem like an archaic solution nowadays, but that's not quite right. 
Many organizations revert from managed cloud to self-hosted solutions because of the excessive cost or to gain full control of their infrastructure and data.

There are open-source alternatives like k3s, rk1, and OpenShift which offer nothing less than managed cloud solutions and can run in this cloud or anything from a single Raspberry Pi to hundreds of bare-metal servers.

For example, as cost, even on cheap virtual machine providers we can set up a high availability multi-zone cluster (99.99% uptime) for as low as $80 per month, which is more than 10x lower than the AWS/GCP single-zone cluster.
All that for a fraction of the cost and the chance to have a whole cloud provider fail simultaneously in several data centers is so small that it's negligible. 

Having bugs or faulty updates like the recent one with Crowdstrike is sadly a way big issue and it's not limited only to self-hosted solutions.

### Scaling Strategies

Scaling strategies in Kubernetes are one of the core features.
There are two types of scaling in k8s - for effective and cost-effective operations, we should use both.

-   **Horizontal Pod Autoscaler (HPA)**: Automatically scales the number of pods in a deployment or ReplicaSet based on observed CPU/RAM/Network utilization.
-   **Cluster Autoscaler**: Automatically adjusts the number of nodes in the k8s cluster when pods fail to launch or nodes are underutilized and can be safely transferred without the downtime of service.

### Multi-Zone and Multi-Cloud Deployments

For high reliability and global reach, K8s can be deployed across multiple zones and clouds.
As this is not a cost-effective solution, it's the only solution to guarantee our services will be undisrupted even in cases where whole zones have problems or even whole countries where our k8s nodes reside.

Even in cases where the cloud operator has issues, we will have an undisrupted operation with multi-cloud deployments.

-   **Multi-Zone**: Running k8s across multiple data centers or cloud zones can protect against zone-specific failures and provide low-latency access to users in different geographic locations.
-   **Multi-Cloud**: k8s’ API allows for deployment across different cloud providers, making our applications more cloud-agnostic and operational.

In conclusion, while Kubernetes portability and scalable deployment strategies, decisions about self-hosting versus managed services, should be guided by specific business requirements, cost considerations, and technical capabilities.


## Secrets Management

Effective secrets management is a must to maintain the security and integrity of applications, users, and company resources, especially as organizations scale and complexity increases.
It involves securely handling, storing, retrieving, distributing, and updating secrets such as passwords, API keys, tokens, SSH keys, and other data.

### Password Managers

Password managers are essential tools for any development team, offering a secure way to store, manage, and access passwords and other sensitive credentials.

-   **Centralized Security**: Password managers store all credentials in a centralized, encrypted vault, which reduces the risk of exposing passwords with improper share or storing in files like passwords.txt
-   **Strong Password Enforcement**: Global policies for password complexity and reuse.
-   **Shared Access**: Team-based password managers allow credentials to be shared securely among team members without exposing the actual passwords.
-   **Audit**: Most enterprise-grade password managers provide logs and audit trails. 

When selecting a password manager, consider features like multi-factor authentication, ease of integration with other tools, user access controls, and compliance with security standards.

### HashiCorp Vault

HashiCorp Vault is a more sophisticated tool specifically designed for securing, storing, and controlling access to tokens, passwords, certificates, and API keys in a cloud environment.
It's making application-based access to secrets effortless and working almost out-of-the-box, having the ability to use service accounts and offering its RESTful API for runtime secrets retrieval.

-   **Dynamic Secrets**: Vault can generate secrets on-demand for specific applications or services, not storing them long-term, and can be revoked immediately after use, reducing the risk of exposure.
-   **Secure Secret Storage**: Secrets are encrypted both in transit and at rest, ensuring that sensitive data is always protected.
-   **Fine-Grained Access Control**: Vault uses policies to control who can access what secrets.
-   **Audit Logging**: It has audit logs.
-   **Integration with External Systems**: Integration with LDAP, Active Directory, Kubernetes, and cloud platforms.

Incorporating HashiCorp Vault into a security strategy will enhance security by centralizing and automating the management of secrets.


## Backup and Recovery

Backup and recovery strategies are crucial to any organization's data integrity and availability.
Ensuring that data is safely backed up and tested if we can revert from a backup of data loss due to hardware failures, viruses or human error is a must.

History shows us that even in the biggest corporations with virtually unlimited funds for the best possible hardware, errors, and failures happen very often, data is lost, services are interrupted, and months and years of development are lost.

For proper backup strategies, we need to ensure the data is copied on a regular schedule and perform test recoveries to ensure not if, but when errors and fails happen, to restore any service from its most recent backups and minimize the damage done as fast as possible.

### Multi-provider Backup Strategy

A multi-provider backup strategy is preferred as it uses multiple cloud solutions and/or self-hosted solutions to store backups and versioning.
This approach ensures redundancy and mitigates risks associated with relying on a single provider. 

-   **Risk Diversification**: By diversifying storage across different providers, we minimize the risk of a catastrophic loss if one provider suffers a service disruption or data loss.
-   **Geographic Redundancy**: Different providers offer data centers in various geographical locations, which can be strategically selected to enhance disaster recovery time and even comply with legal regulations.
-   **Vendor Lock-in Avoidance**: Prevent dependency on a single cloud.

The **3-2-1 backup rule** is a fundamental concept in computer systems from the dawn of computers.
The rule is pretty simple and it was coined way before cloud providers and solutions like S3 which offers multi-zone hosting and versioning, still depending on a single S3 or minio instance can (and most probably will) lead to data/backup loss or unavailability.

The rule:
-   **3** copies of your data.
-   **2** different storage types.
-   **1** offsite backup.

Almost any cloud provider from the smallest to the big names like AWS and GCP, offers a sort of block/object storage for backups. 
Choosing 2 or 3 solutions is mainly down to connectivity between cloud providers and the cost of the storage.

```bash
┌──────────────────────────────────Infrasctucture and Services────────────┐
│                                                                         │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐   │
│  │      │  │      │  │      │  │ Data │  │      │  │      │  │      │   │
│  │  VM  │  │  VM  │  │  VM  │  │ Base │  │ App  │  │ App  │  │ App  │   │
│  │      │  │      │  │      │  │      │  │      │  │      │  │      │   │
│  └───▲──┘  └───▲──┘  └───▲──┘  └───▲──┘  └───▲──┘  └───▲──┘  └───▲──┘   │
│      │         │         │         │         │         │         │      │
│      │         │         │         │         │         │         │      │
│      │         │      ┌──┴─────────┴────────┐│         │         │      │
│      │         │      │                     ││         │         │      │
│      └─────────┴──────┤  BackUp Controller  ├┴─────────┴─────────┘      │
│                       │                     │                           │
│                       └──────────┬──────────┘                           │
└──────────────────────────────────+──────────────────────────────────────┘
                                   │                                       
                                   │                                       
   ┌─────────────┐            ┌────▼────┐                                  
   │             ◄────────────┤         │                                  
   │   AWS S3    │   ┌────────┤  MinIO  ├────────┐                         
   │             │   │        │         │        │                         
   └─────────────┘   │        └────┬────┘        │                         
                     │             │             │                         
                     │             │             │                         
                ┌────▼───┐    ┌────▼───┐    ┌────▼───┐                     
                │        │    │        │    │        │                     
                │ Node 1 ◄────► Node 2 ◄────► Node 3 │                     
                │        │    │        │    │        │                     
                └────▲───┘    └────────┘    └───▲────┘                     
                     │                          │                          
                     └──────────────────────────┘                          
```

### Log Management Tools

Log management tools are used to aggregate, monitor, and analyze logs.

The popular opinion is that tools like ELK, Splunk, and Loki are just a webUI with search functionality for easier use than the need to SSH to multiple machines and grep files.
As this is partially true, in a modern managed cloud environment, SSH to managed service is impossible and all logs and metrics are sent to other services self-hosted on cloud managed.

Additionally having a tools for log management reduce the needs for the developers/qa to have access to production servers and increasing security as reducing the copies and ssh keys having god-mode like permissions.
