---
layout: single
title: "Observing GitHub Actions with Azure Security Center"
categories: technology
tags: azure security github
classes: wide
---

> One of the more personally interesting demonstrations at Microsoft Build this year was the **Scaling DevSecOps with GitHub and Azure** talk. The tech was still in preview at the time of the presentation (and still is). I found the demonstration at Build was very ClickOps oriented so here I'll endeavour to ensure that my examples will be using the terminal and/or config-as-code. I also felt that - due to time constraints - that a lot of the pre-requisite work was glossed over during the presentation so I'll attempt to include as much as is sensible in this post.
{: .notice--success}
> UPDATE: I tried to obtain the **Authentication Token** and **Connection String** via the CLI however it doesn't look like they've made changes to the API so neither az cli or PowerShell worked :sob:
{: .notice--danger}
> On another note, I'm a real fan of GitHub Actions, after having had worked on Jenkins, Harness and Bamboo I've got to say it's my favourite CI/CD tool to-date. The simplicity it offers in terms of just getting stuff done is amazing and now with some of the new security features being introduced I hope that more organisations will take it up as their tool of choice too.
{: .notice--success}

## Prerequisites

Obviously please do go checkout the original blog post available [here](https://techcommunity.microsoft.com/t5/azure-developer-community-blog/kickstart-collaborative-devsecops-practices-with-github-and/ba-p/2357730) especially if my CLI steps aren't making much sense.

I'm going to assume you already have an Azure Subscription to use - otherwise you can setup a [free account](https://azure.microsoft.com/en-gb/free/). This will give you Azure Security Center from which we'll be switching on the Azure Defender.

> If you don't already have an Azure Container Registry (ACR) setup [here's a post](/technology/enable-azure-container-registry) on how to do that and get it setup with GitHub Actions to push container images up to the registry.
{: .notice--warning}

I'm also going to write ~~all~~ ~~most~~ as many as I can of the commands using ``azure CLI`` and ``github CLI``. I may one day write a port for ``Powershell`` but I'm sure if you're using Powershell you'll already know what to do :wink:

