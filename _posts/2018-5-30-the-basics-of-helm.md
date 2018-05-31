---
layout: post
title: The basics of Helm
tags: helm kubernetes docker
---

### What is Helm?
Helm calls itself the package manager for Kubernetes. That's true, but I think I can help simplify: Helm provides a way to package multiple Kubernetes resource templates that can be released, upgraded and rolled back together. It allows you to define a much larger and cohesive atomic deployment unit as a collection of multiple resources and dependencies.

### Packaging Kubernetes resources with Helm
Helm requires a specific filesystem structure in order to work correctly. As of the time of writing, the following is the structure of a Helm chart:
```
chart-name/
  Chart.yaml          # A YAML file containing information about the chart
  LICENSE             # OPTIONAL: A plain text file containing the license for the chart
  README.md           # OPTIONAL: A human-readable README file
  requirements.yaml   # OPTIONAL: A YAML file listing dependencies for the chart
  values.yaml         # The default configuration values for this chart
  charts/             # A directory containing any charts upon which this chart depends.
  templates/          # A directory of templates that, when combined with values,
                      # will generate valid Kubernetes manifest files.
  templates/NOTES.txt # OPTIONAL: A plain text file containing short usage notes
```

Let's use a true example - this post is based on the following GitHub repository, specifically this Helm chart:
[https://github.com/brbarnett/much-todo-about-containers/tree/master/helm-charts/src/todo-app](https://github.com/brbarnett/much-todo-about-containers/tree/master/helm-charts/src/todo-app)

There is a collection of Kubernetes resource manifest files in the `./templates` directory that will all get deployed at the same time as a release. 

Before we get started, make sure `kubectl` is pointing at the correct Kubernetes instance with `kubectl config current-context`, then initialize Helm:

```
helm init --upgrade --service-account default   # required, otherwise you get a 'no available release name' error

helm repo update
```

### Creating a new release
When creating a new Helm release in Kubernetes, you can pass in either a known chart name (e.g., `stable/nginx-ingress`) or a local chart directory. Since I'm just testing this from my local machine on an AKS cluster, I'll use the directory here. 

```
helm install ./helm-charts/src/todo-app \
    --name todo-release \
    --set web.replicas=6
```

What's with the `--set web.replicas=6`? Helm templates are not exactly Kubernetes resource manifests, but rather templates for them. It allows you to pass in configurable variables that dynamically render in final manifests prior to release. In this case, I have a value in my `values.yaml` file that the `--set` keyword is overriding.

### Upgrading a release

```
helm upgrade todo-release ./helm-charts/src/todo-app
```

### Rolling back a release

```


helm history todo-release

helm rollback todo-release 1
```

[ image of history ]