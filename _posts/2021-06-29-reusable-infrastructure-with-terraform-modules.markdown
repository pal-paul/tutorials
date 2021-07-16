---
layout: post
title: "Reusable infrastructure with Terraform modules"
date: 2021-06-29 12:12:12 +0200
categories: Terraform
permalink: /:title/
---

# Terraform module

A Terraform module is simple set of Terraform configuration files in a folder is called a module.
Modules are containers for multiple resources that are used together. A module consists of a collection of terraform files .tf
and/or .tf.json files kept together in a directory.

Modules are the main way to package and reuse resource configurations with Terraform.

[More Information](https://www.terraform.io/docs/language/modules/index.html)

## How to define module?

A module can be define using below syntax:

```
module "NAME" {
  source = "SOURCE"

  [CONFIG ...]
}
```

Within the module definition, the source parameter specifies the folder where the module’s code can be found.
For example, you can create a new file in `services/compute_instance/main.tf` and use the `compute_instance` module in it as follows:

```
provider "google" {
    credentials = file("gcp-account.json")
    project     = "gcp-terraform-307119"
    region      = "europe-west4"
    zone        = "europe-west4-a"
}

module "compute_instance" {
  source = "modules/services/compute_instance"
}
```

Now `compute_instance` module can be reuse the exact same way in the different environment by creating main.tf for each environment.

Every module should have its own `variables.tf` to use it inside it and it can be access out side of the module by `module.<MODULE_NAME>.<OUTPUT_NAME>`

Lets over an example by creating a module `pubsub-topic`

with files `main.tf` , `variables.tf` and `outputs.tf`

main.tf will be as follows:

```
resource "google_pubsub_topic" "topic" {
  name = var.name
  labels = {
    env       = var.env
    component = var.component
  }
}
```

outputs.tf will be as follows:

```
variable "name" {
}
variable "env" {
}
variable "component" {
}
```

variables.tf will be as follows:

```
output "id" {
  value = google_pubsub_topic.topic.id
}
```
