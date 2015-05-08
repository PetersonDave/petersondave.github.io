---
layout: post
title:  "Alternate Sitecore 8 Azure Deployment Options"
date:   2015-05-08 11:20:00
categories: Sitecore Azure
comments: true
---

We all know the Sitecore Azure module for deploying Sitecore instances to the cloud. At the time of this post, Azure module support for Sitecore 8 has yet to be released. For those of us who want to move to a PaaS offering with Sitecore 8, we're left searching for alternative methods of deployment. 

This post covers the two other recommended approaches of deploying Sitecore 8 to Azure. Deployment through Visual Studio or Microsoft Azure PowerShell.

#Option 1: Visual Studio

Deployments to Azure from Visual Studio can be executed both manually and through an automated process. <a href="https://kb.sitecore.net/en/Articles/2014/07/30/16/07/983166.aspx" target="_blank">Following steps to configure your solution</a>, you must first <a href="https://msdn.microsoft.com/en-us/library/azure/hh420322.aspx" target="_blank">convert your web project</a> to a Microsoft Azure Cloud Service Project.


##Required Tools
* Visual Studio
* <a href="https://www.microsoft.com/en-us/download/details.aspx?id=44938" target="_blank">Microsoft Azure SDK 2.5</a>

##Diagnostics
Noting the <a href="https://msdn.microsoft.com/en-us/library/azure/dn186185.aspx" target="_blank">differences between the Azure SDK versions</a>, both diagnostic and caching has changed since 2.4.1:

* New diagnostic configuration file (diagnostics.wadcfgx) introduced in 2.5.
* Configuration via code has been removed.

##Azure Package Creation
Upon converting your web project to a Cloud Service project, publishing the cloud service project generates your .cspkg file in the bin of your Cloud Service project and publishes to Azure with credentials entered when prompted at the start of the publishing process.

In preparation for deployment, any files included in your web project must exist, as missing references to any files will prevent your deployments from succeeding. Visual studio will fail the build and mark the missing references as errors.

![azure package creation](/assets/images/sitecore8-azure-1.png)

Output of the build will resemble:

![azure package output](/assets/images/sitecore8-azure-2.png)

##Deployment Experience
The benefits of this approach allow for publishing to Azure directly from Visual Studio, which is fine if you're a developer and already familiar with the IDE.  If you're not as tech-savvy, this process can be automated through command-line deploys using MSBuild or through Azure PowerShell (more on this later).

##Solution Configuration
In order to deploy from Visual Studio, all necessary files to run Sitecore under a cloud service will need to be included in your solution, otherwise, they will not be deployed. Such a configuration is helpful in that you can see all files necessary for deployment, but quickly can clutter-up your solution and projects. Not to mention performance implications of including all the necessary folders and files required for either a CE or CD instance. 

This may just be a personal preference of mine, but I like to keep my project file and folder inclusions to a minimum. Including only those files needed for my local development instance, while copying and merging in dependencies through build scripts specific to each deployment target.

##Conclusion

Go with this approach if you're testing a local sandbox environment and do not have a solid automated build and deployment process in place.

#Option 2: Azure PowerShell

Much like the approach with Visual Studio, deployment to Azure is a two step process -- create the package then deploy it to Azure. You can generate the .cspkg file directly by calling <a href="https://msdn.microsoft.com/en-us/library/azure/gg432988.aspx" target="_blank">cspack.exe</a> from the command line. Azure PowerShell will take care of the deployment.

Prior to deploying via PoweShell, you must first install Microsoft Azure PowerShell on the server you wish to deploy from. The Microsoft Azure PowerShell cmdlets allow for obtaining and pushing assets to the cloud. Following a similar approach for configuring <a href="http://azure.microsoft.com/en-us/documentation/articles/cloud-services-dotnet-continuous-delivery/#step4" target="_blank">continuous delivery of a Cloud Service to Azure</a>, we can write a PowerShell script to deliver Sitecore 8 to Azure using PowerShell cmdlets. 

Common cmdlets used in deploying Sitecore 8 to Azure:

* <a href="https://msdn.microsoft.com/en-us/library/dn495203.aspx" target="_blank">Select-AzureSubscription</a>
* <a href="https://msdn.microsoft.com/en-us/library/dn495189.aspx" target="_blank">Set-AzureSubscription</a>
* <a href="https://msdn.microsoft.com/en-us/library/azure/dn495134.aspx" target="_blank">Get-AzureStorageAccount</a>
* <a href="https://msdn.microsoft.com/en-us/library/azure/dn495119.aspx" target="_blank">New-AzureService</a>
* <a href="https://msdn.microsoft.com/en-us/library/azure/dn495143.aspx" target="_blank">New-AzureDeployment</a>
* <a href="https://msdn.microsoft.com/en-us/library/azure/dn495146.aspx" target="_blank">Get-AzureDeployment</a>

##Required Tools:
* PowerShell
* <a href="http://azure.microsoft.com/en-us/documentation/articles/powershell-install-configure/" target="_blank">PowerShell Azure Cmdlets</a>
* <a href="https://www.microsoft.com/en-us/download/details.aspx?id=44938" target="_blank">Microsoft Azure SDK 2.5</a>

##Deployment Authentication Options

For automated deployments, you must also automate authentication in order to push your build to either a new or existing cloud service. There are two options: use a Publish Settings file or through a management certificate. 

###1. Publish Settings File
	
The first, publish settings file, contains your Azure subscription information . You can use this file to set the active subscription through ```Import-AzurePublishSettingsFile``` followed by ```Get-AzureSubscription```. While this is a useful approach, note that your publish settings file allows full access to all features available by your subscription. 

![publish settings file](/assets/images/sitecore8-azure-5.png)

###2. Management Certificate

Another option is to use a <a href="https://msdn.microsoft.com/en-us/library/azure/gg432987.aspx" target="_blank">Service Certificate for Azure</a>. Generate a certificate local to the machine where you will be deploying. You can do this either from the <a href="https://msdn.microsoft.com/en-us/library/azure/gg551722.aspx" target="_blank">command line</a> or directly in the MMC Certificate Snap-in. Thanks to <a href="https://twitter.com/kamsar" target="_blank">Kam Figy</a> on discovering this approach.

If using the MMC Snap-in, export the certificate to a .cer file by clicking on "Copy to File…".

![certificate](/assets/images/sitecore8-azure-6.png)

Upload the certificate file to your subscription in Azure. Notice the Thumbprint will then be listed along with your certificate name.

![azure portal](/assets/images/sitecore8-azure-7.png)

You can then use the certificate thumbprint as a parameter for ```Set-AzureSubscription```, eliminating the need for obtaining and importing the publish settings file.

_Note that calling ```Get-AzurePublishSettingsFile``` also generates a certificate. That certificate has a default expiration of 24 hours._

###Example

The following is Azure PowerShell cmdlets output for deploying Sitecore 8 to a new Cloud Service:

![azure portal](/assets/images/sitecore8-azure-8.png)

While the script is running, you will periodically receive updates on progress, such as uploading:

![azure portal](/assets/images/sitecore8-azure-10.png)

and deployment of the new cloud service:

![azure portal](/assets/images/sitecore8-azure-9.png)

##Conclusion

Go with Azure PoweShell for any continuous delivery solutions or automated deployments through build scripts. 
