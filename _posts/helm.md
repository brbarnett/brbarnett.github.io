---
layout: post
title: The basics of Helm
tags: helm kubernetes docker
---

This post is based on the following GitHub repository, specifically this Helm chart:
[https://github.com/brbarnett/much-todo-about-containers/tree/master/helm-charts/src/todo-app](https://github.com/brbarnett/much-todo-about-containers/tree/master/helm-charts/src/todo-app)

### What is Helm?
Helm calls itself the package manager for Kubernetes. That's true, but I think I can help simplify: Helm provides a way to package multiple Kubernetes resource templates that can be released, upgraded and rolled back together. It allows you to define a much larger atomic deployment unit as a collection of multiple resources and dependency that make sense to deploy together.

### Packaging Kubernetes resources with Helm


### Creating a new release

```
helm install ./helm-charts/src/todo-app --name todo-release
```

### Upgrading a release

```
helm upgrade todo-release ./helm-charts/src/todo-app
```

### Rolling back a release

```


helm history todo-release

helm rollback todo-release 1
```