You'll need to switch on **Azure Defender** to get access to some of the settings - this is a **paid service** based upon what you're "defending" so if you don't want to pay-to-play just ensure you don't enable the plan on any running resources and switch it off after the 30-day trial runs out. More info is available [here](https://docs.microsoft.com/en-gb/azure/security-center/security-center-pricing?WT.mc_id=Portal-Microsoft_Azure_Security)

I'm just going to be switching on Azure Defender for Azure Container Registry but you can enable it for a whole swag of different services: ``VirtualMachines, SqlServers, AppServices, StorageAccounts, SqlServerVirtualMachines, KubernetesService, ContainerRegistry, KeyVaults, Dns, Arm, OpenSourceRelationalDatabases``

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

[![image-right](/assets/images/asc_settings.png)](/assets/images/asc_settings.png){: .align-right}

Once you've switched on Azure Defender for Container Registries navigate into the **Azure Portal > Security Center > Settings > Integrations**

Alternatively, if you know your **subscriptionId** then you can drop this into your browser window. Replacing the **{subscriptionId}** with your own.

{% highlight html %}

https://portal.azure.com/#blade/Microsoft_Azure_Security/PolicyMenuBlade/threatDetection/subscriptionId/{subscriptionId}/pricingTier/0/defaultId/

{% endhighlight %}

Once on the integrations page you'll want to click the **Configure CI/CD integration** button.

[![image-center](/assets/images/asc_integrations.png)](/assets/images/asc_integrations.png){: .align-center}

[![image-left](/assets/images/asc_cicd.png)](/assets/images/asc_cicd.png){: .align-left} As of now there's only two regions where you can set your default workspace and choosing one or the other will update your **Connection String**. The **Authentication Token** is generated for you based upon your **subscriptionId**.

You'll need to copy both the **Connection String** and the **Authentication Token** as we'll be saving them as [Encrypted Secrets in GitHub](https://docs.github.com/en/actions/reference/encrypted-secrets)

If only we could've just pulled this data using the Azure CLI.

## Create Encrypted Secrets in GitHub for Azure Security Center

To set up the authenticated integration between GitHub and Azure Security Center we need to save our authentication tokens as Encrypted Secrets into GitHub.

The [GitHub CLI](https://github.com/cli/cli) provides a simple method of [setting your secrets](https://cli.github.com/manual/gh_secret_set). I've done it the lazy way just *copy+pasting* into the command prompt but it's possible to use Environment Variables or [File Redirection](https://www.gnu.org/software/bash/manual/html_node/Redirections.html) if you're wanting to potentially script the process.

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

[![image-center](/assets/images/asc_secrets.png)](/assets/images/asc_secrets.png){: .align-center}

## Consuming Secrets using GitHub Actions

We're getting closer to the good stuff now. Here I'll be updating an existing GitHub Actions workflow which pushes ``myapplication`` into Azure Container Registry. If you don't already have ACR up and running you can check out this [blog post](/technology/enable-azure-container-registry/) on how to do that.

We need to add in two additional steps into our workflow file.

- Scan image for vulnernabilities
- Post scan results to ASC

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
    - name: Checkout code
      uses: actions/checkout@main

    - name: Login to Azure Container Registry
      uses: azure/docker-login@v1
      with:
        login-server: ${ env.REGISTRY_NAME }.azurecr.io
        username: ${ secrets.AZ_SP_CLIENT_ID }
        password: ${ secrets.AZ_SP_CLIENT_SECRET }

    - name: Build the Docker image
      run: docker build . -t ${ env.REGISTRY_NAME }.azurecr.io/${ env.APP_NAME }:${ github.sha }

    - name: Scan local image for vulnerabilities
      uses: Azure/container-scan@v0.1
      id: container_scan
      continue-on-error: true
      with:
        image-name: ${ env.REGISTRY_NAME }.azurecr.io/${ env.APP_NAME }:${ github.sha }

    - name: Push Image to ACR
      run: docker push ${ env.REGISTRY_NAME }.azurecr.io/${ env.APP_NAME }:${ github.sha }

    - name: Post Logs to ASC
      uses: Azure/publish-security-assessments@v0
      with: 
        artifact-type: containerImage 
        scan-results-path: ${ steps.container_scan.outputs.scan-report-path }
        connection-string: ${ secrets.AZ_APPINSIGHTS_CONNECTION_STRING }
        subscription-token: ${ secrets.AZ_SUBSCRIPTION_TOKEN }

{% endhighlight %}

## Viewing the results in GitHub

I deliberately made some tweaks to my Dockerfile to force it to get some vulnerabilities as we can see in the following screenshots.

In your repo navigate to **Actions > {Your Action}** otherwise you can also use the ``github cli``.

{% highlight bash %}

$ gh workflow view --repo bradtho/myapplication
? Select a workflow demo (push.yml)
demo - push.yml
ID: 10484951

Total runs 8
Recent runs
✓  demo                                                demo  master  push  944934764
X  demo                                                demo  master  push  944913273
X  fixing CI error                                     demo  master  push  944898002
X  removing the distroless step and amending workflow  demo  master  push  944893062
✓  updating push workflow for container scanning       demo  master  push  944690742

$ gh run view 944934764 --repo bradtho/myapplication

✓ master demo · 944934764
Triggered via push about 1 hour ago

JOBS
✓ demo in 1m19s (ID 2845203828)

ANNOTATIONS
X Vulnerabilities were detected in the container image
demo: .github#1

{% endhighlight %}

[![image-center](/assets/images/asc_workflow.png)](/assets/images/asc_workflow.png){: .align-center}

The workflow found some vulnerabilities so let's drill into the job to see the events.

{% highlight bash %}

$ gh run view 944934764 --log --repo bradtho/myapplication

## I'm not going to give you the full log but this one entry is probably worthwhile recording

demo    Post scan results to ASC        2021-06-17T03:03:38.7972762Z Posting scan results to Azure AppInsights. Request Id: 6071c717-eb94-4c37-b668-33cd71ccf1d6

{% endhighlight %}

[![image-center](/assets/images/asc_vulnerabilities.png)](/assets/images/asc_vulnerabilities.png){: .align-center}

OK it found a lot of vulnerabilties and this report has been sent to Azure Security Center

[![image-center](/assets/images/asc_report.png)](/assets/images/asc_report.png){: .align-center}

## Viewing the results in Azure Security Center

I suppose this is where things fall down. We've successfully identified Vulnerabilites in our GitHub Actions workflow however as you'll see they're just not showing up in Azure Security Center.

From the Azure Security Center navigate to the Recommendations page and select the ``Vulnerabilities in Azure Container Registry images should be remediated (powered by Qualys)`` option.

[![image-center](/assets/images/asc_recommendations.png)](/assets/images/asc_recommendations.png){: .align-center}

[![image-center](/assets/images/asc_policy.png)](/assets/images/asc_policy.png){: .align-center}

No Vulnerabilities detected :interrobang: OK let's drill into this some more then - select **Affected Resources** and click the **Healthy Registries** tab > ``{your registry}`` > ``{your repository}`` > take a note of the ``{Image Tag}``

[![image-center](/assets/images/asc_healthy_1.png)](/assets/images/asc_healthy_1.png){: .align-center}

[![image-center](/assets/images/asc_healthy_2.png)](/assets/images/asc_healthy_2.png){: .align-center}

[![image-center](/assets/images/asc_healthy_3.png)](/assets/images/asc_healthy_3.png){: .align-center}

So I'm not too sure what's going on here. Is there a difference in policy? Is there a false positive coming from GitHub even with the logs being shipped successfully?

## Troubleshooting

Let's see what Azure is considering as a scanned image.

First test we'll comment out the ``Post Logs to ASC`` step in the workflow to not report the findings.

:white_check_mark: ``Not Scanned``

Second test we'll then uncomment and observe the results.

:white_check_mark: ``Found``

We can see that the reports are correctly being sent to Azure but Azure Security Center isn't considering the image vulnerable - even though there are CVE's being reported in the GitHub Actions output.

[![image-center](/assets/images/asc_troubleshoot_1.png)](/assets/images/asc_troubleshoot_1.png){: .align-center}

I decided to rewatch the [Scaling DevSecOps with GitHub and Azure](https://mybuild.microsoft.com/sessions/87cc3b82-bc57-483d-90b3-e91e12516352?source=sessions) session that's available on demand to see if I'd missed anything - nothing missed. The only thing I could think of was to try using the Tailwind Traders Website repo instead of my own (at least to provide some level of comparison)

### The Results

Lo and behold there's a completely different set of results for this image than with my own.

I get a bunch of vulnerabilities as I did in my own application image.

[![image-center](/assets/images/asc_troubleshoot_2.png)](/assets/images/asc_troubleshoot_2.png){: .align-center}

However, this is what I was expecting to see when I uploaded my image with vulnerabilities to my registry.

[![image-center](/assets/images/asc_troubleshoot_3.png)](/assets/images/asc_troubleshoot_3.png){: .align-center}

We can now drill into the repo to view the details of the vulnerability

[![image-center](/assets/images/asc_troubleshoot_4.png)](/assets/images/asc_troubleshoot_4.png){: .align-center}

and further drill into the image itself to see the findings.

[![image-center](/assets/images/asc_troubleshoot_5.png)](/assets/images/asc_troubleshoot_5.png){: .align-center}

## Conclusion

I definitely feel that there is a long way to go with this Azure Security Center integration (yes, it's in Preview so hopefully it does get some attention). Reporting false negatives within a security aspect is definitely not an ideal scenario. It definitely diminishes the trust you can have in your monitoring tools when they don't report as expected. I'd like to think that what's [documented](https://docs.microsoft.com/en-us/azure/security-center/defender-for-container-registries-cicd) as working actually works as expected all the time.

I think for now I'll rely on the reporting provided by GitHub Actions.
