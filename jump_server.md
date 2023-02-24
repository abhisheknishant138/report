---
# User change
title: "Deploy Arm instances on GCP and provide access via Jump Server"

weight: 4 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

## Before you begin

Any computer which has the required tools installed can be used for this section. 

You will need a [Google Cloud account](https://console.cloud.google.com/). Create an account if needed. 

Three tools are required on the computer you are using. Follow the links to install the required tools.
* [Terraform](/install-tools/terraform)
* [Google Cloud CLI](/install-tools/gcloud)

## Deploy Arm instances on GCP and provide access via Jump Server

## Introduction to Jump Server
A Jump Server (also known as bastion host) is an intermediary device responsible for funnelling traffic through firewalls using a supervised secure channel. By creating a barrier between networks, jump servers create an added layer of security against outsiders wanting to maliciously access sensitive company data. Only those with the right credentials can log into a jump server and obtain authorization to proceed to a different security zone.

## Generate key-pair(public key, private key) using ssh keygen

Before using Terraform, first generate the key-pair (public key, private key) using `ssh-keygen`. To generate the key-pair, follow this [documentation](/learning-paths/server-and-cloud/aws/terraform#generate-key-pairpublic-key-private-key-using-ssh-keygen).

## Deploying Arm instances on GCP and providing access via Jump Server
For deploying Arm instances on GCP and providing access via Jump Server, the Terraform configuration is broken into 4 files: main.tf, outputs.tf, variables.tf, and terraform.tfvars.

**main.tf** creates an instance with OS Login configured to use as a bastion host and a private instance to use alongside the bastion host.
```console
terraform {
  required_version = ">= 0.12.26"
}

# Create a Management Network for shared services
module "management_network" {
  source = "./modules/vpc-network"
  project     = var.project
  region      = var.region
}

# Create an instance with OS Login configured to use as a bastion host
resource "google_compute_instance" "bastion_host" {
  project      = var.project
  name         = "bastion-vm"
  machine_type = "t2a-standard-1"
  zone         = var.zone
  tags = ["public"]
  boot_disk {
    initialize_params {
      image = "ubuntu-os-cloud/ubuntu-2204-lts-arm64"
    }
  }
  network_interface {
    subnetwork = module.management_network.public_subnetwork
    // If var.static_ip is set use that IP, otherwise this will generate an ephemeral IP
    access_config {
      nat_ip = var.static_ip
    }
  }
  metadata = {
    enable-oslogin = "TRUE"
  }
}
# Create a private instance to use alongside the bastion host.
resource "google_compute_instance" "private" {
  project = var.project
  name         = "bastion-private"
  machine_type = "t2a-standard-1"
  zone         = var.zone
  allow_stopping_for_update = true
  tags = ["private"]
  boot_disk {
    initialize_params {
      image = "ubuntu-os-cloud/ubuntu-2204-lts-arm64"
    }
  }
  network_interface {
    subnetwork = module.management_network.private_subnetwork
  }
  metadata = {
    enable-oslogin = "TRUE"
  }
}
```

**outputs.tf** defines the output values for this configuration.
```console
output "public_ip_bastion_host" {
  description = "The public IP of the bastion host."
  value       = google_compute_instance.bastion_host.network_interface[0].access_config[0].nat_ip
}

output "private_ip_instance" {
  description = "Private IP of the private instance"
  value       = google_compute_instance.private.network_interface[0].network_ip
}
```

**variables.tf** describing the variables referenced in the other files with their type and a default value.
```console
variable "project" {
  description = "The name of the GCP Project where all resources will be launched."
  type        = string
}

variable "region" {
  description = "The region in which the VPC netowrk's subnetwork will be created."
  type        = string
}

variable "zone" {
  description = "The zone in which the bastion host VM instance will be launched. Must be within the region."
  type        = string
}

variable "static_ip" {
  description = "A static IP address to attach to the instance. The default will allocate an ephemeral IP"
  type        = string
  default     = null
}
```


**terraform.tfvars** contains actual values of the variables defined in **variables.tf**
```console
project = "your project ID"
region = "us-central1"
zone = "us-central1-a"
```

### Terraform Commands
To deploy the instances, we need to initialize Terraform, generate an execution plan and apply the execution plan to our cloud infrastructure. Follow this [documentation](/content/learning-paths/server-and-cloud/gcp/terraform.md#terraform-commands) to deploy the **main.tf** file.

### Verify the Instance and Bastion Host setup
In the Google Cloud console, go to the [VM instances page](https://console.cloud.google.com/compute/instances?_ga=2.159262650.1220602700.1668410849-523068185.1662463135). The instances we created through Terraform must be displayed in the screen.

![Jump Server](https://user-images.githubusercontent.com/71631645/203951115-5a8f8ac1-e415-4e82-bb3d-aae65c1f3c65.png)
   
### Use Jump Host to access the Private Instance
Connect to a target server via a Jump Host using the `-J` flag from the command line. This tells ssh to make a connection to the jump host and then establish a TCP forwarding to the target server, from there
```console
  ssh -J username@jump-host-IP username@target-server-IP
```
![ssh-j](https://user-images.githubusercontent.com/71631645/203960729-38f353d1-8a4e-4704-b039-04608896d114.jpg)

### Clean up resources
Run `terraform destroy` to delete all resources created.
```console
  terraform destroy
```
It will remove all resource groups, virtual networks, and all other resources created through Terraform.

![tf destroy](https://user-images.githubusercontent.com/71631645/203960620-bc580385-2fd6-477d-93c3-29895eeb5290.jpg)
