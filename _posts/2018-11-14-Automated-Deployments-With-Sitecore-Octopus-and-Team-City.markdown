---
layout: post
title:  "Automated Deployments with Sitecore Octopus and Team City"
date:   2018-11-14 20:00:00
categories: Sitecore
comments: true
---
Once thought of as a luxury, an automated deployment strategy is now largely considered a necessity as Sitecore expands in terms of complexity, dependencies and standardization of configuration. Having gone through the pain of manual deployments using build scripts, diff tools and carefully updating of configuration files, it's easy to see the need for a reliable and repeatable deployment process. Don't think this is a need across the Sitecore community? Attend any session at Sitecore Symposium or user group centered around this topic. Sitecore Symposium specifically had standing room only for talks on continuous deployment and delivery.

# A Two-Phased Approach

The ultimate goal is to deliver continuous deployment, a process in which developers can commit directly to the master branch, kicking off builds, a suite of automated tests, then packaging and deploying updates seamlessly without interruption to end-users and editorial team members. Getting there is much easier during a new build, as dependencies are much more manageable. For any new build, this should not be an after-thought, but a requirement before going live. 

For those of you building new Sitecore implementations, please advocate for this up front rather than under consideration as a "phase two" deliverable. Your clients will thank you later.
For us, however, moving directly into a continuous delivery strategy is much harder, as our solution is much more mature than a new build with a considerably large set of content and a need for high availability within the editorial team. A two-phased approach made the most sense for us:

## Project Phases

1. Automate the deployment process
2. Move to continuous delivery

In this post, I'll cover phase one at a high-level, as we're still working our way to the ultimate goal of continuous delivery.

# Phase 1: Automate the deployment process

## Continuous Integration

There are numerous options for continuous integration products. We won't go into specifics here, as each option has their own set of advantages and disadvantages. For us, Team City made the most sense. We knew we wanted a product that integrated well with Octopus Deploy and supported a <a href="https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow" target="_blank">GitFlow strategy</a> for building and pushing of deployment packages. As for versioning, we use <a href="https://gitversion.readthedocs.io/en/latest/" target="_blank">GitVersion</a> for semantic versioning of our NuGet packages to Octopus.

Following GitFlow, our GitVersion configuration file resembles the following

{% highlight xml %}
branches:
  dev(elop)?(ment)?$:
    tag: alpha
  hotfix(es)?[/-]:
    mode: ContinuousDeployment
    tag: hotfix
  releases?[/-]:
    mode: ContinuousDeployment
ignore:
  sha: []
{% endhighlight %}

With the above configuration, specific branches are monitored, which results in packages being named and pushed to Octopus in various channels based on their package names.

## Branching Strategy

* _Develop_ - Packages are prefixed with "alpha" and sent to the Alpha deployment channel in Octopus deploy.
* _Hotfix_ - Packages are prefixed with "hotfix" and sent to the Hotfix deployment channel in Octopus deploy.
* _Release_ - Packages are prefixed with "beta" (default in GitVersion) and sent to the Beta deployment channel in Octopus deploy

The rest of the configuration is rather standard, building and deploying NuGet packages:

1. _Website package_ - The post-build website root, excluding any and all OOTB Sitecore files.
2. _Unicorn package_ - All YAML files for Unicorn syncing

## Octopus Deploy

Configuring Octopus Deploy, we had the following set of goals:

1. Automate the delivery of new builds
2. Replace variables through Octopus per environment and role
3. Leave CM vs CD configuration transformations to Sitecore through roles-based configuration in Sitecore 9
4. Automate delivery to our testing environment, for all servers simultaneously
5. Introduce manual intervention after deploying to the CM, CD cluster 1 and CD cluster 2
6. Automate the notification of success or failure through Slack

### Environments

* _CI_ - Continuous integration server, for anything that needs to be execute via OD tentacle on the CI server itself. Needed for dependencies prior to executing deployments on behalf of a tentacle.
* _QA_ - All test servers, both CM and CDs.
* _Production CM_ - Editorial environment
* _Production CD Cluster 1_ - Delivery cluster 1
* _Production CD Cluster 2_ - Delivery cluster 2

### Channels

* _Alpha_ - Develop branch, manually triggered via OD for testing purposes. OD channel version range: `^alpha`
* _Beta_ - Release branch, automatically deployed to test and manually triggered to production. This represents our release candidates. Bug fixing directly in the release branch triggers updated builds to test. OD channel version range: `^beta`
* _Hotfix_ - Hotfix branch, for automated deploys to test and manually triggered to production. OD channel version range: `^hotfix`
* _Production_ - Master branch, for snapshots of post release packages. Necessary for rollbacks. OD channel version range: `$^`

### Example Channels and Package Versioning

Notice the semantic versioning of the packages and prefixed with the appriate channel name. GitVersion is critical to the naming and placement of these packages for octopus in their respective channels.

#### Alpha Channel
![alpha](/assets/images/alpha-channel.png)

#### Beta Channel
![beta](/assets/images/beta-channel.png)

#### Hot Fix Channel
![hotfix](/assets/images/hotfix-channel.png)

#### Production Channel
![production](/assets/images/production-channel.png)

### Process

The following is a high-level overview of what our deployment process looks like via Octopus Deploy

1. Take server out of load balancer
2. Wait (for loan balancer)
3. Stop IIS
4. Deploy Unicorn package
5. Clear web root folders
6. Deploy website package
7. Remove configuration transforms
8. Start IIS
9. Verify Sitecore Login Page (CM only)
10. Sync Unicorn (CM only)
11. Verify mult-site landing pages
12. Place server back in load balancer
13. Notify via Slack

The process for deployments leverages OOTB Octopus configuration transformation features to <a href="https://octopus.com/docs/deployment-process/variables/environment-specific-configuration-transforms-with-sensitive-values-in-octopus" target="_blank">transform config files by environment</a>. We commit environment-specific configuration transforms for this purpose and let Octopus swap out values with properly scoped variables.
