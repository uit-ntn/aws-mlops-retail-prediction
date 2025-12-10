---
title: "Announcing Amazon EKS Capabilities for workload orchestration and cloud resource management"
weight: 11
chapter: false
pre: " <b> 3.11. </b> "
---

_Author: Channy Yun (윤석찬) — 30 NOV 2025_

Today, we’re announcing Amazon Elastic Kubernetes Service (Amazon EKS) Capabilities, an extensible set of Kubernetes-native solutions that streamline workload orchestration, Amazon Web Services (AWS) cloud resource management, and Kubernetes resource composition and orchestration. These fully managed, integrated platform capabilities include open source Kubernetes solutions that many customers are using today, such as Argo CD, AWS Controllers for Kubernetes, and Kube Resource Orchestrator.

With EKS Capabilities, you can build and scale Kubernetes applications without managing complex solution infrastructure. Unlike typical in-cluster installations, these capabilities actually run in EKS service-owned accounts that are fully abstracted from customers.

With AWS managing infrastructure scaling, patching, and updates of these cluster capabilities, you can use the enterprise reliability and security without needing to maintain and manage the underlying components.

Here are the capabilities available at launch:

- **Argo CD** – This is a declarative GitOps tool for Kubernetes that provides continuous continuous deployment (CD) capabilities for Kubernetes. It’s broadly adopted, with more than 45% of Kubernetes end-users reporting production or planned production use in the 2024 Cloud Native Computing Foundation (CNCF) Survey.
- **AWS Controllers for Kubernetes (ACK)** – ACK is highly popular with enterprise platform teams in production environments. ACK provides custom resources for Kubernetes that enable the management of AWS Cloud resources directly from within your clusters.
- **Kube Resource Orchestrator (KRO)** – KRO provides a streamlined way to create and manage custom resources in Kubernetes. With KRO, platform teams can create reusable resource bundles that abstract away complexity while remaining natively to the Kubernetes ecosystem.

With these features, you can accelerate and scale your Kubernetes use with fully managed capabilities, using its opinionated but flexible features to build for scale right from the start. It is designed to offer a set of foundational cluster capabilities that layer seamlessly with each other, providing integrated features for continuous deployment, resource orchestration, and composition. You can focus on managing and shipping software without needing to spend time and resources building and managing these foundational platform components.

## How it works

Platform engineers and cluster administrators can set up EKS Capabilities to offload building and managing custom solutions to provide common foundational services, meaning they can focus on more differentiated features that matter to your business.

Your application developers primarily work with EKS Capabilities as they do other Kubernetes features. They do this by applying declarative configuration to create Kubernetes resources using familiar tools, such as kubectl or through automation from git commit to running code.
![B11img](/images/blog11/b1101.png "B11img")

## Get started with EKS Capabilities

To enable EKS Capabilities, you can use the EKS console, AWS Command Line Interface (AWS CLI), eksctl, or other preferred tools. In the EKS console, choose Create capabilities in the Capabilities tab on your existing EKS cluster. EKS Capabilities are AWS resources, and they can be tagged, managed, and deleted.
![B11img](/images/blog11/b1102.png "B11img")
You can select one or more capabilities to work together. I checked all three capabilities: ArgoCD, ACK, and KRO. However, these capabilities are completely independent and you can pick and choose which capabilities you want enabled on your clusters.
![B11img](/images/blog11/b1103.png "B11img")
Now you can configure selected capabilities. You should create AWS Identity and Access Management (AWS IAM) roles to enable EKS to operate these capabilities within your cluster. Please note you cannot modify the capability name, namespace, authentication region, or AWS IAM Identity Center instance after creating the capability. Choose Next and review the settings and enable capabilities.
![B11img](/images/blog11/b1104.png "B11img")
Now you can see and manage created capabilities. Select ArgoCD to update configuration of the capability.
![B11img](/images/blog11/b1105.png "B11img")
You can see details of ArgoCD capability. Choose Edit to change configuration settings or Monitor ArgoCD to show the health status of the capability for the current EKS cluster.
![B11img](/images/blog11/b1106.png "B11img")
Choose Go to Argo UI to visualize and monitor deployment status and application health.
![B11img](/images/blog11/b1107.png "B11img")
To learn more about how to set up and use each capability in detail, visit Getting started with EKS Capabilities in the Amazon EKS User Guide.

## Things to know

Here are key considerations to know about this feature:

- **Permissions** – EKS Capabilities are cluster-scoped administrator resources, and resource permissions are configured through AWS IAM. For some capabilities, there is additional configuration for single sign-on. For example, Argo CD single sign-on configuration is enabled directly in EKS with a direct integration with IAM Identity Center.
- **Upgrades** – EKS automatically updates cluster capabilities you enable and their related dependencies. It automatically analyzes for breaking changes, patches and updates components as needed, and informs you of conflicts or issues through the EKS cluster insights.
- **Adoptions** – ACK provides resource adoption features that enable migration of existing AWS resources into ACK management. ACK also provides read-only resources which can help facilitate a step-wise migration from provisioned resources with Terraform, AWS CloudFormation into EKS Capabilities.

## Now available

Amazon EKS Capabilities are now available in commercial AWS Regions. For Regional availability and future roadmap, visit the AWS Capabilities by Region. There are no upfront commitments or minimum fees, and you only pay for the EKS Capabilities and resources that you use. To learn more, visit the EKS pricing page.

Give it a try in the Amazon EKS console and send feedback to AWS re:Post for EKS or through your usual AWS Support contacts.
