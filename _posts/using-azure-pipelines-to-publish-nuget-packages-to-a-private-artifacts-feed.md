---
layout: post
title: Using Azure Pipelines to publish NuGet packages to a private Arifacts feed
tags: azure azure-devops pipelines nuget artifacts
---

image

My team is building a suite of small ASP.NET Core WebAPI projects for one of our clients - call it dabbling in microservices - and we have applied a few standards (e.g., implementing Swagger, and a `X-Correlation-ID` header) across all of the apps. Instead of repeating the same configuration code many times, we have decided to use [Azure Artifacts](https://azure.microsoft.com/en-us/services/devops/artifacts/) to manage a common NuGet package across all implementations. This post explains how we automated publishing of one of our packages using [Azure Pipelines](https://azure.microsoft.com/en-us/services/devops/pipelines/).

<!--more-->

Please see [this Gist](https://gist.github.com/brbarnett/f7f4788f13d9a70bdd6ef997180a7c1e) for full code references.

## Project structure
The project we're working with is a simple .NET Core class library that we'd like to use to configure ASP.NET Core startup. For reference, our project looks like this:

```
/src
  /lib - .NET Core class library project (this is the main package entrypoint)
  /test - .NET Core test project
azure-pipelines.cd.yaml
azure-pipelines.ci.yaml
```

Remember the project structure as it informs how the CI and CD builds are set up, and that our project is using YAML build definitions for Pipelines.

## Versioning
We are using modified [semver](https://semver.org/) to version our NuGet packages with the format `[major].[minor].[buildId]`, with an optional `-prerelease` suffix for packages not quite ready for production. We're using the build ID instead of patch because it's easier to trace back to a build, and a new build suggests that we have updated code so usually falls under the definition of a patch. 

The following needs to be added to the `lib`'s project file to enable this versioning scheme:

```
<PropertyGroup>
  <!-- This version number uses modified SemVer as Major.Minor.BuildId. Change the Major.Minor version to increment NuGet package (e.g., 0.1 to 0.2) -->
  <Version>0.1$(VersionSuffix)</Version>
</PropertyGroup>
```

## CI and CD builds
We have two build definitions in this project that accomplish different goals:

- CI build - this build is kicked off automatically on a feature branch when a PR is created due to a build validation step on the master branch policy. Its job is to restore packages and build each of the `lib` and `test` projects, and then to run our unit tests to promote code quality.
- CD build - this build is kicked off automatically on the master branch using a CI trigger defined in the build definition. It does everything the CI build does, but it also packs our build artifacts into versioned NuGet packages and publishes those packages for use by our Release pipeline.

Note that the CD build is actually creating two packages: `[major].[minor].[buildId]` and `[major].[minor].[buildId]-prerelease`. We did this for two reasons:

- Our approach is that all artifacts should be created by Build and are immutable thereafter. All configuration should be passed in at runtime.
- NuGet packages and their versions are immutable, so we can't update from prerelease to release.

## Release
We have set up two release environments, which is how we push prerelease packages for testing and then promote those packages to release.

![_config.yml]({{ site.baseurl }}/images/using-azure-pipelines-to-publish-nuget-packages-to-a-private-artifacts-feed/release.jpg)

Each release only has a single `dotnet push` task that targets our custom Artifacts feed for distribution.

This is the result of the Release process:

![_config.yml]({{ site.baseurl }}/images/using-azure-pipelines-to-publish-nuget-packages-to-a-private-artifacts-feed/artifacts.jpg)

I'll follow up with a subsequent post on how to consume these artifacts from a container build definition.