---
title: Terraform in Proxmox VE
date: 2026-06-11 17:30:00 +0300
categories: IaC
tags: proxmox sops age terraform                  # Tag names should always be lowercase
image:
  path: /assets/img/headers/terraform-proxmox.webp
  lqip: data:image/webp;base64,UklGRtoAAABXRUJQVlA4IM4AAAAQBQCdASoUAA0APpE4l0eloyIhMAgAsBIJZgCdFQAmksiENIl/QAWO/2YPrlNH/94AAP60EIgIrMd7RXnf9tFKhOQflCZ2c+eWqeSTHOdqabDF2HVy3NJ8cz6md52Tkn4xjMn29MUb+7y1eNch6TnEu2gZ6BDjNWg6lWT7fcuUVVTYNJ8f165Dl7f2EO6kNxh2lkGjL1QidMaM9r414x7MM5i3G9+36u4ve/19i/TJj9ionP9/wkk7+QqMux1OZrAJzASwYVLQ+8Lp1ZAAAA==

---

# Terraform provisioning of VM instances in Proxmox VE

Transitioning a homelab from manual "Click-Ops" to a fully automated workflow is a massive step forward. My goal was to create a repeatable, secure, and modular way to deploy virtual machines on Proxmox VE.

In this post, I’ll walk through the architecture, the setup process, and the technical hurdles I overcame to build a professional-grade IaC (Infrastructure as Code) pipeline.

# 📄 The Strategy: Packer + Terraform

The core of this setup follows a two-stage blueprint:

- **The Gold Image (Packer)**: Instead of manually installing Ubuntu, I use Packer to build a "Golden Template." This includes pre-installed dependencies (like QEMU agents) and `Cloud-Init configurations`.
- **The Deployment (Terraform)**: Using the Telmate Proxmox provider, Terraform clones these templates to create a specific infrastructure. For example Ubuntu VMs that will be used later on as a k3s cluster or standalone VMs for testing.

# 🛠️ Prerequisites

Before starting, ensure you have the following:
- Proxmox VE Server: With an API Token created (e.g., terraform-prov@pve).
- Linux Environment: (WSL2 on Windows works perfectly).
- Git: For version control.

# 📥 Installation & Initial Setup

### 1. Install Terraform
```bash
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update && sudo apt-get install terraform
```

### 2. Install SOPS & age (Secrets Management)
SOPS handles the encryption, while age provides the modern, simple key format.
```bash
# Install SOPS
sudo wget https://github.com/getsops/sops/releases/download/v3.8.1/sops-v3.8.1.linux.amd64 -O /usr/local/bin/sops
sudo chmod +x /usr/local/bin/sops

# Install age
sudo apt install age
```

### 3. Generate your Encryption Key
```bash
mkdir -p ~/.age
age-keygen -o ~/.age/age.agekey
```
<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> Make sure to keep the public key (starting with `age1...`) handy for the encryption step.
{: .prompt-info }
<!-- markdownlint-restore -->

# 🔐 The Security Strategy: SOPS + age

Storing API tokens or passwords in plain-text .tfvars files is a major security risk. To solve this, I implemented an encrypted workflow:
1. **Create Secrets**: I created a `secrets.json` containing all my Proxmox credentials.
2. **Encrypt**: I used SOPS to transform that into an encrypted `secrets.enc.json`.
3. **Bridge with Locals**: To keep the resource code clean, I used a `locals.tf` file to map these decrypted values.
```hcl
# Example: Using locals to handle decrypted data types
locals {
  pm_api_url = data.sops_file.my_secrets.data["PROXMOX_URL"]
  vm_id      = tonumber(data.sops_file.my_secrets.data["VM_ID"]) # Convert string to number
}
```

# 🧩 Modularity: Demo vs. Production

Instead of one giant file, I split the project:
`demo-resources.tf`: For testing new Packer templates or create temporary testing infrastructure.
`k3s-resources.tf`: For the live cluster, using the count meta-argument to scale masters and workers dynamically.
```hcl
resource "proxmox_vm_qemu" "k3s_master" {
  count = 3
  name  = "k3s-master-0${count.index + 1}"
  vmid  = local.vm_id + count.index
  # ... specs ...
}
```

# 🚧 Challenges & Lessons Learned

### Data Type Friction
The SOPS provider treats all decrypted values as strings. Since Proxmox expects integers for `vmid` or `memory`, I had to explicitly use the `tonumber()` function in Terraform.

### Provider Discovery
Terraform by default looks for an official Hashicorp SOPS provider. Since it's a community provider, I had to explicitly define the source as `carlpett/sops` in the `required_providers` block.

# 🏆 Final Result

The result is a workflow where I can update an Ubuntu template in Packer and roll out an entirely new infrastructure using terraform.

```bash
# to check what resources will be created
terraform plan
```

```bash
# to create/update those resources 
terraform apply
```

# Repository

You can find the full source code in my GitHub repository:

👉 [Link to your GitHub Repo Here](https://github.com/alexgrajdan/homelab-infra/tree/main/proxmox/terraform)

Stay tuned for more! 😎