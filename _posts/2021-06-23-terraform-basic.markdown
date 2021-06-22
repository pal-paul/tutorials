---
layout: post
title: "Terraform Basic"
date: 2021-06-22 13:44:45 +0200
categories: Terraform
permalink: /:title/
---

## What is Terraform?

Terraform is an open source infrastructure as code (IaC) software tool that allows DevOps engineers to programmatically provision the physical resources an application requires to run.

### Pre-requisite

Before we start working with Terraform, here are the pre-requisites -

- You must install terraform [click here on how to install terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)
- You must have either AWS or (Google Cloud)[https://cloud.google.com/] service account key

### Prepare terraform file

Now we are going to write our first Terrafrom script.
Create a file named `main.tf`

### Add provider details

we are going to provision the VMInstance on Google, so we need to select the provider as google.

```
provider "google" {
     credentials = file("gcp-account.json")
     project     = "gcp-terraform-307119"
     region      = "europe-west4"
     zone        = "europe-west4-a"
}

```

**gcp-account.json** - is the JSON key file of a sevice account that we going to use to provision the VMInstance on Google

**project** - is the google project id

**region && zone** - is the region and zone which you need to mention to deploy the VMInstance

### Add resource

Since we aim to create a VMInstance on Google Cloud Platform, so we need to add `google_compute_instance` configuration.

```
resource "google_compute_instance" "default" {
  name         = var.instance_name
  machine_type = var.instance_type

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-9"
    }
  }

  network_interface {
    network = "default"

    access_config {
      // Ephemeral IP
    }
  }
}

```

**name** - You can keep the name of your virtual instance as per your choice

**machine_type** - I have chosen e2-micro but you can choose from - e2-small, e2-medium, e2-standard-2 etc.

**boot_disk** - Here you need to mention the host OS and I have opted for - debian-cloud/debian-9

**network_interface** - This configuration is needed for getting the IP address of the virtual machine.

Now create `variable.tf` but without any default values

```
variable "instance_type" {
}
variable "instance_name" {
}
```

There can be situation where you need create multiple tfvars files based on the environment like stage, production.

create your `stage.tfvars` for staging

```
instance_type="t2.micro"
instance_name ="stage_vm"
```

create your `prod.tfvars` for production

```
instance_type="t2.micro"
instance_name ="prod_vm"
```

Finally files structure will look like -

```
/root/
    main.tf
    variable.tf
    stage.tfvars
    prod.tfvars
```

## Terraform

Now we have completed all the pre-requisites needed for provisioning the VMInstance on Google Cloud.

The first command which we are going to run is -

```
$ terraform init
```

This command is going to download all the required dependencies based on the provider name mentioned in the main.tf. In the current example the provider name is Google so it is going to install Google’s terraform dependencies onto your machine

The second command which we are going to run is -

```
$ terraform plan -var-file="stage.tfvars"
```

This command is tell us how many resources are going to be created/destroyed or change for a specific environment (tfvars)
_Note - terraform plan never creates the VM on google plan, it just tells you what it is going to perform._

The final command which we are going to run is terraform apply.

This command is going to install/setup the virtual machine on Google Cloud.

```
$ terraform apply -var-file="stage.tfvars"
```

After running the above command you should see as per the plan it will created/change/destroyed the resources for a specific environment (tfvars)

## Terraform setting variable using command line

```
provider "google" {
     credentials = file("gcp-account.json")
     project     = "gcp-terraform-307119"
     region      = "europe-west4"
     zone        = "europe-west4-a"
}

resource "google_compute_instance" "default" {
  name         = var.instance_name
  machine_type = var.instance_type

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-9"
    }
  }

  network_interface {
    network = "default"

    access_config {
      // Ephemeral IP
    }
  }
}

variable "instance_type" {
}

variable "instance_name" {
}

```

To specify individual variables on the command line, use the -var option when running the terraform plan and terraform apply commands:

```
$ terraform init
$ terraform plan -var="instance_type=t2.micro" -var="instance_name:prod_vm"
$ terraform apply -var="instance_type=t2.micro" -var="instance_name:prod_vm"
```

If we need to destroy the environment using following command -

```
$ terraform destroy -var="instance_type=t2.micro" -var="instance_name:prod_vm"
```
