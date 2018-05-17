---
layout: post
title: How Kubernetes Ingress and LoadBalancer resources work together
redirect_from: "/blog/2018/05/how-kubernetes-ingress-and-loadbalancer-resources-work-together/"
---

In a previous post, I showed you how to create a reverse proxy container image for use in a Docker Swarm. If you're using Kubernetes, you will still use a similar service for ingress (in that you use something like an nginx reverse proxy) but the nomenclature and how it glues together is a bit different. I'll go into some specifics with some example resources and how they work in Azure AKS.

To perform ingress on a Kubernetes cluster, you must deploy both an Ingress resource and an ingress controller. The reason I'm writing this post is because when I was getting started, I had a difficult time understanding why ingress requires both resources. The Kubernetes documentation on ingress goes into just about every detail except for how they interact with each other. Time to dive deeper -- I'll help you understand how this works.

Here's the repository I've been working from for reference and further instructions: https://github.com/brbarnett/much-todo-about-containers

Ingress resource
Simply, the Ingress resource in Kubernetes is a configuration abstraction of how traffic should be directed within a Kubernetes cluster. Its job is to provide a way to version and deploy those rules from source control without getting into the details of - for example - nginx.conf that I did in a previous post. In fact, keep that in mind because what we are setting up is actually similar.

Here's an example of an Ingress resource definition:

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: todo-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: todo-aks-cluster.centralus.cloudapp.azure.com
    http:
      paths:
      - path: /
        backend:
          serviceName: todo-web
          servicePort: 80
      - path: /api
        backend:
          serviceName: todo-api
          servicePort: 80
Pay special attention to the kubernetes.io/ingress.class annotation, as it's required to make Ingress work with the ingress controller.

Ingress controller
An ingress controller (e.g., nginx-ingress) is basically just a fancy reverse proxy that does a bit more extra magic. The ingress controller is deployed as a LoadBalancer service type, which under a standard configuration can expose itself publicly on a static IP over common ports 80 and 443. Even more impressively, deploying a ingress controller automatically deploys an Azure Load Balancer resource in the same resource group as the cluster nodes. The Load Balancer can now route traffic from the static IP into the cluster, balancing requests between all nodes.

Since ingress controllers don't need to be configured directly, this is a great opportunity to use the Helm package manager to deploy a standard ingress controller. More on how to do this in a future post.

But wait, how does the ingress controller actually route traffic? That was my question, too.

The secret sauce
After seemingly endless Googling, I found a post that helped nudge me in the right direction. I can't find it right now but if I come across it again, I'll update the post to give the author credit. The post reminded me of what Kubernetes' ingress-nginx documentation states, which is that the ingress controller's job is to monitor the Kuberentes apiserver' /ingress updates. They also mentioned that the controller is running a daemon that automatically updates nginx's configuration when /ingress updates.

Curious, I checked it out. After deploying an ingress controller, I found the deployed pod and executed interactive bash on the container:

kubectl get pods --namespace=kube-system
kubectl exec -it nginx-ingress-controller-8467dcb994-76bfz bash --namespace=kube-system
Once inside, I poked around a bit and eventually found the nginx.conf file here:

cat /etc/nginx/nginx.conf
This was my aha moment -- the configuration it printed out has some of my Ingress rules built directly into the nginx.conf config file.

So basically, the flow looks something like this:

Kubernetes ingress

The ingress controller's daemon listens to the apiserver's /ingress updates. On update, it automatically generates its nginx.conf file. From that point on, it's effectively a reverse proxy, routing traffic via DNS inside the cluster.

Makes more sense! Next post will be about how to actually deploy these resources to an AKS cluster.