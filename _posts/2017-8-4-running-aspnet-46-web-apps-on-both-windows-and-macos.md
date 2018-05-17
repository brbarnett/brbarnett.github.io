---
layout: post
title: Running ASP.NET 4.6 web apps on both Windows and macOS
redirect_from: "/blog/2017/08/running-aspnet-46-web-apps-on-both-windows-and-macos/"
---

The .NET world is hot on .NET Core right now, and why not? As the next iteration of the framework, it also allows developers more freedom to use their device and hosting platform of choice. While I use a PC at work, I have a Mac at home and love the idea of collaborating on projects from either device. .NET Core is the natural, easy way to do that.

But wait -- what about legacy applications? Not everyone can upgrade yet, given the expense of refactoring an entire application. This is especially true if you don't expect ROI from a migration.  Good news: Visual Studio for Mac has a solution for you. The debugger uses Mono now, which means you can fire up a framework 4.6 application and debug as if you were on your Windows machine. There is a bit of a snag when creating a project on a PC first, however.

To test it out, I created a GitHub repository and added a new 4.6 web application from both my PC and Mac to compare them. When I pulled the PC project to my Mac, I got a runtime error: Could not find file "(project-path)/bin\roslyn\csc.exe"

As it turns out, there is one small quirk from PC-created projects, and it's based on a section of the web.config:

<system.codedom>
	<compilers>
		<compiler language="c#;cs;csharp" extension=".cs" type="Microsoft.CodeDom.Providers.DotNetCompilerPlatform.CSharpCodeProvider, Microsoft.CodeDom.Providers.DotNetCompilerPlatform, Version=1.0.5.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35" warningLevel="4" compilerOptions="/langversion:default /nowarn:1659;1699;1701" />
		<compiler language="vb;vbs;visualbasic;vbscript" extension=".vb" type="Microsoft.CodeDom.Providers.DotNetCompilerPlatform.VBCodeProvider, Microsoft.CodeDom.Providers.DotNetCompilerPlatform, Version=1.0.5.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35" warningLevel="4" compilerOptions="/langversion:default /nowarn:41008 /define:_MYTYPE=\"Web\" /optionInfer+" />
	</compilers>
</system.codedom>
This concept does not exist on a Mac, so it complains about not being able to find the Roslyn compiler. I removed that section of the web.config, and then I was able to run my 4.6 app on both PC + Mac, brought to you by Mono. The Mac project worked on PC with no issues.