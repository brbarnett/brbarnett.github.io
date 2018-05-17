---
layout: post
title: Reference architecture&colon; Angular 4 and Web API on .NET Core 2.0 Preview 1 with Azure AD authentication
redirect_from: "/blog/2017/06/reference-architecture-angular-4-and-web-api-on-net-core-20-preview-1-with-azure-ad-authentication/"
tags: angular azure azure-ad spa dotnet-core
---

The more time I spend in the tech community and market these days, the more it has become clear that the appetite for newer, more cutting-edge technologies has gone up. Lately, the new kids on the block that we're tending to build more apps in right now are .NET Core 2.0 and Angular 4, and since we typically build internal applications, I'm seeing Azure AD used a lot for authentication. Since I am seeing this type of application so often, I thought it would be a good idea to build a reference architecture to tie it all together as a quickstart.

Reference Github repository: https://github.com/brbarnett/dot-net-core-spa (read the solution readme for setup/deployment details)

I am assuming you have read my [post on hosting Angular 4 apps in .NET Core]({{ site.baseurl }}/blog/2017/06/hosting-angular-4-in-a-net-core-20-preview-1-application/): 

### High-level architecture
The solution is based on two primary web-hosted projects: an Angular 4 SPA and a second Web API project. These applications are built to be hosted separately to promote scalability and deployment independence. The Client/Web application communicates with the API application via CORS across domains. Authentication is handled by the Angular application, which passes a bearer token to the API application to be validate against Azure AD. It looks something like this:

![_config.yml]({{ site.baseurl }}/images/2017-6-13-reference-architecture-angular-4-and-web-api-on-net-core-20-preview-1-with-azure-ad-authentication/angular-architecture.jpg)

### Web application
The Web project/application should almost exclusively be comprised of the front-end application; in this case, that's the Angular app. The only purpose of the server-side application (ASP.NET Core in this case) is to handle URL routing, and all other logic is handled in the Angular TypeScript files. All data that feeds the application will come from cross-domain AJAX calls from the Web app to the API app.

My favorite reason for separating out my applications this way is for deployment: I can manage my continuous integration pipeline independently for builds of the client application and the API. This ultimately reduces risk by decreasing unnecessary code deployments (e.g., if I were to update a TypeScript file but my CI pipeline picked up the entire build for Web + API) and has an added benefit of speeding up my releases. I haven't set it up this way in my Github repo, but another benefit we see is that I can now scale my API independently from my client application, which can have major cost savings implications.

Note that because we're making cross-domain requests, we have to use CORS and can no longer use relative URLs. I'll cover the CORS part in the next section, but for this application I want to note that I created a URL service whose purpose is to read in environment-specific configuration (see `./src/environments`) and generate an absolute URL to the API.

You do not need any authentication UI components, as that is all handled by Microsoft. When the application code determines the user does not have a token or their token is expired, it redirects to Azure AD which then will post them back to the app with a JWT token. All AJAX requests to the API must include that bearer token in their headers.

### API application
The API application has no client-side code - in fact, its purpose is to host restful endpoints that the client application consumes via AJAX requests. Remember that we're using CORS, so we have to pull in environment-specific URLs and open them up for CORS requests, handled by the Startup class.

Authentication and authorization are handled by this application. We are using Azure AD via JWT bearer authentication, which means that the client application needs to pass a token via headers. Decorating your controllers and/or methods with the `[Authorize]` attribute will lock down your APIs to only those who pass valid tokens in request headers.

### Azure deployment
The last component I've included is an Azure ARM template project, which allows a quick deployment of the resources required to host this multi-project solution. It includes a (mostly) auto-generated PowerShell file, a resource template and individual environment-specific parameters files. Please follow the instructions in the Github readme to deploy your infrastructure.