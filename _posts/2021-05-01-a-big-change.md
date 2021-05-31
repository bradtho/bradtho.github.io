---
layout: single
title: "A Big Change"
categories: career
classes: wide
---

> I'm definitely not the most prolific of writers so my updates come through in dribs and drabs or whenever I feel like I have something to contribute to the greater community. Well... recently I took up the amazing opportunity to switch brands at my workplace (without actually leaving the company) and focus on starting up a [Microsoft Azure focused brand](https://www.azenix.com.au).
{: .notice--success}

## Coming Full Circle

Oddly enough I took my first steps into my cloud journey back in 2012 with Microsoft Office 365 - it wasn't an amazing experience back then and truth be told, even though it's a much better product now, I still don't like having to work with it. Since then I've worked on and off with various Microsoft virtualisation and cloud products

So why am I throwing my lot back in with Azure after having spent the last two-and-a-half years steering away from Microsoft products?

- Microsoft has changed a **lot** since the Ballmer years where I felt that I was committing a capital crime by using a Linux OS in favour of Windows.

<div style="text-align:center;">
<iframe src="https://giphy.com/embed/l3q2zbskZp2j8wniE" width="480" height="279" frameBorder="0" class="giphy-embed" allowFullScreen></iframe>
</div>

- In 2018 I switched to using MacOS but with support for [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-macos) and [Powershell](https://docs.microsoft.com/en-us/powershell/scripting/install/installing-powershell-core-on-macos?view=powershell-7.1) I don't feel like I'm hamstrung by continuing to use my MacBook. I can even keep on using Bash to write my scripts.
- Whilst the dreaded Azure Resource Manager templates are still a thing they're now wrappered with a DSL called [Bicep](https://github.com/Azure/bicep) which makes creating ARM templates so much easier. I can also keep on using my (limited) Terraform skills to deploy to Azure - so it wasn't a totally waste of life learning it :smile:
- I no longer feel that the concept of Operating System (looking at you Windows Server) is really all that relevant anymore*. In recent years, I've primarily been using varying containerised and serverless systems. Azure provide some really awesome tech within these two areas - [AKS](https://azure.microsoft.com/en-au/services/kubernetes-service/), [Arc](https://azure.microsoft.com/en-au/services/azure-arc/) and [Function Apps](https://azure.microsoft.com/en-au/services/functions/)
- With their acquisition of GitHub (yes, Azure DevOps has been around for a while) I could see that Microsoft were going to continue heading in the right direction of enabling DevOps culture. I love using [GitHub Actions](https://github.com/features/actions)
- Finally, after having had used both AWS and Google Cloud platforms in recent years I felt that Azure provides a far better user experience - at least for me.

*Yup controversial I know, but I honestly can't remember the last time I've actually needed to install an operating system. I'll probably have to do an IaaS lift and shift now :facepalm:

## So What Now

In the next few weeks I'll be updating my [Journey](/about/) to be more Azure centric.

[Microsoft Build](https://bit.ly/3c34FgE) is coming up later in the month so hopefully I'll have something a little bit more exciting than a career update (HINT: It'll have something to do with Serverless).
