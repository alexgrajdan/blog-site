---
title: Terraform in GCP
date: 2026-09-14 21:00:00 +0300
categories: IaC Cloud
tags: gcp automation terraform gcloud                    # Tag names should always be lowercase
image:
  path: /assets/img/headers/terraform-gcp.webp
  lqip: data:image/webp;base64,UklGRo4AAABXRUJQVlA4IIIAAADQAwCdASoUAA0APpE6l0eloyIhMAgAsBIJQBUgAmAFCSgW0uIFCMAA/pFmg4RtHsD16jEddq8oHXR/0kEg2ijOZHxpZ30/fGZLxYIf/x+AfMHgKIob4dS6mHYOJ/x6SoS6xC1PqX916yUaG/lECvY1RBCX3aZz/Shef68UxhX8EAAA
---

# Introduction

Welcome to this quick-start guide on getting up and running with **Google Cloud Platform (GCP)**. Whether you're automating your infrastructure or building out your latest lab environment, this walkthrough will guide you through installing and configuring the **Google Cloud CLI (gcloud)** and using **Terraform** to provision your first cloud resources seamlessly. Let's dive in!

### Step 1: Install and Initialize the Google Cloud CLI

First, ensure you have the `gcloud` CLI installed on your local machine.

1. **Create a new GCP project**: Login to [GCP](https://console.cloud.google.com/) and setup a new project there.

2. **Install gcloud**: Follow the official installation guide for your operating system (macOS, Linux, or Windows) via the  [Google Cloud CLI documentation.](https://cloud.google.com/sdk/docs/install)

2. **Initialize gcloud**: 
```bash
gcloud init
```
- This process will initialize your gcloud cli and will also ask you if you want to sign in with your google account.
- Press `Y` and it will open a browser window for you to sign in with your GCP credentials.
- After that, select the project you just created earlier. 
- (Optional) Setup a default zone/region.

3. **Authenticate Application Default Credentials (ADC)**:
Terraform uses ADC to authenticate API requests to GCP.
```bash
gcloud auth application-default login
```

### Step 2: Install Terraform

If you don't have Terraform installed yet, download it from the [HashiCorp Terraform website](https://developer.hashicorp.com/terraform/install) or use a package manager (like `brew install terraform` on macOS or `choco install terraform` on Windows).

### Step 3: Write Your Terraform Configuration

Create a dedicated directory for your Terraform project, navigate into it, and create the following structure:

```shell
.
├── main.tf
├── terraform.tfvars
└── variables.tf
```

In `main.tf` you can have the following:
```terraform
provider "google" {
  project     = var.project
  region      = var.region
  zone        = var.zone
}

# VM instance in GCP
resource "google_compute_instance" "gcp_instance" {
  count                     = var.instance_count
  name                      = "gcp-instance-${count.index}"
  machine_type              = "e2-standard-2"
  zone                      = var.zone
  allow_stopping_for_update = true

  boot_disk {
    initialize_params {
      image = var.os-image
    }
  }

  network_interface {
    network = "default"
    access_config {
      // Ephemeral public IP
    }
  }

  # Metadata for SSH access
  metadata = {
    ssh-keys = "${var.ssh_user}:${file(var.ssh_public_key_path)}"
  }
}

# Custom VPC network
resource "google_compute_network" "custom_network" {
  name                    = "terraform-network"
  auto_create_subnetworks = false
}

# Custom subnetwork
resource "google_compute_subnetwork" "terraform_subnet" {
  name          = "terraform-subnetwork"
  ip_cidr_range = "<your-range-here>" # e.g "10.10.10.0/24"
  region        = "europe-central2"
  network       = google_compute_network.custom_network.id

}
```

In `terraform.tfvars` you can have something like this:
```terraform
project        = "<YOUR_GCP_PROJECT_ID>" # can be found on GCP web portal
instance_count = <HOW_MANY_INSTANCES_YOU_WANT> # e.g. 3
```
In `variables.tf` there are the variables called earlier:
```terraform
variable "project" {}

variable "region" {
  type    = string
  default = "europe-central2"
}

variable "zone" {
  type    = string
  default = "europe-central2-a"
}

variable "os-image" {
  type    = string
  default = "ubuntu-os-cloud/ubuntu-2604-lts-amd64"
}

variable "instance_count" {
  type    = number
  default = 1
}

variable "ssh_user" {
  type    = string
  default = "ubuntu"
}

variable "ssh_public_key_path" {
  type    = string
  default = "<YOUR_PATH_TO_KEY>" # e.g. "~/.ssh/id_ed25519.pub"
}
```

### Step 4: Initialize and Apply the Terraform Configuration

With your configuration in place, spin up the infrastructure using standard Terraform workflows.

1. **Initialize the working directory:**
This downloads the required Google provider plugins.
```bash
terraform init
```

2. **Preview the execution plan:**
Check what resources Terraform is planning to create.
```bash
terraform plan
```

3. **Apply the configuration:**
Provision the resources in GCP. Type `yes` when prompted to confirm.
```bash
terraform apply
```

4. *(Optional) List created components components*
- for compute instances:
```bash
gcloud compute instances list
```
- for network components:
```bash
gcloud compute networks list
```
- test ssh connection using gcloud
```bash
gcloud compute ssh <YOUR_INSTANCE> --zone=<YOUR_ZONE> --ssh-key-expire-after=2m
```


### Step 5: Clean Up
If you are just testing and want to avoid incurring any charges or cluttering your project, you can easily tear down the infrastructure you just created:
```bash
terraform destroy
```