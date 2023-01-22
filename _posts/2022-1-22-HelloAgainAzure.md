---
title:  "Hello Again, Azure"
tags: [azure, azuread, aws]
lastModified: 2022-1-22
---

After spending the last several years building things on AWS (mostly using Kubernetes - and also clearly not writing anything in this blog), I moved to a new spot in September where Azure is the norm. As such, I've been spending time furiously trying to broaden my understanding of the Azure ecosystem and translate many of the concepts with which I'm familiar in AWS into the Azure world. Several years back I used AzureAD at a previous employer, but this is really my first foray into Azure's IAAS/PAAS offerings.

Microsoft has an entire set of training materials focused on the "Azure for AWS professionals" space: <https://learn.microsoft.com/en-us/azure/architecture/aws-professional/>

There are also a number of community docs that have been helpful in navigating AWS vs Azure:
* <https://arsenvlad.medium.com/mapping-aws-iam-concepts-to-similar-ones-in-azure-c31ed7906abb>
* <https://cloud.google.com/free/docs/aws-azure-gcp-service-comparison>

As I continue to dig into Azure, for my own learning I'll be documenting the things that stand out to me as "different" and worth noting coming from an AWS background. This post will likely evolve over time, and this post isn't intended to be all-encompassing - more a running list of things I've found thus far.

### AWS vs Azure - Differences

#### AWS Accounts vs Azure Acccounts/Subscriptions/Directories

AWS is relatively simple here - everything in AWS lives in the context of an AWS account, including identities.

I'm still not sure I 100% understand how everything works in the Azure space. In general, "accounts" can hold multiple subscriptions, and subscriptions have a trust relationship to AzureAD Directories (where identities are contained with Azure) - Directories can trust multiple subscriptions, a subscription can only trust a single Directory. <https://learn.microsoft.com/en-us/azure/architecture/aws-professional/accounts>

Similar to with AWS Organizations, there are ways to group subscriptions with Azure Management Groups <https://learn.microsoft.com/en-us/azure/governance/management-groups/overview>.

#### Resource Groups

In AWS, resources can be grouped into any number of resource groups. Personally I've never leveraged this in AWS, but it's there.

In Azure, resource groups are a mandatory and fundamental aspect of resources in Azure - resources must belong to a management group, and they can only belong to a single group at at time. <https://learn.microsoft.com/en-us/azure/architecture/aws-professional/resources#resource-groups>

#### Centralized AWS IAM vs AzureAD + Azure RBAC

Since AWS stores identities in the account, the IAM service in the account offers a single pane of glass view for identities and permissions to resources in the account.

In Azure, this appears to be done in two separate places - AzureAD for identities, and Azure IAM for RBAC. Identities are stored in AzureAD Directories, which are linked to Subscriptions. The subscriptions contain RBAC roles assignments connecting identities to permissions/role definitions, applied at specific scopes (subscriptions, resource groups, specific resources, etc). This can make tracking down exactly where permissions are applied more difficult since they can be applied at specific scopes - not sure there's a single pane of glass to quickly identify what resources an identity has access to.

AWS SSO/IAM Identity Center is very similar to AzureAD in that it now offers a centralized identity store.

#### AzureAD Applications vs IAM Users

In AWS, IAM users can be tied to a person or application (although generally both are discouraged in favor of ephemeral creds via AWS roles/IRSA). Users can have username/passwords for the GUI, or access key and secret id for CLI/API operations.

In Azure, AzureAD users are used by humans.

I'm still trying to wrap my head around the AzureAD Applications piece fully, but in general, AzureAD application registrations can be created which create "cookie cutter" or "templates" for applications. As this application is installed into AzureAD Directories, an Enterprise Application/Service Principal is created, which represents the actual instantiation of that application in that tenant. Applications can be single- or multi-tenant.

The service principal piece actually extends even further with concepts like Managed Identity (both system and user) - I believe these create service principals as well.

