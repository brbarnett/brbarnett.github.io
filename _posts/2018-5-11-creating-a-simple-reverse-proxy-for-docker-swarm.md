---
layout: post
title: Creating a simple reverse proxy for Docker Swarm
redirect_from: "/blog/2018/05/creating-a-simple-reverse-proxy-for-docker-swarm/"
tags: docker docker-swarm nginx
---

In a microservice architecture, it's common to split an API into multiple, independently-deployable applications. While technically the services that run in front of the application containers can expose themselves directly to the internet via ports, it's best practice to serve traffic traffic from a cluster over ports 80/443 and route internally via DNS. 

To do that, we'll use an nginx service to act as a reverse proxy. It will look something like this:

![_config.yml]({{ site.baseurl }}/images/2018-5-11-creating-a-simple-reverse-proxy-for-docker-swarm/nginx-reverse-proxy.jpg)

Once traffic reached the nginx service, it will be routed to services within the cluster by the DNS name. For example, `~/api/` will be proxied to `http://api:80/`, which is our api service. The following configuration uses regex matching and `proxy_pass` to direct traffic based on the inbound URL:

```
events {
  worker_connections  1024;  # default is 1024, and this section is required
}

http {
  resolver 127.0.0.11 ipv6=off valid=30s; # required for proxy_pass if you append URL bits ($1 below). this is Docker's static DNS IP

  server {
    listen       80;  # the port that the server is listening on. this will give you http only

    location ~* ^/api/(.*)$ {  # match all urls that begin with ~/api/
      proxy_pass      http://todo-api/$1; # resolve api service, append everything that came after ~/api/. This removes the /api/ fragment
    }

    location / {  # everything else passes to the web service
      proxy_pass      http://todo-web;
    }
  }
}
```

In order to get this config into the nginx container image, use this simple Dockerfile:

```
FROM nginx:alpine
COPY nginx.conf /etc/nginx/nginx.conf
```

That is about as thin as it gets - more instructions on how to test this solution out here: https://github.com/brbarnett/much-todo-about-containers

I'll talk more about Kubernetes in a future post as well, where we'll use ingress rules instead of a custom nginx configuration.