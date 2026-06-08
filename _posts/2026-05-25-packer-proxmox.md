---
title: Packer in Proxmox VE
date: 2026-05-25 17:30:00 +0300
categories: iac
tags: proxmox sops age packer                  # Tag names should always be lowercase
image:
  path: 
  lqip: 
  # lqip = Low-Quality Image Placeholder
  # This is used for faster loading times
  # install imagemagick
  # convert assets/img/headers/hello-homelab.webp -resize 20x20 -quality 20 -strip assets/img/headers/hello-homelab-lqip.webp
  # base64 -w 0 assets/img/headers/hello-homelab-lqip.webp > lqip.txt
---

# Building Templates in Proxmox VE using Packer

Automating VM templates is something that comes in mind of any Homelab or DevOps enthusiast. Recently, I managed to get a final version of this automation using Packer in Proxmox

What started as a single Packer script evolved into a modular, secure pipeline using **SOPS** and **age**. Here is the story of that journey, the hurdles I hit, and the final architecture.

---

## 🛠️ Prerequisites & Installation

Before diving into the journey, you'll need a few tools in your belt. Here is how to get set up on a Linux-based system (like Ubuntu or WSL):

### 1. HashiCorp Packer
Packer is the engine that builds the VM.
```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install packer
```

### 2. SOPS (Secrets Operations)
SOPS is used to encrypt our secrets file.
```bash
# Download the latest binary from GitHub
Q_VERSION=$(curl -s https://api.github.com/repos/getsops/sops/releases/latest | grep -Po '"tag_name": "v\K[0-9.]+')
wget "https://github.com/getsops/sops/releases/download/v${Q_VERSION}/sops-v${Q_VERSION}.linux.amd64" -O sops
chmod +x sops
sudo mv sops /usr/local/bin/
```

### 3. Age - File Encryption
Age is the modern encryption backend that SOPS uses to manage keys.
```bash
sudo apt install age
# Generate your master key (Save this somewhere safe!)
mkdir -p ~/.sops
age-keygen -o ~/.sops/age.agekey
```

### 4. Create API Configuration (The "Least Privilege" Setup)
Before Packer can talk to Proxmox, we need to give it a specific identity and the right set of "keys." Instead of using the root account, we’ll create a dedicated role and user.

#### Create a dedicated Role for Packer
You can do this in two ways, via Proxmox VE GUI or console.

On GUI:
- Go to Datacenter -> Permissions -> Roles -> Create
- Specify the name
- Add the necessary Privilages: `VM.Allocate VM.Config.Disk VM.Config.CPU VM.Config.Memory VM.Config.Network VM.Config.HWType VM.Config.Options VM.Monitor VM.Audit VM.PowerMgmt Datastore.AllocateSpace Datastore.Audit Sys.Audit`

On CLI:
- Go to shell
- Type the following:
```bash
pveum role add PackerRole -privs "VM.Allocate VM.Config.Disk VM.Config.CPU VM.Config.Memory VM.Config.Network VM.Config.HWType VM.Config.Options VM.Monitor VM.Audit VM.PowerMgmt Datastore.AllocateSpace Datastore.Audit Sys.Audit"
```
#### Create a Packer user
on GUI:
- Go to Datacenter -> Permissions -> Users -> Add
- Type the name, e.g. `packer`

on CLI:
- Go to shell
- Type the following:
```bash
pveum user add packer@pam
```

#### Generate API token
API tokens are better than passwords because they can be revoked easily.

On GUI:
- Go to Datacenter -> Permissions -> API Tokens -> Add
- Select the user created early
- Type the same name on ID field, e.g. `packer`
- **Uncheck** Privilage Separation
- Add a comment if you want or expiration date
- Hit the `Add` button
<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> **Important**: Copy the `value` shown after this, it won't be shown again! Place it somewhere safe!
{: .prompt-tip }
<!-- markdownlint-restore -->

On CLI:
- Go to shell
- Type the following:
```bash
pveum user token add packer@pam packer-token --privsep 0
```
<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> **Important**: Copy the `value` shown after this, it won't be shown again! Place it somewhere safe!
{: .prompt-tip }
<!-- markdownlint-restore -->

#### Bind the Role to the user
Now, we tell Proxmox that our new user (and its token) has the permissions defined in PackerRole across the entire cluster (/).

On GUI:
- Go to Datacenter -> Permissions -> Groups -> Create
- Name the group, e.g. `packer`
- Go to Datacenter -> Permissions -> Users
- Go to packer user -> Edit
- Assign `packer` group

