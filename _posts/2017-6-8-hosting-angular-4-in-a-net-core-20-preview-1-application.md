---
layout: post
title: Hosting Angular 4 in a .NET Core 2.0 Preview 1 application
redirect_from: "/blog/2017/06/hosting-angular-4-in-a-net-core-20-preview-1-application/"
---

For reasons I'll detail in a later post, I prefer to host my SPA and API applications separately. Combine that with the fact that I prefer to stay on the Microsoft stack and it raises the question: what is the best hosting environment for my Angular (or any SPA) application?

If you're from the JavaScript/Node side of town, the answer is easy: Node is a great because it's lightweight and configuration is very simple. Thankfully these days Microsoft has a real competitor in .NET Core -- you get all the benefits of a serious application without the bloat that comes with a ASP.NET 4.x application. So now if I've bought in and decided on .NET Core, how do I actually serve my application? Let's do a little bit of configuration to use the /wwwroot directory that comes with .NET Core applications.

Reference Github repository: https://github.com/brbarnett/dot-net-core-spa

Using the [Angular CLI](https://github.com/angular/angular-cli) makes building Angular apps dead simple. When using the CLI, though, there's no great way to create both the Web app (.NET Core) and Angular app in the same directory. My process is as follows:

1. Create a new empty .NET Core application with no authentication (remember we've separated the two -- auth is handled by the API using bearer tokens)
2. Rename the Web app to `[app-name].bak` on the disk
3. [Install the Angular CLI](https://github.com/angular/angular-cli#installation) if you haven't
4. Run `ng new app-name`
5. After the npm packages all install, cut everything from your .NET Core app backup and paste into the new Angular project app directory
6. Delete the `[app-name].bak` project directory

Now that you have that created, there are a few more steps to ensuring that .NET Core and Angular play well together. There are basically two issues, which are that Angular wants to use a /dist directory by default, and that Visual Studio wants to build TypeScript when you build the project. Here's how to work around that:

1. In the `.angular-cli.json` and `tsconfig.json files`, replace the "outDir" property's "dist" value with "wwwroot"
2. Add the following to the first PropertyGroup in your .csproj file: `<TypeScriptCompileBlocked>true</TypeScriptCompileBlocked>`
3. Add `/wwwroot` to your Angular project's `.gitignore` file. There shouldn't be any content in there that the CLI doesn't generate on `ng build`

One last thing for actually hosting your project in Azure or similar is that you have to tell the .NET routing engine to forward all routes to your main index.html file. To do this, I've encapsulated this logic into a static method you can pull into your project:

```
public static IApplicationBuilder UseAngularRouting(this IApplicationBuilder app, string indexPath,
    string[] ignoredRoots)
{
    return app.Use(async (context, next) =>
    {
        await next();

        if (ignoredRoots.Any(root => context.Request.Path.Value.StartsWith(root))) return;

        // If there's no available file and the request doesn't contain an extension, we're probably trying to access a page.
        // Rewrite request to use app root
        if (context.Response.StatusCode == 404
            && !Path.HasExtension(context.Request.Path.Value))
        {
            context.Request.Path = indexPath; // Put your Angular root page here 
            context.Response.StatusCode = 200; // Make sure we update the status code, otherwise it returns 404
            await next();
        }
    });
}
```

Credit: http://benjii.me/2016/01/angular2-routing-with-asp-net-core-1/

Usage is the following within `Startup.cs`:

```
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    app.UseAngularRouting("/index.html", new[] {"/api/"});
}
```

Basically, this is ensuring that all URLs get handled by `index.html`, which is what loads your Angular application with the Router module. Optionally, you can exclude the `/api/` path if you're using the same application for your SPA and API and need to let the engine handle that fragment differently.

That's it - for local dev, run `ng serve`. For deployment to Azure, just add a publish profile to a Web App like you normally would, work `ng build --prod` into your pipeline and publish. It should take the /wwwroot with it.