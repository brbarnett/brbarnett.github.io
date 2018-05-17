---
layout: post
title: Creating portable development environments with Docker
redirect_from: "/blog/2018/03/creating-portable-development-environments-with-docker/"
tags: docker
---

The rate of change in software development tools has never been faster, which has allowed developers to continue to demand increased flexibility in how they deliver. Gone are the days of an entire development team sharing consistent hardware (PC vs Mac), and I have seen a massive uptick in once-Windows-only devs moving to MacOS. Enabling this change are the framework giants like Microsoft, who have been moving toward OS-agnostic frameworks - for example, .NET Core running on Windows, MacOS and Linux.

This flexibility comes with a cost: when your job is to develop and/or maintain multiple projects, you are now required to manage specific OS and framework versions in order to promote consistency within the broader development team. Can you imagine having to upgrade/downgrade Node or dotnet versions between projects?

Enter: [Docker](https://www.docker.com/). Docker allows you to manage OS and framework versions within images, and those images run in containers that are completely independent of each other and the host OS. What that means for you is that you can run a Node 6.x container next to a Node 8.10.0 (current LTS) without needing to even install Node on your host. I'm going to show you not only how to load your code into containers that support specific frameworks, but also how to connect those containers to your host's file system to allow hot reload -- meaning, whenever you update the code on your file system, the container automatically restarts with the latest version. How cool is that??

### Prerequisites
I am going to assume that you have a grasp on the fundamentals of containers, have at least explored Docker and understand some of Docker's basic CLI commands. If you're interested in learning more about Docker, I highly recommend going through a few of [Nigel Poulton](https://twitter.com/nigelpoulton)'s Pluralsight courses: http://blog.nigelpoulton.com/training-videos/

At the time of writing, I'm using Docker version 17.12.0-ce on my Mac (though the platform doesn't matter). Please [install Docker](https://docs.docker.com/install/#supported-platforms) if you would like to follow along!

### The demo
This demo includes both Node.js and ASP.NET Core projects which we can run side-by-side, and I've hooked them both up to some file watchers to allow hot reload. The base repository/branch and Docker Compose file for this demo can be found here: https://github.com/brbarnett/hello-docker/blob/mount-hot-reload/docker-compose.yml

I've done a few important things here:

* I specifically wrote my Node.js and ASP.NET Core apps so that they're not aware of Docker. This means that you can still take these apps and deploy them to your favorite cloud host's PaaS offerings - you are **NOT** required to use Docker in higher-level environments
* Note lines 6 and 13 in the referenced `docker-compose.yml` file - this is how I got hot reload to work using [nodemon](https://github.com/remy/nodemon) and [dotnet watch](https://github.com/aspnet/Docs/blob/master/aspnetcore/tutorials/dotnet-watch.md)
* Note lines 7 and 14 as well - these are references from the host's file system that map/mount to the containers' `./app` directories. Your containers will actually read and monitor files from the host's file system to run your applications, but will still run them within the frameworks installed on the containers

The purpose of this demo is to show you that this development environment is not only portable, but it doesn't require you to install any dependencies on your actual host OS! Let's check it out.

### Creating a clean Linux development VM
To show you how this works, I'm going to spin up a clean Linux VM on Azure. You don't have to go through these instructions, but I did want to show you the declarative deployment because I'm trying to make the point that I'm not installing anything special on the VM other than Docker and Docker Compose. In this case, I'm using Azure CLI to deploy the VM.

Usually I would suggest that you install Azure CLI to your local OS, but remember: Docker allows us to run containers with dependencies installed that don't have to exist on the host. Running Azure CLI in a container is a perfect example of this at work. Here's how to do it:

```
docker run -it microsoft/azure-cli  # note: -it flag means Interactive Mode
```

Your Docker daemon just downloaded the `microsoft/azure-cli` image, and you're running its container in interactive mode. Keep in mind that this doesn't install Azure CLI to your host's OS, but you still have access to it within a thin Linux OS. Cool!

Here are the instructions to spin up that Linux VM on Azure:

```
az login
az account set --subscription 00000000-0000-0000-0000-000000000000 # your subscription ID
az group create --name hello-docker --location northcentralus
az network vnet create \
    --resource-group hello-docker \
    --name hello-docker-vnet \
    --address-prefix 192.168.0.0/16 \
    --subnet-name hello-docker-subnet \
    --subnet-prefix 192.168.1.0/24
az network public-ip create --resource-group hello-docker --name hello-docker-ip --dns-name hello-docker-dns
az network nsg create --resource-group hello-docker --name hello-docker-nsg
az network nsg rule create \
    --resource-group hello-docker \
    --nsg-name hello-docker-nsg \
    --name hello-docker-nsgrule-ssh \
    --protocol tcp \
    --priority 1000 \
    --destination-port-range 22 \
    --access allow
az network nsg rule create \
    --resource-group hello-docker \
    --nsg-name hello-docker-nsg \
    --name hello-docker-nsgrule-web \
    --protocol tcp \
    --priority 1001 \
    --destination-port-range 8080 8081 \
    --access allow
az network nic create \
    --resource-group hello-docker \
    --name hello-docker-nic \
    --vnet-name hello-docker-vnet \
    --subnet hello-docker-subnet \
    --public-ip-address hello-docker-ip \
    --network-security-group hello-docker-nsg
az vm availability-set create --resource-group hello-docker --name hello-docker-as
az vm create \
    --resource-group hello-docker \
    --name hello-docker-linux \
    --location northcentralus \
    --availability-set hello-docker-as \
    --nics hello-docker-nic \
    --image UbuntuLTS \
    --admin-username azureuser \
    --generate-ssh-keys
```

Now we have a clean Ubuntu-based VM available for us to SSH into and install/run Docker containers.

### Running our portable development environment
Now that we have this clean VM, let's install Docker and Docker Compose:

```
ssh azureuser@hello-docker-dns.northcentralus.cloudapp.azure.com

curl -fsSL get.docker.com -o get-docker.sh
sudo sh get-docker.sh
docker version # make sure it's installed properly

sudo curl -L https://github.com/docker/compose/releases/download/1.20.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version # make sure it's installed properly
```

Now that Docker is up and running, it's time to pull down some code so that we can play with composing and hot reloading the applications:

```
git clone https://github.com/brbarnett/hello-docker
cd hello-docker
git checkout mount-hot-reload
sudo docker-compose up -d
```

It will take a few minutes once you run `docker-compose up` because Docker is downloading images from DockerHub in order to run them as the base images for your two containers. Check out your running containers by running the `docker ps` command. Also, you're now able to see the two services exposed at the following URLs:

http://hello-docker-dns.northcentralus.cloudapp.azure.com:8080/
http://hello-docker-dns.northcentralus.cloudapp.azure.com:8081/api/values

Important note, and I can't make this point enough: notice that I didn't install Node.js or dotnet on the Linux VM. This is what makes the environment portable, in that the environment is totally self-contained and independent of your host.

Also don't forget I said we'd use hot reload on these containers to update them. Usually you would pull up your project(s) in your favorite IDE (e.g., Visual Studio Code), but to simulate that here go ahead and run one of the following commands in your terminal to edit either the api's server.js file, or the data-service's `ValuesController.cs` (default controller):

```
vim api/server.js
vim data-service/Controllers/ValuesController.cs
```

Change Hello World or the values array to another value and save the file. Once you do, give it a moment and hit the corresponding URL -- the container watched you update the code and has now restarted with an updated app! It's both the volumes references in the `docker-compose.yml` and `nodemon`/`dotnet watch` that enable this functionality. 

### Why this is important
As you are pursuing environment continuity across your development team, consider that using Docker will help you ensure that everyone's running exactly the same code on exactly the same infrastructure/networks, and that you don't have to worry about dependencies getting in the way. I don't want my developers to care about installing a specific version of SQL Server or Node.js, and this is especially true for those on our team who maintain code long-term because over time the drift in versions can create infrastructure hell.

Also consider that if your company tends to write the same types of apps over and over, creating your own base images in DockerHub will help standardize your development process by pre-installing any dependencies you use every time.

There is a lot more to consider in order to host Docker containers in production (e.g., multi-stage builds and Docker Swarm or Kubernetes orchestrators), but so far I think we've set you up to at least make sure you can start to standardize your development environments across different hardware and OSs.