On CLI:
- Go to shell
- Type the following:
```bash
pveum acl modify / -user packer@pam -role PackerRole
pveum acl modify / -token 'packer@pam!packer' -role PackerRole
```


## The Starting Point: The "Monolith" Problem

Initially, everything was in one giant `ubuntu-2404.pkr.hcl` file. It worked, but it was messy. My Proxmox API tokens, SSH passwords, and hardware specs were all mashed together. 

**The Goal:**
1. **Modularize:** Treat Packer like Terraform (split variables, sources, and build blocks).
2. **Secure:** Use SOPS (Secrets Operation) to encrypt sensitive data so I can safely commit my code to GitHub.

---

## Challenge #1: Modularizing HCL2

Packer HCL2 allows you to split files, but it behaves slightly differently than Terraform. 

**The Fix:** 
I split the monolith into:
- `packer.pkr.hcl` (Plugins)
- `variables.pkr.hcl` (Definitions)
- `source.pkr.hcl` (Hardware/ISO)
- `build.pkr.hcl` (Provisioning scripts)

**Key Takeaway:** When you split files, you can no longer run `packer build file.pkr.hcl`. You must run `packer build .` to tell Packer to merge all files in the current directory.

---

## Challenge #2: The SOPS & Age Learning Curve

I wanted to use **SOPS** with **age** keys for encryption. This is where the real "debugging journey" began.

### The "Malformed Key" Mystery
I kept getting `malformed recipient` errors. 
**The Lesson:** Age keys are checksummed. A single extra space or a missing character during a copy-paste into `.sops.yaml` will break everything. I learned to use `age-keygen -y` to derive the public key directly from the private key to ensure accuracy.

### The "Invalid Character 'P'" Error
When I tried to encrypt my secrets file, SOPS crashed with: `Could not unmarshal input data: invalid character 'P'`.
**The Lesson:** I was trying to use HCL syntax inside a `.json` file. SOPS is a *structured* encryption tool. If the file extension is `.json`, the content **must** be valid JSON. Once I converted my variables to a JSON object, SOPS worked like a charm. 😄

---

## Challenge #3: The Packer Pipe Problem

In a perfect world, I wanted to decrypt secrets directly into memory using process substitution:
`packer build -var-file=<(sops -d secrets.enc.json) .`

**The Problem:** Packer refused to read it, complaining: `Could not guess format of /dev/fd/63`. Packer requires a file extension (`.json` or `.hcl`) to know how to parse variables.

**The Solution: The "Trap" Strategy.**
I wrote a `build.sh` wrapper that:
1. Decrypts the secrets to a temporary `tmp_secrets.json` file.
2. Uses a shell `trap` to ensure that even if the build fails or I hit `Ctrl+C`, the temporary plaintext file is **immediately deleted**.

```bash
# The magic shell trap
trap 'rm -f tmp_secrets.json' EXIT
sops -d secrets.enc.json > tmp_secrets.json
packer build -var-file=tmp_secrets.json .
```

# Key Takeaways for Your Own Journey
1. JSON for Secrets, HCL for Definitions
Keep your variable definitions in HCL so you can write comments and documentation. Keep your values in JSON because SOPS handles it better, allowing you to see which "keys" changed in Git while keeping the "values" encrypted.
2. The Power of **`.sops.yaml`**
Don't pass your public keys in the command line. Create a `.sops.yaml` file. It allows you to run a simple `sops -e -i secrets.json` without remembering long strings of characters.
3. Cleanup is Part of the Build
A template is only good if it's "clean." My build process now includes a shell provisioner that:
- Truncates `/etc/machine-id`
- Cleans `cloud-init`
- Removes SSH host keys. This ensures that every VM I clone from this template gets a unique identity.

# Repository

You can find the full source code, including the modular HCL files and the build script, in my GitHub repository:

👉 [Link to your GitHub Repo Here](https://github.com/alexgrajdan/homelab-infra/tree/main/proxmox/packer)

# Conclusion
Moving from a single plaintext file to a modular, encrypted pipeline felt like "overkill" at first, but the peace of mind is worth it. I can now push my Homelab configuration to GitHub without fear, knowing my Proxmox credentials are safe behind an Age key.

Tools used:
- Proxmox (Hypervisor)
- Packer (Image Automation)
- SOPS (Encryption)
- Age (Key Management)

Next:

I will be using Terraform to deploy VMs using this template created with Packer.