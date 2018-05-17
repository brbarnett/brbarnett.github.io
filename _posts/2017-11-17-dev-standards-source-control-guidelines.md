---
layout: post
title: Dev standards&colon; Source control guidelines
redirect_from: "/blog/2017/11/dev-standards-source-control-guidelines/"
tags: vsts git source-control
---

As technology companies continue to grow and project teams expand to deliver larger projects, it is important to reflect and make sure your organization's foundation skills are strong. By no means would I say that learning Git will solve all of your problems, but I think it is fair to say that there are a few basic guidelines out there that keep your projects hovering at least closer to the "pit of success."

These are guidelines that serve to document basic process around how I deliver all technical phases of our projects. While this process serves as the default, there are potentially reasons to change based on specific needs. If you do not follow this process, please be prepared to defend your decision(s) with your leadership.

Here's what works for us:

* **You should rarely change locations without pushing code** commits to a remote source control repository. You never know when your development machine will be unavailable or stop working. This does happen (I can give you examples) and you will lose work if you are not backed up.
* **Use Git for source control.** New to Git? Play the [Learning Git Branching game](https://learngitbranching.js.org/) to get started quickly! I typically use [VSTS](https://www.visualstudio.com/team-services/) on my projects for source control and work management.
* **Use feature branches for development.** Nobody should be committing code directly to master past initial solution setup. Use branch policies (not permissions) on that branch for enforcement. Use this document for guidance.
* **Use pull requests** (commonly called PRs) to merge branches to master. Why pull requests? [Check this out](https://docs.microsoft.com/en-us/vsts/git/concepts/pull-requests-overview)
    * All PRs should be reviewed by at least one other team member regardless who wrote it:
        * Less experienced developers want and need code quality/best practices review and coaching.
        * Senior developers and even Architects should be writing code that those less experienced can understand. Having a less experienced developer review code can also teach them some of your best practices.
    * Make sure the PR builds prior to approving!
    * If you are on a project and nobody is reviewing your code (a common example is if you represent a highly-specialized development skill), let your leadership know -- ask them to assign someone else to review your code. This is important to your growth.
* **Create and monitor automated builds** when pull requests are completed to help ensure a clean, deployable solution.

These are guidelines that work for us, but this is by no means an exhaustive list. What works best for you? Have a favorite? Tweet at me [@brandonbarnett](https://twitter.com/brandonbarnett)