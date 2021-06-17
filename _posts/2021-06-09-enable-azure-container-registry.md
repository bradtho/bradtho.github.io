---
layout: single
title: "Enabling Azure Container Registry for GitHub Actions"
categories: technology
tags: azure containers
classes: wide
---

> When writing my post about [setting up CICD Integration between GitHub and Azure Security Center](/technology/github-and-azure-security/) I quickly realised that one of the major components was setting up Azure Container Registry (ACR). I really didn't want to bog down the reader with a full how-to of setting up programmatic access between GitHub Actions and ACR and felt it would detract from the main purpose of showing how Container Scanning works. So here's a side-post on how to get ACR up and running with a Service Principal that you can use with GitHub Actions.
{: .notice--success}

### Create an Azure Container Registry

Setting up ACR is a fairly trivial process. Create a resource group and stick ACR into it. However, as with most things that are simple to install, the integration can be a little bit tedious as you'll see as we move through the steps. Also, [here's](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-best-practices) some design considerations for ACR - definitely worth a read if you're wanting to use ACR in production.

{% highlight bash %}

## Create a Resource Group to keep ACR in

$az group create --name bradlab-acr-rg --location australiasoutheast

{
  "id": "/subscriptions/{subscriptionId}/resourceGroups/bradlab-acr-rg",
  "location": "australiasoutheast",
  "managedBy": null,
  "name": "bradlab-acr-rg",
  "properties": {
    "provisioningState": "Succeeded"
  },
  "tags": null,
  "type": "Microsoft.Resources/resourceGroups"
}

## Create an Azure Container Registry - ensure that the name is unique

$az acr create --resource-group bradlab-acr-rg --name bradlab --sku Basic

{
  "adminUserEnabled": false,
  "anonymousPullEnabled": false,
  "creationDate": "2021-06-01T02:35:18.102251+00:00",
  "dataEndpointEnabled": false,
  "dataEndpointHostNames": [],
  "encryption": {
    "keyVaultProperties": null,
    "status": "disabled"
  },
  "id": "/subscriptions/{subscriptionId}/resourceGroups/bradlab-acr-rg/providers/Microsoft.ContainerRegistry/registries/bradlab",
  "identity": null,
  "location": "australiasoutheast",
  "loginServer": "bradlab.azurecr.io",
  "name": "bradlab",
  "networkRuleBypassOptions": "AzureServices",
  "networkRuleSet": null,
  "policies": {
    "quarantinePolicy": {
      "status": "disabled"
    },
    "retentionPolicy": {
      "days": 7,
      "lastUpdatedTime": "2021-06-01T02:35:21.198813+00:00",
      "status": "disabled"
    },
    "trustPolicy": {
      "status": "disabled",
      "type": "Notary"
    }
  },
  "privateEndpointConnections": [],
  "provisioningState": "Succeeded",
  "publicNetworkAccess": "Enabled",
  "resourceGroup": "bradlab-acr-rg",
  "sku": {
    "name": "Basic",
    "tier": "Basic"
  },
  "status": null,
  "systemData": {
    "createdAt": "2021-06-01T02:35:18.102251+00:00",
    "createdBy": "brad.thomas@azenix.com.au",
    "createdByType": "User",
    "lastModifiedAt": "2021-06-01T02:35:18.102251+00:00",
    "lastModifiedBy": "brad.thomas@azenix.com.au",
    "lastModifiedByType": "User"
  },
  "tags": {},
  "type": "Microsoft.ContainerRegistry/registries",
  "zoneRedundancy": "Disabled"
}

{% endhighlight %}

The above is the full output of the ``az acr create`` command which contains a bunch of metadata, networking and policy information in addition to the basic information we'll need going forward. It's also the same output as the ``az acr list --resource-group {resourceGroup} --output json`` command so in this case where we just need to take a note of the **loginServer** and Registry **name** from the output. We can run the following command (keeping in mind that because it's a ``list`` we'll need to treat the output as an array)

{% highlight bash %}

## Using tsv strips the JSON quotes

$az acr list --resource-group bradlab-acr-rg --query '[[].loginServer, [].name]' --output tsv
bradlab.azurecr.io
bradlab

{% endhighlight %}

## Programmatic access to ACR

How do we access our new Azure Container Registry when we will be using GitHub Actions to run our pipeline? Normally we'd just be able to run ``az acr login --name {registry-name}`` but I doubt anyone wants to be storing their Azure Credentials (even as encrypted secrets) anywhere other than their favour password manager.

