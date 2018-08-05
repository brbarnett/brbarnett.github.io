---
layout: post
title: Securely enabling az aks get-credentials
tags: azure aks kubernetes
---

![_config.yml]({{ site.baseurl }}/images/2018-8-4-securely-enabling-az-aks-get-credentials/aks.jpg)

Over the past months, I have been digging into Kubernetes pretty hard. Since most of my clients are using Azure, AKS is currently the managed Kubernetes service of choice in my world. Since it has gone GA in June and now supports RBAC, I have been looking to provision a Kubernetes playground for the broader organization. 

Following the [Azure AD integration docs](https://docs.microsoft.com/en-us/azure/aks/aad-integration) has been a breeze, but it left me one step short when it comes to using the Azure CLI to get an AKS cluster credentials. In order for `az aks get-credentials` to succeed right now on AKS, the user has to have at least `Contributor` access to the cluster. The `Reader` role yields the following error message:

`The client '[janedoe@domain.com]janedoe@domain.com' with object id '00000000-0000-0000-0000-000000000000' does not have authorization to perform action 'Microsoft.ContainerService/managedClusters/accessProfiles/listCredential/action' over scope '/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/Resource-Group-Name/providers/Microsoft.ContainerService/managedClusters/AKS-Cluster-Name/accessProfiles/clusterUser'.`

But when you think about it, even the `Reader` role is too much access. There's no reason that a cluster user needs to even view the cluster resource in their Azure portal. What we need is a specialized role to both get around the error above as well as create the minimal access required to get cluster credentials until RBAC kicks in.

The solution for us was to create a custom Azure role with minimal access. This is my `AKSClusterConfigurationReader.json` role configuration file:
```
{
   "Name":"AKS Cluster Configuration Reader",
   "Id":"{{ create a unique guid }}",
   "IsCustom":true,
   "Description":"Can get AKS configuration.",
   "Actions":[
      "Microsoft.ContainerService/managedClusters/accessProfiles/listCredential/action",
      "Microsoft.ContainerService/managedClusters/listClusterUserCredential/action"
   ],
   "NotActions":[
   ],
   "DataActions":[
   ],
   "NotDataActions":[
   ],
   "AssignableScopes":[
      "/subscriptions/{{ your subscription ID }}"
   ]
}
```

To create the role, just run the following command: `az role definition create --role-definition /path/to/AKSClusterConfigurationReader.json`

Now, giving people in your organization access to this role on your cluster allows them to minimally be able to get cluster credentials, providing cluster access for `kubectl` without too much Azure portal access.