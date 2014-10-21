---
layout: post
title:  "Verifying Sitecore Azure Deployments Against Specified Path or File Name Too Long Error"
date:   2014-8-15 10:18:00
categories: Sitecore Azure
comments: true
---

When running a Sitecore Azure deployment, ever see this error?

![azure build error](/assets/images/azure-build-error.png)

<em>"The specified path, file name, or both are too long. The fully qualified file name must be less than 260 characters, and the directory name must be less than 248 characters"</em>

This is [not an error specific to Sitecore](https://www.google.com/search?q=%22The+specified+path%2C+file+name%2C+or+both+are+too+long.+The+fully+qualified+file+name+must+be+less+than+260+characters%2C+and+the+directory+name+must+be+less+than+248+characters%22&amp;oq=%22The+specified+path%2C+file+name%2C+or+both+are+too+long.+The+fully+qualified+file+name+must+be+less+than+260+characters%2C+and+the+directory+name+must+be+less+than+248+characters%22&amp;aqs=chrome..69i57.752j0j1&amp;sourceid=chrome&amp;es_sm=93&amp;ie=UTF-8).

In an attempt to better understand the Sitecore Azure API and find the files/paths preventing a successful deploy, I built the [Sitecore Azure Build Verifier](https://github.com/PetersonDave/AzureBuildVerifier). This tool will execute a dry-run of a Sitecore Azure deploy, alerting you of files and folders which fail validation.

Follow the [instructions](https://github.com/PetersonDave/AzureBuildVerifier#installation) within the project ReadMe to install.

## Configuring a Build

What I found interesting about this problem was the fact that I had installed a test version of Sitecore 7.2 with Azure 7.2 (rev 140411) under the path <em>C:\inetpub\wwwroot_sandbox\Azure72 </em>-- not an overly verbose instance name or root path location.

For those of you new to Azure in Sitecore, specifying the target location for builds is located within an Environment item, under the field <em>Build Folder</em>.

![environment item](/assets/images/environment-item.png) 

As you can see, the default is <em>C:\inetpub\wwwroot_sandbox\Azure72\Data\AzurePackages.</em> Sitecore Azure will, also by default, append additional folders to this path for each build instance executed from within the Azure module interface resulting in a build path such as:

<em>C:\inetpub\wwwroot_sandbox\Azure72\Data\AzurePackages\(19) DevFabric\DevFabricLCd01Role01ScBaf20140816030333.cspkg\roles\SitecoreWebRole\approot</em>

Quickly looking at the path above, you'll notice references to:

* Build instance
* Environment name
* Role instance and name
* DNS host name

## Quick Fix

Simply change the build folder to something with a shorter path. For example, set your path to <em>C:\AzureBuild</em> and you're past this issue.

## Verifying The Build

If you're interested in which files/folders are preventing a successful deploy, install the Azure Build Verifier module. After installation, select an Azure Deployment item and right-click. Select the "Verify Deployment" option:

![context menu](/assets/images/context-menu.png)

Upon making the selection, you'll be presented with a dialog box displaying the list of files and paths which exceed the limits previously seen in the failed deployment:

![dialog](/assets/images/dialog.png)

If you're interested in the implementation, check out the [GitHub repo](https://github.com/PetersonDave/AzureBuildVerifier). Dig into the [artifacts processor](https://github.com/PetersonDave/AzureBuildVerifier/blob/master/Source/AzureBuildVerifier/Processors/Sharknado/ArtifactsPipelineProcessor.cs) or the main [verify class](https://github.com/PetersonDave/AzureBuildVerifier/blob/master/Source/AzureBuildVerifier/Verifier.cs) leveraging the Sitecore.Azure library to [resolve its sources](https://github.com/PetersonDave/AzureBuildVerifier/blob/master/Source/AzureBuildVerifier/App_Config/Include/AzureBuildVerifier.Pipeline.config) prior to processing.