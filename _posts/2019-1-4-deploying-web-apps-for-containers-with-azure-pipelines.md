---
layout: post
title: Deploying Web Apps for Containers with Azure Pipelines
tags: azure azure-devops pipelines docker devops
---

![_config.yml]({{ site.baseurl }}/images/2019-1-4-deploying-web-apps-for-containers-with-azure-pipelines/web-apps-containers-azure-pipelines.jpg)

I am currently working on a project where we have elected to use .NET Core and have been asked to allow deployment flexibility. The platform versatility of .NET Core allows us the option to containerize our apps, which is exactly what we're doing. But instead of diving into the deep end of what could eventually become Kubernetes right off the bat, we want to keep things simple. In this case, we've chosen to use [Azure Web Apps for Containers](https://azure.microsoft.com/en-us/services/app-service/containers/). This post walks through our code management process and continuous delivery of infrastructure and containers.

Please see [this Gist](https://gist.github.com/brbarnett/7cacd4a30bed946e9ad681c261765fbd) for full code references.

## Code management
The overall solution we are creating has somewhere in the range of 10 API applications that need to be deployed. We are managing each of the apps within its own repository in [Azure DevOps](https://azure.microsoft.com/en-us/services/devops/) in order to tie the application deployment directly to a modified [Gitflow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow) process.

## Infrastructure as code
There is nothing special about the [App Service Plan deployment](https://gist.github.com/brbarnett/7cacd4a30bed946e9ad681c261765fbd#file-deploy-json-L37-L50) in this case, provided that you use the `"kind": "linux"` flag because Web Apps for Containers does not work on Windows as far as I know.

They key to making this work is within the [Web App resource definition](https://gist.github.com/brbarnett/7cacd4a30bed946e9ad681c261765fbd#file-deploy-json-L52-L87), specifically the `siteConfig` values. You may have seen other posts add an additional `siteConfig` value of `"linuxFxValue": "DOCKER|<image-name>:<tag>"`, which only works for ARM template deployment. **Note**: if you are going to use [Azure Pipelines](https://azure.microsoft.com/en-us/services/devops/pipelines/) (as I am in this post), _do not set this value_ as it will get overwritten every time you deploy your ARM template. Our Release will override this value.

## Continuous delivery
Since we're using containers, the build definitions are fairly simple because the process steps are defined within our `Dockerfile`. I have included my YAML [CI](https://gist.github.com/brbarnett/7cacd4a30bed946e9ad681c261765fbd#file-azure-pipelines-ci-yaml) and [CD](https://gist.github.com/brbarnett/7cacd4a30bed946e9ad681c261765fbd#file-azure-pipelines-cd-yaml) build definitions in the Gist for reference.

The delivery process looks like this:
1. Developer makes changes in a feature branch.
2. Developer creates a Pull Request. CI Build is automatically queued, which is required for merge.
3. Lead merges Pull Request. CD Build is automatically queued.
4. CD Build pushes container image automatically to our [Azure Container Registry](https://azure.microsoft.com/en-us/services/container-registry/) (ACR).
5. Release is triggered automatically on new ACR push. This is important, because a typical release would trigger on CD build completion. We made this decision to keep our options open later -- for example, this allows us to make minimal changes if we move to Kubernetes with Spinnaker instead of Pipelines.
6. Dev environment Web App underlying image is updated by Release.

The Release only has a single `Azure App Service Deploy` task. To configure it to deploy containers, switch the App Type to Linux Web App, which then defaults to an image source of Container Registry.

I took a look at the logs from when the task runs, and it outputs the following:

```
Updating App Service Configuration settings. Data: {"appCommandLine":null,"linuxFxVersion":"DOCKER|<ACR-instance-name>.azurecr.io/<image>:<build-id>"}
```

This is why it's OK that you left the `linuxFxVersion` value empty in the ARM template since Release overrides it.