Like iam users with access key and secret id, service principals can have a secret or certificate to allow for authentication.

#### AWS IAM Roles (AssumeRole)

AWS IAM roles are a fundamental aspect of working with AWS, whether that be interactive work at the CLI, in the browser, or via applications like Atlantis. In general, there is no direct equivalent in Azure.

Azure System and User Managed Identities are a close approximation for services, but cannot be used by users.

There appears to be nothing approaching the temporary role-hopping functionality in AssumeRole for either users or services in Azure.

#### VNET Default Outbound Connectivity in Azure

In AWS, machines in private subnets need a NAT gateway to get outbound connectivity. NAT gateways are the subject of much controversy in the AWS world.

Azure provides Default Outbound Access <https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/default-outbound-access>, which routes outbound connectivity through addresses owned by Azure. These addresses are subject to change, cannot be predicted, and the use is considered "Insecure by default" by Azure <https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/plan-for-inbound-and-outbound-internet-connectivity>. Azure supports attaching a NAT gateway to VNETs and recommends this for scalability and security reasons, but it's not necessary for outbound connectivity to function.

#### Lack of OIDC Support for Identity in Kubernetes (Preview)

IAM Roles for Service Accounts (IRSA) has been available with EKS and AWS since 2019, and is a standard mechanism of providing OIDC-based access to kubernetes based workloads in AWS without having to provide long-lived credentials.

As of 1/2023, Azure has announced Azure AD Workload Identity <https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview>, providing OIDC-based auth to workloads in AKS.

One wrinkle I've noticed in testing this so far is that while IRSA has you associate the Role ARN with the k8s service account (which is generally set by the user as opposed to system-generated), Azure has you associate the client id of the workload identity with the service account, which is generate by Azure and is a GUID. This may make it difficult to destroy environments and rebuild them for testing purposes without having to swap out the GUIDs (which isnt necessary in AWS).

#### Wildcards are not Supported in External Identity Provider Configurations

AWS supports wildcards in external identity provider configurations (OIDC, etc).

Azure does not <https://learn.microsoft.com/en-us/azure/active-directory/develop/workload-identity-federation-create-trust>. Additionally, a maximum of 20 federated identity credentials can be added to an application or user-assigned managed identity.This will likely make it difficult to config OIDC trusts with other tools unless those tools support customizing the exchanged fields (for example, subject).

#### Login Context is Not Synced between Powershell and Azure CLI

This one is just maddening and should be table stakes for a public cloud nowadays - it causes unnecessary confusion <https://github.com/Azure/azure-cli/issues/16460>.

#### No Equivalent of AWS Certificate Manager (ACM) in Azure

AWS has AWS Certificate Manager, which allows for the creation of certificates for use within the AWS ecosystem (for things like AWS load balancers, etc).

Azure has "App Service Certificate Manager", as well as "Azure Front Door managed certificate", but doesn't appear to have any central certificate issuing capability. Because of this, functionality is limited - for example, Azure's regional HTTP load balancing solution "App Gateway" doesn't appear to have any cert issuing capability nor the ability to use certs from App Service Cert Manager.

There is at least one open source solution that attempts to address this in Azure, [Key Vault Acmebot](https://github.com/shibayan/keyvault-acmebot).

#### Day 2 Niceties in Azure

From what I've seen so far, Azure seems to have a lot of "Day 2 niceties" that I haven't seen in AWS. I've often seen AWS described as a bunch of lego blocks - you can assemble them however you'd like, but you're responsible for them, connecting them together, and running them over time. Azure seems to have more tools around managing these building blocks "at scale".

Things like Azure Storage Firewall to restrict access to storage accounts (this would be possible with S3 bucket policies), Azure Kubernetes Fleet Manager, etc.

### Wrap-Up

I've found it really helpful to have a solid base in another public cloud with which to do comparisons against Azure, and I'm really enjoying learning new things and solving additional puzzles. Onward!
