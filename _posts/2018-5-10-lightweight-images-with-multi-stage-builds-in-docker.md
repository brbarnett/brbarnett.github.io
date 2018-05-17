---
layout: post
title: Lightweight images with multi-stage builds in Docker
redirect_from: "/blog/2018/05/lightweight-images-with-multi-stage-builds-in-docker/"
---

Size matters. When you're shipping code in containers, it's important to remember that your container images are pulled on every node within your Swarm or Kubernetes clusters. This means that as you scale out your cluster nodes or schedule workloads on new nodes in your cluster, the size of your image becomes non-trivial for cold start time.

It's a best practice to use containers to resolve dependencies (dotnet restore, npm install) and build (dotnet build, webpack) code because it ensures a consistent environment where the team's specific version of Node.js becomes less important. Unfortunately, this also requires that the containers in which we build code must also have build dependencies installed, such as the .NET Core SDK or Node.js. Since you don't need all those dependencies in production, how can you slim down your images?

### Multi-stage builds
Multi-stage builds allow you to restore and build code in a container with a more robust base image, but then copy the resulting app to a slimmer image that only has the bare minimum. Here's an example Dockerfile:

```
# Starting with a feature-rich base image for build
FROM node:8.11.1 as build-env
WORKDIR /src
COPY . .
RUN npm install

# Switching to a slimmer base image for production
FROM alpine:latest

# Adding node
RUN apk add --no-cache nodejs
WORKDIR /app

# Copy artifacts from build container to slimmer container image
COPY --from=build-env /src .

CMD node app.js
```

I tried building the image from the first 4 lines, then from the entire Dockerfile. Here's the difference in image size between stages:

![_config.yml]({{ site.baseurl }}/images/2018-5-10-lightweight-images-with-multi-stage-builds-in-docker/multi-stage-images.jpg)

That is a substantial size difference between the two images, mostly because the `node:8.11.1` base image is 672MB while the `alpine:latest` image is only 4.15MB. It's always a good idea to use the slimmest responsible base image for production workloads.

Note: for those using .NET Core, Microsoft has a build base image with the SDK (`microsoft/aspnetcore-build`) and a slimmed down base image with just the runtime (`microsoft/aspnetcore`).