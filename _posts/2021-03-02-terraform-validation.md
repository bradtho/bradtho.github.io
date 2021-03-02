---
layout: single
title: "Validation of Terraform Custom Variables using Regex"
categories: infrastructure-provisioning terraform
classes: wide
---

> Sometimes you need to be able to validate that what is being entered into code is actually correct. Either because of compliance needs or simply because you can't handle spelling mistakes.
{: .notice--success}

## A Simple Validation

This validation checks that the value entered for location is exactly **australia-southeast1**

>Using **\b** at the start and at the end of each word we want to match we create a word boundary.
{: .notice--success}
>Note too that you also have to escape the **"'s** to ensure that Terraform renders them correctly.
{: .notice--success}

{% highlight hcl %}

variable "location" {
  description = "Resource Region."
  type        = string

  validation {
    condition     = (regex("\\baustralia-southeast1\\b")
    error_message = "Resource Region must only be set to \"australia-southeast1\"."
  }
}

{% endhighlight %}

## A Slightly more complex validation

In this example we loop through a map of labels to validate that each has been correctly set.

{% highlight hcl %}

variable "labels_and_metadata" {
  description = "Labels and Metadata"
  type = map(object({
    labels   = map(string)
    metadata = map(string)
  }))
  default = {
    ingress = {
      labels   = {}
      metadata = {}
    }
    apps = {
      labels   = {}
      metadata = {}
    }
  ## This first validation only checks that the variable contains a particular value e.g. big-value1 would be a valid variable
  validation {
    condition     = can([for v in var.labels_and_metadata : contains(["value1", "value1"], v.metadata.key)])
    error_message = "The key must contain either \"value1\" or \"value2\"."
  }
  ## This second validation is more strict ensuring that the variable exactly matches either low, medium or high. If we were to use the contains function then "blow" or "I like to get high" would both be valid variables.
  validation {
    condition     = can([for v in var.labels_and_metadata : regex("\\blow\\b|\\bmedium\\b|\\bhigh\\b", v.labels.speed)])
    error_message = "Speed must be either 'low', 'medium' or 'high'."
  }

{% endhighlight %}

Once you wrap your head around needing to escape the regex and ensure it's passed in as a single string it's not so bad and is definitely a great way to put in some quick validation checks without needing to use a tool such as Conftest.

## Helpful stuff

Here's links to the Terraform documentation on:

- [Regex](https://www.terraform.io/docs/language/functions/regex.html)
