---
layout: post
title: Including custom configuration files in .NET Core 2.0 Preview 1
redirect_from: "/blog/2017/06/including-custom-configuration-files-in-net-core-20-preview-1/"
---

I'm doing some modernizing of legacy apps and decided to use it as an opportunity to play with some new technologies -- in this case, we're talking about .NET Core 2.0 Preview 1. While building a reference architecture to talk about on Github, I ran into a snag while trying to figure out how to hide my Azure AD credentials. I want to include a config section, but don't want to commit it with the rest of my appsettings.json file.

Reference Github repository: https://github.com/brbarnett/dot-net-core-spa

When I created my new application, the Program.cs file contained a minimal web host configuration:

        public static IWebHost BuildWebHost(string[] args) =>
            WebHost.CreateDefaultBuilder(args)
                .UseStartup()
                .Build();
This automatically wires up the default appsettings.json and the environment-specific versions (e.g., appsettings.Development.json) in a less verbose way. The problem here is that I actually do want to tell the application how to wire up my custom config instead of just using the default(s). My project requires an appsettings.Authentication.json file which specifically just includes my Azure AD credentials. Here's how I did it:

        public static IWebHost BuildWebHost(string[] args)
        {
            IConfigurationRoot config = new ConfigurationBuilder()
                .SetBasePath(Directory.GetCurrentDirectory())
                .AddJsonFile("appsettings.Authentication.json", optional: false)  // add custom config file
                .Build();

            return WebHost.CreateDefaultBuilder(args)
                .UseConfiguration(config)  // include reference to config
                .UseStartup()
                .Build();
        }
Then I included an appsettings.Authentication.json.example file in my repository so that anyone using it could have an example to work from to create the now-required appsettings.Authentication.json file. Don't forget if you do this to use a .gitignore to specifically exclude your custom configuration files.