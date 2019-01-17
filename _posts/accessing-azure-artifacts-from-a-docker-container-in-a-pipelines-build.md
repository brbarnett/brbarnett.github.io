---
layout: post
title: Accessing Azure Artifacts from a docker container in a Pipelines build
tags: azure azure-devops pipelines nuget artifacts docker
---

image

In my [previous post]({{ site.baseurl }}/blog/2019/01/2019-1-16-using-azure-pipelines-to-publish-nuget-packages-to-a-private-artifacts-feed/) I talked about how our team is publishing NuGet packages using Azure Pipelines to a DevOps Artifacts feed. Now it's time to consume those packages. There are plenty of examples on how to connect to a feed in Build, but our case is a bit different because our applications are containerized. The purpose of this post is to share specifically how we authorize access to Artifacts from a docker build context.

<!--more-->

Please see [this Gist](https://gist.github.com/brbarnett/c55c80dd63b89465cfd9bc6b74c0548e) for full code references.

## Security
The purpose of using Azure Artifacts is to avoid a public package manager because the team is writing IP into shared libraries. It should then come as no surprise that there are a few additional steps required to ensure that your docker build context has appropriate access to Azure DevOps.

The docker build context is isolated from Azure DevOps, meaning that it should be considered a sandboxed build that doesn't share environment variables and access that typical Build tasks would. The trick to authorizing a docker build is to use a Personal Access Token as a rotatable key that tells Azure DevOps that you are authorized to pull packages from Artifacts. Start by creating a PAT: click your avatar in the upper right of DevOps, then Security in the dropdown. **Make sure it has access to Packaging.**

## Pipelines variable group
The next step is to create a group of variables that you can share between all build definitions. I created an Artifacts group and added an `artifactsAccessToken` (this is your Private Access Token) and `artifactsEndpoint` which I got from clicking Connect to Feed from my Artifacts feed.

image

## CI and CD builds
Go ahead and create your CI and CD builds - feel free to use my files as a template. Note that I am specifically passing in the variables I created in the variable group above using the following method:

```
...

variables:
- group: Artifacts

...

steps:
- task: Docker@1
  displayName: 'Build an image'
  inputs:
    ...
    arguments: '--build-arg ARTIFACTS_ENDPOINT=$(artifactsEndpoint) --build-arg ACCESS_TOKEN=$(artifactsAccessToken)'
```

This ensures that the variables get passed to the `Dockerfile` ARGS that we also need to define.

**Important: once you create the build in DevOps, you need to click through Variables > Variable groups and click _Link variable group_. This is a security feature.**

## Dockerfile
Now that we have builds set up, create a `Dockerfile` that defines how your app is build and run. To use the variables we passed in via Build above, create `ACCESS_TOKEN` and `ARTIFACTS_ENDPOINT` ARGS to accept them. The key changes from a typical `Dockerfile` is that you need to install the [Azure Artifacts Credential Provider](https://github.com/Microsoft/artifacts-credprovider) and then set the `NUGET_CREDENTIALPROVIDER_SESSIONTOKENCACHE_ENABLED` and `VSS_NUGET_EXTERNAL_FEED_ENDPOINTS` environment variables to let it do its job. Also, alter your `dotnet restore` task to use the new feed endpoint.