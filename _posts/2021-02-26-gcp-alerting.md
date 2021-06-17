---
layout: single
title: "Getting alerts from Google CloudOps using Terraform"
categories: technology
tags: monitoring alerting gcp
classes: wide
---

> I've always found that when getting things working quickly is required that using ClickOps is great. However, when you need to repeat something over and over - as was the case at work recently - nothing beats Infrastructure as Code. In this post I'm going to demonstrate how to setup Slack and Email alerting for Google Cloud Platform CloudOps (formerly Stackdriver) using Terraform.
{: .notice--success}

## Create a Slack App Bot

Instructions on how to create a Slack App Bot are available [here](https://slack.com/intl/en-au/help/articles/115005265703-Create-a-bot-for-your-workspace). The creation of the Bot is not overly complex but you will need to know which permissions to grant the bot and to also copy the Bot's API key into GCP Secret Manager

Here's an example of **the API Key** we will need to copy.

![full](/assets/images/bot_api_key.png)
{: .full}

Next we need to set the right permissions for our Alerting Bot

![full](/assets/images/bot_permissions.png)
{: .full}

## Setup Variables

We need to define the variables we plan to use in our Terraform files. These are the ones I've used throughout this post. Feel free to use whatever suits your needs.

{% highlight hcl %}

variable "project_id" {
  description = "Project ID."
  type        = string
}

variable "project_name" {
  description = "Project Name."
  type        = string
}

variable "location" {
  description = "GCP Region."
  type        = string
  default     = "australia-southeast1"
}

variable "email_address" {
  description = "Email Address for sending alerts."
  type        = string
}

variable "slack_channel" {
  description = "Slack Channel for sending alerts."
  type        = string
  default     = "#something-is-broken"
}

{% endhighlight %}

## Use Secret Manager to store the Slack API Key created earlier

Let's assume you've already enabled the Secret Manager API in your GCP project. If not, you can ClickOps your way through just fine using [these instructions](https://cloud.google.com/secret-manager/docs/configuring-secret-manager). After enabling Secret Manager you will need to create a Secret.

>Something to note here is that a Secret is essentially a bucket in which you place Versions which are what actually contain the actual data we want to keep secret. So note in the next step we're not placing any actual API key data in here at this stage just creating a place to keep it.
{: .notice--success}

{% highlight hcl %}

resource "google_secret_manager_secret" "api_key" {
  secret_id = "api-key"
  project   = var.project_id

  replication {
    user_managed {
      replicas {
        location = var.location
      }
    }
  }
}

{% endhighlight %}

## Copy Secret Data

>Sometimes there's just no escaping a little bit of the **copy + paste** we're so used too.
{: .notice--success}

1. Go to the [Secret Manager page in the Cloud Console](https://console.cloud.google.com/security/secret-manager).
2. On the Secret Manager page, Under **Actions** click the **View More** and select **Add new version.**
3. In the **Add new version** dialog, in the **Secret value** field, enter the copied API key.
4. Click the **Add new version** button.

You can always use the **gcloud** command-line tool but will need to copy your key to a txt file first.

{% highlight bash %}

gcloud secrets versions add api-key --data-file="/path/to/file.txt"

{% endhighlight %}

## Retrieve the Slack API Key from Secrets Manager

Now we've got a Secret with a Version ready for use we now need to be able to extract the key we copied there during the Notification Channel creation process.

{% highlight hcl %}

data "google_secret_manager_secret_version" "secret" {
  depends_on = [
    google_secret_manager_secret.api_key,
  ]

  project = var.project_id
  secret  = "${var.project_name}-api-key"
}

{% endhighlight %}

## Create Notification Channels

Here I'm creating Notification Channels for Slack and Email, but there's other options available.

> This is what stumped me for the longest time **sensitive_labels** needs to extra the data of the secret not the secret. You'll find examples on how to get this setup but they all seem to use secret in their example and not **secret_data**. Make sure you use **secret_data** as that's the actual API key you copied earlier. Secret will just be the resource's name - which is pretty useless.
{: .notice--success}

{% highlight hcl %}

resource "google_monitoring_notification_channel" "slack_channel" {
  project      = var.project_id
  display_name = "Slack Notification Channel"
  type         = "slack"
  labels = {
    channel_name = var.slack_channel
  }
  sensitive_labels {
    auth_token = data.google_secret_manager_secret_version.secret.secret_data
  }

  depends_on = [
    google_secret_manager_secret.api_key,
    data.google_secret_manager_secret_version.secret,
  ]
}

resource "google_monitoring_notification_channel" "email_channel" {
  project      = var.project_id
  display_name = "Email Notification Channel"
  type         = "email"
  labels = {
    email_address = var.email_address
  }
}

{% endhighlight %}

> Did you use **secret_data** and not just secret?
{: .notice--success}

## Create policies on which to alert on

This is another bit of a pain as the Terraform resource attributes don't necessarily match those of the Google console

Here's an example of the creating Terraform Monitoring Alert Policies for [Early Boot Validation](https://cloud.google.com/compute/docs/instances/integrity-monitoring)

{% highlight hcl %}

resource "google_monitoring_alert_policy" "early_boot_validation" {
  display_name = "GKE Node Boot Validation"
  combiner     = "OR"
  conditions {
    display_name = "Early Boot Validation for failed by label.status [SUM]"
    condition_threshold {
      filter          = "metric.type=\"compute.googleapis.com/instance/integrity/early_boot_validation_status\" resource.type=\"gce_instance\" metric.label.\"status\"=\"failed\""
      threshold_value = "0"
      duration        = "60s"
      comparison      = "COMPARISON_GT"
      aggregations {
        per_series_aligner = "ALIGN_SUM"
        group_by_fields = [
          "metric.label.status",
        ]
        alignment_period = "300s"
      }
    }
  }
  conditions {
    display_name = "Late Boot Validation for failed by label.status [SUM]"
    condition_threshold {
      filter          = "metric.type=\"compute.googleapis.com/instance/integrity/late_boot_validation_status\" resource.type=\"gce_instance\" metric.label.\"status\"=\"failed\""
      threshold_value = "0"
      duration        = "60s"
      comparison      = "COMPARISON_GT"
      aggregations {
        per_series_aligner = "ALIGN_SUM"
        group_by_fields = [
          "metric.label.status",
        ]
        alignment_period = "300s"
      }
    }
  }

  notification_channels = [
    google_monitoring_notification_channel.slack_channel,
    google_monitoring_notification_channel.email_channel,
  ]

  documentation {
    content   = "Node has failed to start. Investigate the bootdisk encryption and CMEK keys"
    mime_type = "text/markdown"
  }

  enabled = true

}

{% endhighlight %}

You should now have a Slack and Email Notification Bot which will notify you when Alert Policies are triggered within the Google Cloud Platform.

## Helpful stuff

Here's links to the Terraform documentation on:

- [Creating Notification Channels](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/monitoring_notification_channel)
- [Retrieving Secret Manager Versions](https://registry.terraform.io/providers/hashicorp/google/latest/docs/data-sources/secret_manager_secret_version)
- [Creating Monitoring Alert Policies](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/monitoring_alert_policy)
