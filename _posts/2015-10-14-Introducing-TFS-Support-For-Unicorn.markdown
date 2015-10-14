---
layout: post
title:  "Introducing TFS Support for Unicorn"
date:   2015-10-14 20:35:00
categories: Sitecore Unicorn
comments: true
---
Where would we be without Unicorn? It fits nicely into both distributed and centrally located teams. Unicorn runs behind the scenes, syncing updates to the file system, while keeping development teams moving forward and focused on coding. It's a tool that allows us to be successful without getting in the way. For me, at least, it's a must have on any new project. 

Recently, a client project called for TFS 2010 as the main repository for SCM. This immediately presented a roadblock in using Unicorn. With TFS 2010, and server workspaces in TFS 2012 and later, file system access is completely restricted until first checking out files for edit within TFS. Since Unicorn runs in the background, updating the file system on item save, insert or remove events, file access was denied by read-only permissions set on each file in the repository. 

#TFS Plug-In for Rainbow

Unicorn's default target syncing provider, or Rainbow, provides direct access to the file system. The <a href="https://github.com/PetersonDave/Rainbow.Tfs" target="_blank">TFS Plug-In for Rainbow</a>, extends the default provider by first communicating with TFS prior to any action taken by Rainbow when performing a sync operation.

There are two areas of interest to note when using the TFS plug-in:

##1. TFS API Dependencies

Pre-2015 APIs for TFS are in the form of 32-bit assemblies. In order to use this plug-in, you must <a href="https://github.com/PetersonDave/Rainbow.Tfs#iis" target="_blank">update the application pool</a> within your development instance for 32-bit support. Without it, Rainbow will not be able to communicate with TFS. 

##2. Credentials

Two sets of credentials are required. The first, for file system access. The identity of the application pool must run as the current logged in developer. This is necessary for the TFS API, as the library reads from local TFS cache stored in the local data folder of the current user's profile. <a href="https://github.com/PetersonDave/Rainbow.Tfs#iis" target="_blank">See steps 4-7 in the repository's README</a>.

Cache locations are typically in the following directories when connecting to TFS using Team Explorer:

<ul>
	<li><strong>Visual Studio 2013:</strong> C:\Users\[profile]\AppData\Local\Microsoft\Team Foundation\5.0\Cache</li>
	<li><strong>Visual Studio 2015:</strong> C:\Users\[profile]\AppData\Local\Microsoft\Team Foundation\6.0\Cache</li>
</ul>

The second set of credentials is for access to the TFS repository. Those can be found in the <a href="https://github.com/PetersonDave/Rainbow.Tfs#rainbow" target="_blank">TFS-specific rainbow config file</a> within your ```App_Config\Include\Unicorn``` folder.

#Performance

Since Rainbow must communicate with TFS prior to taking action on the file system, Unicorn performance takes a slight hit. If you're used to U3 outside of TFS, you'll notice a minor lag when editing items included within the serialization predicate settings of Unicorn. Sitecore log updates maintain status when communicating with TFS and alert the client if communication with TFS is no longer active. 

#Install It Today

The TFS Plug-In for Unicorn is in early Beta. Read through the README for additional information and installation steps. If you install it, feedback is much appreciated.