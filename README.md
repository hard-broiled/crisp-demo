[[_TOC_]]

## Introduction
At a glance, this documentation covers the following areas:
- [Initial Conditions](https://dev.azure.com/jbavacerts/Crisp_Demonstration/_wiki/wikis/Crisp_Demonstration.wiki?wikiVersion=GBwikiMaster&anchor=initial-conditions) and [Assumptions](https://dev.azure.com/jbavacerts/Crisp_Demonstration/_wiki/wikis/Crisp_Demonstration.wiki?wikiVersion=GBwikiMaster&anchor=assumptions)
- [IaC](https://dev.azure.com/jbavacerts/Crisp_Demonstration/_wiki/wikis/Crisp_Demonstration.wiki?wikiVersion=GBwikiMaster&anchor=initial-conditions) considerations, implementation details, resources used, etc.
- [Data Processing Application](https://dev.azure.com/jbavacerts/Crisp_Demonstration/_wiki/wikis/Crisp_Demonstration.wiki/1/?wikiVersion=GBwikiMaster&_a=edit&pagePath=/README.MD) considerations
- [CI/CD](https://dev.azure.com/jbavacerts/Crisp_Demonstration/_wiki/wikis/Crisp_Demonstration.wiki/1/?wikiVersion=GBwikiMaster&_a=edit&pagePath=/README.MD) implementation details and considerations

Please utilize either the table of contents or the links provided in the introduction to jump to desired areas, or skip the initial considerations and assumptions as desired.
 


## Goals, Problem Statement
As stated in the provided document, we have the following guidance for the goals and problem statement of this effort: " 
1. Provision Infrastructure: Use infrastructure as code (IaC) to provision the necessary cloud resources.
2. Deploy a Data Processing Application: Deploy a containerized application that processes CSV files
into Parquet format. Note that you do not need to author the application yourself, but feel free to use
this as an opportunity to demonstrate your ability to author application code in any programming
language of your choosing! However, it is also acceptable to use a publicly available application such
as https://github.com/gmirsky/python-duckdb-csv-convert-to-parquet or any other application of your
choice for this assignment.
3. Implement CI/CD Pipeline: Set up a CI/CD pipeline to automate the deployment process.
4. Document Your Work: Provide documentation detailing your approach, the tools used, and how to
deploy and manage the infrastructure and application."

## Assumptions
### Initial Conditions
The initial conditions for this scenario are a fresh Azure ADO organization and project, with a corresponding fresh Azure Free subscription account with no preexisting resources provided. Based on this, the scenario requires a blending between standard Azure Administration, Project Administration, and DevOps or CI/CD engineering responsibilities. While a DevOps engineer can and could carry out actions across all three realms, this isn't an industry recommended approach. These roles should be separated into the standard roles as suggested by the responsibility categories. To best support the responsibilities for DevOps engineering in a more mature scenario, a DevOps engineer should have the insight and know-how to successfully fulfill all three areas of support, while delegating those responsibilities exclusively covered by the Azure Admin and Project Admin roles as is necessary and required by any given mature project environment. 
That being said, after exploring the capabilities provided by the Azure Free subscription, I found that there are several available solution routes available, but the final architecture configuration would not be up to expected best practice standards for security configurations, resiliency, and reliability, specifically when it comes to networking and private endpoint configurations. This is due to the requirement of a self-hosted agent to be able to successfully deploy IaC elements and the application code to a compute solution within a created virtual network. 
### Separation of Azure Administration from CI/CD responsibilities
As mentioned previously, there are certain resources and responsibilities that shouldn't necessarily be owned by a single DevOps team or engineer that are required in this scenario due to the starting point being a 'blank slate'. This condition is motivated by the fact that larger scope organizations benefit from a separation of responsibilities, and dedicated teams to ensure the different areas of responsibility are maintained with best practices being utilized, and the pillars of cloud first, or in this case the Microsoft Well Architected Framework (WAF) pillars of focus. 
A brief description of these areas of responsibility is as follows:
- Azure Administration
  - The first consideration in this realm is the creation/setting up of self-hosted agents to allow for CI/CD actions or deployments of resources via IaC code base elements to/for Azure resources held within a virtual network with no public access granted to those resources to maintain a zero-trust environment.
  - Another standard key element would be the utilization of Environments within the ADO project, and their reflection within the Azure ecosystem for this organization. A suggested design for this would be:
    - Sandbox: Solely relegated for DevOps and IaC testing
    - Dev/Test: Developer test environment to ensure all created applications/functionality performs in the cloud
    - Integration: Initial environment to ensure that all cloud resources are correctly configured to connect with one-another as necessary, and to support initial smoke testing and integration testing
    - User Acceptance Testing: Production like data testing, stress testing, and final non-production environment testing considerations for this environment
    - Production: The live environment for the organization's product to be held. No testing to be done in this environment. Azure Administration and DevOps teams should be equally invested in the monitoring and maintenance of this environment, favoring CI/CD based actions for the creation of new or updating of existing resources. Minimal direct action from Azure Administration is the best route for this environment. 
- Project Administration/Ownership
 - A responsibility that wouldn't necessarily be owned by a DevOps team in this scenario is the creation and management of service principals, and associated service connections. The decision of which role/team owns these actions can vary by organization, but in my experience I've seen the ownership of service principals be delegated away from DevOps engineers or have the responsibility be owned by both DevOps and Administration teams. For the purposes of this exercise, some elements of this responsibility were traversed for proof of concept. 
### Azure Free Tier Features & Functionality
Certain changes have taken place since I last used the resources and functionality supported by the Azure 'Free' tier. Namely, parallel processing for any CI/CD pipeline actions is not built in to new Free subscriptions, and a lower CPU core limitation (four cores). Other limitations that are necessary to consider are the limitations on deploying certain resources with proper security and networking settings within free tier offerings. Given that the Azure agent pools and agents aren't able to connect to resources on secured virtual networks out of the box, and several critical resources aren't able to utilize private endpoint connections or virtual network connectivity unless utilizing premium SKUs (ACR, ACI, AKS, as examples). It should also be noted that while the AKS control plane is included in the free SKU tier, the compute nodes still incur cost. Given that fact, AKS was not opted for in this solution. It should be noted, that AKS would be the most robust compute option for scaling and reliability purposes, and would also interface well with any created virtual network for this application.




## IaC
### Azure Resources
The resource types used are as follows:
- Storage Account (Blob)
  - This serves as the landing point for incoming CSV files, and out-going parquet files that are created by the data processing application. This scenario utilized only one storage account for input and output, but this design could be improved by separating the input/output points into two storage accounts with bespoke management of each as required by the larger data processing environment as a whole.
- Azure Container Registry
  - The ACR is utilized as a private, and directly managed repository for the container that this application will use. The interest of this choice is so that as the application where to evolve in the future, there are no dependencies on externally controlled containers. Additionally, hosting the container privately fits with security best practices and is in the best interest of the project. 
- Azure Container Applications
  - This is a compute option for this application. ACA is a good solution for applications that don't expect high demand, and a corresponding need for very large scaling capabilities. Additionally, ACA avoids much of the overhead complexity present for utilizing AKS, which is ideal for the assumptions made for this project. While this application could certainly exceed the scaling capabilities provided by ACA, we are assuming that to not be the case as this solution is contained with the Free Azure subscription tier. Should those scaling limitations become an issue, then a conversation and plan to shift this solution to AKS would be a smart decision. As AKS or K8s administration can become very complex very quickly, it is considered a best practice to have dedicated personnel for that task first and foremost.
- Azure Virtual Network
  - The networking solution of choice for Azure. All infrastructure would be expected to live within subnets held within a virtual network in order to start this solution off with proper security practices. Private endpoints can be used for encrypted access to each resource, and encryption at rest where available. 
- Azure Private Endpoints
  - PEs will be a key piece to this solution maintaining best security practices. 
- Azure Private DNS Zones, NSGs, (?)
  - These resources are to be considered when connection to storage, compute, or container support resources in a case by case manner. There are some limitations presented during configuration and deployment when using the Free subscription tier, but simplified templates should allow for a successful implementation without incurring additional costs.
- Additional Solution Considerations:
  - Azure Function App (?)
    - Utilizing a function app with a storage account blob creation trigger is a viable solution to be considered in the context of the Free subscription tier. This could also provide an adequate solution in a more production oriented scenario. For the interests of focusing on specifically maintained scaling, the AKS/ACA route would likely be more preferable as the scalability and resiliency of the app could be controlled more finely with that infrastructure in place.
  - Azure Container Instances
    - ACI has a role when AKS is utilized for this solution. If ACI were to be used as the sole compute option, the only scaling involved in that solution would mandate creating a new ACI instance for every additional instance of the app that is required to fulfill the demanded compute requirements. That would be a highly inefficient solution as those instances would have to be spun up and down directly by Azure administrators, or via DevOps pipelines. 
    Within the context of utilizing AKS, ACI is the solution that allows for flexible scaling bursting as it supports the virtual cluster nodes that can be spun up and down automatically via AKS configuration.  
  - Azure Kubernetes Services
    - As stated previously, AKS presents the most robust compute solution for supporting and app like this. The exclusive limitation in this scenario is the mandate to not incur additional costs, and utilize the Free subscription tier. To briefly restate how those hindrances present:
      - Compute core slots are rapidly filled when a self-hosted agent is required for deploying infrastructure and app code to resources located within the provisioned virtual network. This will cause collisions with the compute cores that will be required for nodes created within the AKS cluster. 
      - Most significantly, the only feature included in the AKS offering that doesn't incur additional cost is the AKS control plane, which won't actually support and compute functionality on apps deployed to the cluster. So this solution while being the most robust route would require additional budget and direct support.

### CI/CD Scripts, Pipelines, & other code base elements
#### CI/CD Pipelines
Given the position description encompassing 'Azure', Azure ADO Pipelines are chosen as the preferred mechanism for any build/deploy & CI/CD functionality and support.

In general, the idea is to have a library of ARM templates for all of the necessary Azure resources. These templates would be owned and maintained by a DevOps engineer/team that would be responsible for ensuring the security and viability of the templates doesn't become stale. The provisioning of the corresponding Azure resources would be maintained via CI/CD pipelines versus typical Azure Administration actions. Utilizing maintained ARM templates would allow for the immediate re-deployment of any related resources as needed. On that note, there is a specific consideration that should be carried out for the virtual networks. These items in particular should have hybrid support between the DevOps team, and direct Azure administrators. This is due to a best practice consideration of ensuring that the most vital components of the security of this application have the IaC support of the DevOps team, as well as the direct management and hands on monitoring that would be provided by an Azure administration team.

#### ARM Templates
The ARM templates for the ACR, SA, VNet, Additional Networking elements, and ACA will be maintained in an Azure Repository titled IaC. For this submitted GitHub repo, they are held within an IaC directory.

## Application
The provided example GitHub repo with application code shows three scripts that could be used for converting locally stored CSV files into Parquet files. While the core functionality of these scripts would accomplish the data processing goal, the application itself will need to be improved to meet the file conversion requirements. Namely, we need to connect to a storage account, check for new CSV files, and convert them into Parquet files. It should be noted that for our ACA based solution, we could also create an EventGrid and produce a new storage blob event event trigger for the application to consume, but in order to keep the solution more streamlined, this route wasn't traversed. This is similar to the considerations if the compute was supported via a Function App. That app could utilize a new storage blob trigger to execute functionality that takes in all CSVs and converts them to Parquet files. 

## CI/CD pipelines configuration
### Pipelines to be utilized
There will end up being two purposes for pipelines in this project, IaC support, and application code base deployment. The IaC pipelines will be very straightforward in design as the core task will be the 'AzureResourceManagerTemplateDeployment@3' task. There are several static analysis tools and solutions that could be implemented in a production level project for this application. Namely, SonarQube could be utilized for static testing, as well as offerings from Microsoft, but for the enterprise scenario a solution like SonarQube would be a more stable and secure route. There should also be heavy usage of Azure Policy to ensure deployed or to be deployed resources are compliant with all relevant OOTB policies, as well as any required custom policies that are created. Typically, the administration of Azure Policy can be owned by Azure Administrators, or an Azure Cloud Security team, depending on the organization.
### Configuration separated from IaC code
The bulk of the separation of configuration values from the code base will be handled via library variable groups. Standard variable groups can be used to house anything from SKU configurations and resource names as needed. For secure values, such as the SA connection string, library variable groups that are pulling values from an Azure Key Vault will be utilized. These items combined with proper usage of pipeline variables and parameters will allow for dynamic pipelines that can support changes to the underlying ARM templates as required, or even deploy new instances of the same resource type should the need arise.
### Service Connections and Principal
Two service connections will be required for this solution to have full deployment capability. An Azure Resource Manager Service Connection with Contributor permissions on the resource group. An Azure Container Registry Service Connection with the AcrPush permission on the ACR. Finally a Service Principal for deployments with Contributor permissions on the resource group and AcrPush on the ACR instance.
### Necessary Azure administration required for ADO  set up
To create a well organized solution, and a solution that is ready for enterprise level production support and requirements, proper deployment environments would be a key part of this project. These can be reflected in the Azure subscription via umbrella resource groups to represent each deployment target environment. A singular resource group was used in this example solution, and no project environments were created, but it is a notable part of a solution in order for it to be ready for larger scale mature project scenarios. An additional consideration is the ownership of ensuring the required service connections are created and maintained correctly, which is a responsibility that could fall under Project Management, as well as Azure Administration. There are of course additional considerations for how the ADO organization, teams, identities, and remaining elements are initialized and maintained, but these are outside of the scope of this scenario. In that same vein, any and all engineers or administration team members would need to have proper Entra ID configurations and set up done on their behalf while maintaining least privilege access for all resources and access points they may require for day to day responsibilities.
