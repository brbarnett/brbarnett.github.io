---
layout: post
title: Why isn't Azure App Service Deploy copying files to my Web App's wwwroot?
tags: azure azure-devops pipelines
---

![_config.yml]({{ site.baseurl }}/images/2019-2-20-why-isnt-azure-app-service-deploy-copying-files-to-my-web-apps-wwwroot/header.jpg)

I was recently working with a collegue to deploy a Sitecore app to Azure via Pipelines. The release was running successfully, but the deployed files didn't seem to be overlaying the required, pre-loaded Sitecore files. When we viewed `/site/wwwroot` from Kudu, we were seeing only the files from the build artifacts; when we viewed the same directory from FTP, we were seeing only the Sitecore files. What gives?

<!--more-->

## The cause: Azure App Service Deploy task version
Sometime over the past few months, the Azure App Service Deploy release task versioned to `4.*`. In that release, the Microsoft team created an auto-detect feature that tries to guess the best method for code deployment to your Web App. In our case, it chose the new preferred method called _Run From Package_, which has a few quirks.

## What's new in version `4.*`
These are the new features in `4.*` per the Azure DevOps UI at the time of writing:

- Supports Zip Deploy, Run From Package, War Deploy 
- Supports App Service Environments
- Improved UI for discovering different App service types supported by the task
- **Run From Package is the preferred deployment method, which makes files in wwwroot folder read-only**
- [Click here  for more information.](https://github.com/Microsoft/azure-pipelines-tasks/blob/master/Tasks/AzureRmWebAppDeploymentV4/README.md)

## How does that work?
When the release task deploys using _Run From Package_, it does a few things:

- Zips build artifacts and uploads to `/data/SitePackages/`.
- Creates a `/data/SitePackages/packagename.txt` file whose contents point to the deployed zip filename.
- Sets a new Application Setting to the app of `WEBSITE_RUN_FROM_PACKAGE=1`.

The effect of the `WEBSITE_RUN_FROM_PACKAGE` setting is that the `/site/wwwroot/` directory now vitually points to the deployed zip file, not to the actual underlying file system. Try it: download the package zip from `/data/SitePackages/` and compare contents with `/site/wwwroot/` in Kudu. If you extract FTP credentials from the web deploy and access `/site/wwwroot/`, there is a completely different site of files. Here's what I'm seeing from a fresh .NET Core MVC deployment:

### Kudu
![_config.yml]({{ site.baseurl }}/images/2019-2-20-why-isnt-azure-app-service-deploy-copying-files-to-my-web-apps-wwwroot/kudu-contents.jpg)

### FTP
![_config.yml]({{ site.baseurl }}/images/2019-2-20-why-isnt-azure-app-service-deploy-copying-files-to-my-web-apps-wwwroot/ftp-contents.jpg)

## The solution
When Microsoft updated the release task, they made a major change to the default behavior of the task and hid the option under the _Additional Deployment Options_ section.

![_config.yml]({{ site.baseurl }}/images/2019-2-20-why-isnt-azure-app-service-deploy-copying-files-to-my-web-apps-wwwroot/deployment-options.jpg)

When the _Select deployment method_ option is not checked, the task appears to prefer the _Run From Package_ option. Since I want to overlay my deployment files over existing files, the solution for us was to select the option and choose Web Deploy as our deployment method.

![_config.yml]({{ site.baseurl }}/images/2019-2-20-why-isnt-azure-app-service-deploy-copying-files-to-my-web-apps-wwwroot/web-deploy.jpg)

Web Deploy matches the previous behavior of the release task in that it overlays my build artifacts without removing any of the destination files.

![_config.yml]({{ site.baseurl }}/images/2019-2-20-why-isnt-azure-app-service-deploy-copying-files-to-my-web-apps-wwwroot/new-ftp-contents.jpg)

I also noted that when I switched the deployment method to Web Deploy, the release task automatically set the `WEBSITE_RUN_FROM_PACKAGE` setting to `0`, which disabled the `/site/wwwroot/` mismatch we observed.