Enter Service Principals - [here's](https://docs.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals) an explanation of how they work and when to use them. We're going to create a Service Principal for our GitHub Actions and then give it permission to login to our Azure Container Registry.

{% highlight bash %}

## You'll need to enter in your {subscriptionId} and the name of the {resourceGroup} but if you're like me and forever forgetting your {subscriptionId} run this command first

$az account show --query id --output tsv

$az ad sp create-for-rbac --appId "bradlab-acr-sp" --role contributor --scopes /subscriptions/{subscriptionId}/resourceGroups/bradlab-acr-rg --sdk-auth

## Copy the output of the command in to a file which will look similar to this - specifically we'll need the {clientId} and {clientSecret}

{
  "clientId": "{clientId}",
  "clientSecret": "{clientSecret}",
  "subscriptionId": "{subscriptionId}",
  "tenantId": "{tenantId}",
  "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
  "resourceManagerEndpointUrl": "https://management.azure.com/",
  "activeDirectoryGraphResourceId": "https://graph.windows.net/",
  "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
  "galleryEndpointUrl": "https://gallery.azure.com/",
  "managementEndpointUrl": "https://management.core.windows.net/"
}

## If you ever lose or forget the Service Principals credentials you can easily reset them

$az ad sp credential reset --name azenix-acr-sp

## Now we need to give our Service Principal the ability to Push Images to ACR

$az role assignment create --assignee {clientId} --scope /subscriptions/{subscriptionId}/resourceGroups/bradlab-acr-rg/providers/Microsoft.ContainerRegistry/registries/bradlab --role acrpush

{% endhighlight %}

>A compelling reason to keep your ACR instance in its own Resource Group is to keep Service Principal access limited to the scope of the Resource Group that it's in.
{: .notice--success}

## Create Encrypted Secrets in GitHub for Service Principal

To set up the authenticated integration between GitHub and Azure Container Registry we need to set our Service Principal {clientId} and {clientSecret} as Encrypted Secrets into GitHub.

The [GitHub CLI](https://github.com/cli/cli) provides a simple method of [setting your secrets](https://cli.github.com/manual/gh_secret_set). I've done it the lazy way just *copy+pasting* into the command prompt but it's possible to use Environment Variables or [File Redirection](https://www.gnu.org/software/bash/manual/html_node/Redirections.html)

{% highlight bash %}

## Set the Client ID

$ gh secret set AZ_SP_CLIENT_ID --repo bradtho/myapplication
? Paste your secret ***********************************************************
✓ Set secret AZ_SP_CLIENT_ID for bradtho/myapplication

## Set the Client Secret

$ gh secret set AZ_SP_CLIENT_SECRET --repo bradtho/myapplication
? Paste your secret ***********************************************************
✓ Set secret AZ_SP_CLIENT_SECRET for bradtho/myapplication

{% endhighlight %}

Here we can see them present in GitHub web console under **Settings > Secrets**

[![image-center](/assets/images/acr_secrets.png)](/assets/images/acr_secrets.png){: .align-center}

## Setup a GitHub Actions Pipeline

Now that the Service Principal credentials are now available for use as GitHub secrets we can now use them to push Docker Images into the ACR we created earlier.

> If you're not familiar with GitHub Actions then please go checkout the [quickstart guide](https://docs.github.com/en/actions/quickstart)
{: .notice--success}

The [docker/login-action](https://github.com/docker/login-action#azure-container-registry-acr) project on the GitHub Marketpalce has a code block that can be simply setup inside your pipeline similar to the following.

{% highlight yaml %}

name: demo
on: push

env:
  REGISTRY_NAME: bradlab
  APP_NAME: myapplication

jobs:
  demo:
    runs-on: ubuntu-latest

    steps:
    - name: Check out code 
      uses: actions/checkout@main

    - name: Login to Azure Container Registry
      uses: azure/docker-login@v1
      with:
        login-server: ${ env.REGISTRY_NAME }.azurecr.io
        username: ${ secrets.AZ_SP_CLIENT_ID }
        password: ${ secrets.AZ_SP_CLIENT_SECRET }

    - name: Build the Docker image
      run: docker build . -t ${ env.REGISTRY_NAME }.azurecr.io/${ env.APP_NAME }:${ github.sha }

    - name: Push Image to Docker
      run: docker push ${ env.REGISTRY_NAME }.azurecr.io/${ env.APP_NAME }:${ github.sha }

{% endhighlight %}

We can see that this Workflow has passed :white_check_mark: and the Docker Image is now in ACR. Unfortunately I didn't tag these properly so one is the short hash and the other is the full hash but you'll see that they're the same image.

[![image-center](/assets/images/acr_workflow.png)](/assets/images/acr_workflow.png){: .align-center}

[![image-center](/assets/images/acr_images.png)](/assets/images/acr_images.png){: .align-center}

Alternatively you can use the ``gh cli`` and ``az cli`` to perform these validations.

>Now that we're all set with Azure Container Registry and can Build and Push your Docker Images using GitHub Actions it's a good idea to checkout how you can scan those images for vulnerabilities. You can see a demonstration on how to achieve this in [this post](/technology/enable-azure-container-registry.md)
{: .notice--success}
