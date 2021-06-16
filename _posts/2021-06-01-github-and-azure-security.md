---
layout: single
title: "Observing GitHub Actions with Azure Security Center"
categories: technology
tags: azure security github
classes: wide
---

> One of the more personally interesting demonstrations at Microsoft Build this year was the **Kickstart collaborative DevSecOps practices with GitHub and Azure** the tech was still in preview at the time of the presentation but thankfully not when I wrote this post. The demonstration at Build was very ClickOps oriented so here I'll endeavour to ensure that my version will be using the terminal and/or config-as-code. I also felt that - due to time constraints - that a lot of the pre-requisite work was glossed over during the presentation so I'll attempt to include as much as is sensible in this post.
{: .notice--success}
> UPDATE: So, I tried to obtain the **Authentication Token** and **Connection String** you'll see below via the CLI however it doesn't look like they've made changes to the API so neither az cli or PowerShell worked :sob:
{: .notice--danger}
> I'm a real fan of GitHub Actions, after having had worked on Jenkins, Harness and Bamboo I've got to say it's my favourite CI/CD tool to-date. The simplicity it offers in terms of just getting stuff done is amazing and now with some of the new security features being introduced I hope that more organisations will take it up as their tool of choice too.
{: .notice--success}

## Prerequisites

Obviously please do go checkout the original post available [here](https://techcommunity.microsoft.com/t5/azure-developer-community-blog/kickstart-collaborative-devsecops-practices-with-github-and/ba-p/2357730) especially if my CLI steps aren't making much sense.

I'm going to assume you already have an Azure Subscription to use - [free account](https://azure.microsoft.com/en-gb/free/). This will give you Azure Security Center from which we'll be switching on the Azure Defender.

> If you don't have an Azure Container Registry (ACR) [here's a post](/technology/enable-azure-container-registry.md) on how to do that and get it setup with GitHub Actions to push container images up to a private registry.
{: .notice--success}

I'm also going to write ~~all~~ ~~most~~ as many as I can of the commands using **azure CLI** and **github CLI**. I may one day write a port for **Powershell** but I'm sure if you're using Powershell you'll already know what to do :wink:

You'll need to switch on Azure Defender to get access to some of the settings - this is a **paid service** based upon what you're "defending" so if you don't want to pay-to-play just ensure you don't enable the plan on any running resources and switch it off after the 30-day trial runs out. More info is available [here](https://docs.microsoft.com/en-gb/azure/security-center/security-center-pricing?WT.mc_id=Portal-Microsoft_Azure_Security)

I'm just going to be switching on Azure Defender for Azure Container Registry but you can enable it for a whole swag of different services: VirtualMachines, SqlServers, AppServices, StorageAccounts, SqlServerVirtualMachines, KubernetesService, ContainerRegistry, KeyVaults, Dns, Arm, OpenSourceRelationalDatabases

{% highlight bash %}

## Log into Azure

$ az login

## Enable Azure Defender for Azure Container Registry

$ az security pricing create --name ContainerRegistry --tier standard
{
  "freeTrialRemainingTime": "16 days, 21:56:00",
  "id": "/subscriptions/{subscriptionId}/providers/Microsoft.Security/pricings/ContainerRegistry",
  "name": "ContainerRegistry",
  "pricingTier": "Standard",
  "type": "Microsoft.Security/pricings"
}

{% endhighlight %}

## Obtain Connection Strings

![image-right](/assets/images/asc_settings.png){: .align-right}

Once you've switched on Azure Defender for Container Registries navigate into the **Azure Portal > Security Center > Settings > Integrations**

Alternatively, if you know your **subscriptionId** then you can drop this into your browser window. Replacing the **{subscriptionId}** with your own.

{% highlight html %}

https://portal.azure.com/#blade/Microsoft_Azure_Security/PolicyMenuBlade/threatDetection/subscriptionId/{subscriptionId}/pricingTier/0/defaultId/

{% endhighlight %}

Once on the integrations page you'll want to click the **Configure CI/CD integration** button.

![image-center](/assets/images/asc_integrations.png){: .align-center}

![image-left](/assets/images/asc_cicd.png){: .align-left} As of now there's only two regions where you can set your default workspace. Choosing one or the other will update your **Connection String**. The **Authentication Token** is generated for you based upon your **subscriptionId**.

You'll need to copy both the **Connection String** and the **Authentication Token** as we'll be saving them as [Encrypted Secrets in GitHub](https://docs.github.com/en/actions/reference/encrypted-secrets)

If only we could've just pulled this data using the Azure CLI.

## Create Encrypted Secrets in GitHub for Azure Security Center

To set up the authenticated integration between GitHub and Azure Security Center we need to set our authentication tokens as Encrypted Secrets into GitHub.

The [GitHub CLI](https://github.com/cli/cli) provides a simple method of [setting your secrets](https://cli.github.com/manual/gh_secret_set). I've done it the lazy way just *copy+pasting* but it's possible to use Environment Variables or [File Redirection](https://www.gnu.org/software/bash/manual/html_node/Redirections.html)

{% highlight bash %}

## Set the Authentication Token

$ gh secret set AZ_SUBSCRIPTION_TOKEN --repo bradtho/myapplication
? Paste your secret ***********************************************************
✓ Set secret AZ_SUBSCRIPTION_TOKEN for bradtho/myapplication

## Set the Connection String

$ gh secret set AZ_APPINSIGHTS_CONNECTION_STRING --repo bradtho/myapplication
? Paste your secret ***********************************************************
✓ Set secret AZ_APPINSIGHTS_CONNECTION_STRING for bradtho/myapplication

{% endhighlight %}

Here we can see them present in GitHub web console under **Settings > Secrets**

![image-center](/assets/images/asc_secrets.png){: .align-center}

## Consuming Secrets using GitHub Actions

We're getting closer to the good stuff now. Here I'll be updating an existing GitHub Actions workflow which pushes **myapplication** into Azure Container Registry.
