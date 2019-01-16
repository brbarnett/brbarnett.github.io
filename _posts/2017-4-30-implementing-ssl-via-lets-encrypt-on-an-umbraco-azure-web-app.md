---
layout: post
title: Implementing SSL via Let's Encrypt on an Umbraco Azure Web App
redirect_from: "/blog/2017/04/implementing-ssl-via-lets-encrypt-on-an-umbraco-azure-web-app/"
tags: azure tls lets-encrypt paas umbraco
---

![_config.yml]({{ site.baseurl }}/images/2017-4-30-implementing-ssl-via-lets-encrypt-on-an-umbraco-azure-web-app/lets-encrypt-umbraco.jpg)

### Why SSL?
I'm a big believer in SSL, especially lately - privacy on the internet has never been such a divisive topic. Like most people who build and own a blog site like this, I find it hard to stomach spending a few hundred bucks a year to secure a site that doesn't have any forms or collect any kind of information from users. When it comes down to it, though, I love seeing the big green padlock in Chrome from SSL sites, and I'll also take the Google [SEO boost](https://webmasters.googleblog.com/2014/08/https-as-ranking-signal.html) to boot.

<!--more-->

### Let's Encrypt!
This is where [Let's Encrypt](https://letsencrypt.org/) comes in: it's a free, open source certificate authority that allows people like me to implement SSL on my blog site without a hefty bill from Comodo or one of the giants. A typical implementation involves installing Let's Encrypt on a VM and issuing a certificate request from there, but what about Azure PaaS? I don't have control over that by design. Thankfully, there's a site extension for that, and I'm a huge fan of Troy Hunt's blog post / instruction manual on how to get that set up: https://www.troyhunt.com/everything-you-need-to-know-about-loading-a-free-lets-encrypt-certificate-into-an-azure-website/

There is one hiccup, however - when I was trying to set this up, I was getting 404 errors from the Let's Encrypt site extension when it was trying to access created files in the `~/.well-known/acme-challenge/` path. The problem is that Umbraco is hijacking that path and trying to route you to a document and failing to find one. Here are the two quick entries that make this work, all located in your web.config:

```
<configuration>
	...
	<appSettings>
		...
		<add key="umbracoReservedPaths" value="~/umbraco,~/install/,~/.well-known" />
		...
	</appSettings>

	<system.webServer>
		...
		<staticContent>
			<remove fileExtension="." />
			<mimeMap fileExtension="." mimeType="text/plain" />
			...
		</staticContent>
		...
	</system.webServer>
	...
</configuration>
```

Once I got that set up, I was able to select my domain(s) and the site extension figured everything out for me. These web.config changes are the only difference from what Troy describes in his post, though you can add the optional URL rewrite rule to force SSL as follows:

```
<rewrite>
	<rules>
		<clear />
		<rule name="Redirect to https" stopProcessing="true">
			<match url=".*" />
			<conditions>
				<add input="{HTTPS}" pattern="off" ignoreCase="true" />
			</conditions>
			<action type="Redirect" url="https://{HTTP_HOST}{REQUEST_URI}" redirectType="Permanent" appendQueryString="false" />
		</rule>
	</rules>
</rewrite>
```

Happy encrypting!