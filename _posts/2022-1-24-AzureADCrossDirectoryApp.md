---
title:  "AzureAD Cross-Directory App"
tags: [azure, azuread]
---

As noted in {{ site.baseurl }}{% post_url 2022-1-23-HelloAgainAzure %}, I've spent the last few years using AWS for nearly everything, 

One of the pieces I "miss" from AWS is AssumeRole - specifically the concept that you can jump around between accounts with ephemeral roles as-needed vs having individualized credentials in each account. Azure appears to really has no equivalent to this functionality, although there are a few pieces you can use to approach some of this.

After digging in a bit, I found Azure supports the ability to have a single identity be usable across directories (and their trusting subscriptions) - one of the major use cases I've used previously with AssumeRole (this could be particularly useful with an infrastructure deployment system like Atlantis where we want to support the ability to deploy changes to resources serviced by multiple directories).

## Overview

AzureAD supports multitenant applications - that is, applications that are defined once, and can then be installed into multiple directories. When the application is installed into directories, a service principal object is created representing that instance of the application in that specific tenant. Microsoft defines this as (<https://github.com/MicrosoftDocs/azure-docs/blob/main/articles/active-directory/develop/developer-glossary.md#application-object>):

> The application object defines the application's identity configuration globally (across all tenants where it has access), providing a template from which its corresponding service principal object(s) are derived for use locally at run-time (in a specific tenant).

As a possible solution for this particular use case, we could define an application that represents the workload we'd like to use across multiple directories, then install that into each directory where we need that workload to be able to do work. This would allow us to use service principal authentication to log into each directory, using credentials defined in our "home" directory.

This scenario is illustrated below, where we have a home "Management" directory with a number of subscriptions, as well as a "child" directory with a number of subscriptions.

![AzureADCrossDirectoryApp]({{ site.url }}/assets/AzureADCrossDirectoryApp_highlevel.png)

1. Register our application in our Home/"Management" directory (at previous employers, we used "Management" accounts to represent a location containing resources meant for use across other managed accounts/directories). We'll set up a Web Redirect to make things simpler for installing into child tenants, as well as generate a secret for our service principal login.
2. If desired, grant permissions for the service principal in our Home/Management directories to use subscriptions trusted by the Home/Management tenant. This is use-case dependent.
3. Install our application into our child directories.
4. If desired, grant permissions for the service principal in our child directories to use subscriptions trusted by the child tenant(s). This is use-case dependent.

## Configuration

For simplicity's sake, I'll show this in the Azure Portal, and I'll assume you're a Global Admin in all tenants. Similar configuration could be done with Terraform, PowerShell, Azure CLI, etc.

### Build the MultiTenant Application in the Home/Management Azure Active Directory

1. Log into the Azure Portal, and ensure you're logged into the Home/"Management" Azure Active Directory where you want the App Registration to live (you can switch the AAD you're logged into with the Gear icon in the Azure nav bar)
2. Navigator to Azure Active Directory, App Registrations
3. Click New Registration
4. Enter a name for your application (ex: MultiTenant App), and select "Accounts in any organizational directory (Any Azure AD directory - Multitenant)" under "Supported account types". I'm also setting a Redirect URI of type "Web" with a value of "https://portal.azure.com" - this will become useful when installing the app in child directories. Click "Register" to build the app registration. Because we're using the portal, this also does a number of other things for us - the app is installed into our directory as an Enterprise Application/Service Principal, and the "User.Read" Microsoft Graph permission has been added to the app for us. If you use another tool, you'll need to add these yourself. ![AppRegistration]({{ site.url }}/assets/AzureADCrossDirectoryApp_AppRegistration.png). When the app is created, take note of the "Application (client) ID" - this will become our username, as well as the tenant ID for our Home/Management Tenant.
5. Under our new app registration, navigate to "Certificates & secrets", "Client secrets", and click "New client secret" to generate a secret for our application. Set a name for our secret and pick an expiration if desired (or accept the defaults), then click "Add" to generate the secret. Make sure you store this somewhere safe - once you navigate away from the page, the secret won't be shown again.
6. (Optional) If you'd like to use this application in your Home/Management directory, you'll need to assign it a subscription. For simplicity's sake, I'll assign "Contributor" access to my subscription (Navigate to the subscription, "Access control (IAM)", "Role assignments", and assign the "Contributor" role to the subscription) ![SubscriptionRole]({{ site.url }}/assets/AzureADCrossDirectoryApp_SubscriptionRoleMGMT.png)

At this point, we should be able to test login using the Azure CLI to our Home/Management directory with our service principal.

1. Validate that we're not logged in currently at the CLI (if you need to log out, you can use `az logout`): ![NoLogin]({{ site.url }}/assets/AzureADCrossDirectoryApp_NoLogin.png)
2. Log in to the Azure CLI with our service principal: `az login --service-principal --username [Application (client) ID from above] --tenant [Tenant ID for our Home/Management tenant]`. You'll be prompted for the password. If everything worked correctly, you'll be logged in successfully. Depending on what permissions you've assigned to the application, you should also be able to perform operations as this service principal. ![LoginMGMT]({{ site.url }}/assets/AzureADCrossDirectoryApp_LoginMGMT.png).

### Install the MultiTenant Application into Child Azure Active Directories

Now that you've validated that your application works as expected in the Home/Management directory, lets install it into another Azure Active Directory.

1. Open `https://login.microsoftonline.com/[Child Tenant ID]/oauth2/authorize?client_id=[Application (client) ID from above]&response_type=code&redirect_uri=https://portal.azure.com` in the browser, substituting in the Tenant ID of the child Azure Active Directory tenant as "Child Tenant ID", as well as the App Registration's "Application (client) ID" as "Application (client) ID". You'll be presented with a Consent prompt to install the application into the target directory. Check the box to consent on behalf of the org, and click "Accept" to install the app into the directory. ![Consent]({{ site.url }}/assets/AzureADCrossDirectoryApp_InstallConsent.png)
2. Once the app is installed, give the application access to a subscription trusted by the Azure Active Directory. For simplicity's sake, I'll assign "Contributor" access to my subscription (Navigate to the subscription, "Access control (IAM)", "Role assignments", and assign the "Contributor" role to the subscription) ![SubscriptionRole]({{ site.url }}/assets/AzureADCrossDirectoryApp_SubscriptionRoleDEV.png)

At this point, we should be able to test login using the Azure CLI to our child directory with our service principal.

1. Validate that we're not logged in currently at the CLI (if you need to log out, you can use `az logout`): ![NoLogin]({{ site.url }}/assets/AzureADCrossDirectoryApp_NoLogin.png)
2. Log in to the Azure CLI with our service principal: `az login --service-principal --username [Application (client) ID from above] --tenant [Tenant ID for our child tenant]`. You'll be prompted for the password. If everything worked correctly, you'll be logged in successfully. Depending on what permissions you've assigned to the application, you should also be able to perform operations as this service principal. ![LoginDEV]({{ site.url }}/assets/AzureADCrossDirectoryApp_LoginDEV.png).

Rinse/Repeat on any additional Directories as needed.

### Wrap-Up

This works, but it's both significantly more complicated than similar functionality in AWS, as well as less flexible.

Ideally, I'd like to see this kind of functionality supported by Azure Managed Identites, but that's currently not possible <https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/managed-identities-faq#can-i-use-a-managed-identity-to-access-a-resource-in-a-different-directorytenant>. Having cross-directory functionality supported by user-assigned managed identities would be a big win.
