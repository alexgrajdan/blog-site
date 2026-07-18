---
title: Getting started with Ansible
date: 2026-07-16 22:00:00 +0300
categories: IaC
tags: ansible automation                     # Tag names should always be lowercase
image:
  path: /assets/img/headers/ansible-tutorial.webp
  lqip: data:image/webp;base64,UklGRpIAAABXRUJQVlA4IIYAAAAwBACdASoUAAwAPpE4l0eloyIhMAgAsBIJZgCw+BHQMVvRK6E4UsazdAAA/veB0tyTOI+wKwJLSXou3KQTkVtkP0ITHJmEW91eNaAN6DG7+t/uX2iBMjZRupxTYltrHfnCiwEmXpjjuQZ37O4qczr1/uf9UTsvHN+wQpw2sR0EwchBcAAAAA==
---

# Getting started with Ansible

## What is Ansible?

Ansible is an *open-source IT automation engine* that automates configuration management, application deployment, cloud provisioning, and multi-node orchestration. Unlike many traditional automation tools, Ansible is **agentless**, meaning it does not require you to install any background software on the servers you wish to manage. Instead, it securely connects to target machines via standard protocols like SSH (for Linux) or WinRM (for Windows) to execute tasks using human-readable YAML files called *playbooks*.

## Why use Ansible?

Manually configuring servers one by one is time-consuming, prone to human error, and difficult to scale. You might want to adopt Ansible to:

- **Eliminate Repetitive Work**: Tasks like patching operating systems, updating packages, or creating users can be run across dozens of servers simultaneously with a single command.
- **Achieve Consistency (Idempotence)**: Ansible guarantees that a playbook *will only make changes if the system is not already in the desired state*. Re-running a playbook on a correctly configured server changes nothing, keeping your environment stable.
- **Adopt Infrastructure as Code (IaC)**: Because Ansible configurations are plain text YAML files, they *can be stored, versioned, and tracked in Git*. This allows teams to review, share, and roll back infrastructure changes just like software code.
- **Orchestrate Complex Workflows**: Ansible can coordinate multi-tier deployments, ensuring database servers are updated and running before the web servers connect to them.

## Pros & Cons

| Pros              | Cons |
| :---------------- | :------ | 
| **Agentless**: No agent software to install or maintain on target nodes.        |   **Slower Execution**: Relying on SSH connection overhead can slow down tasks   | 
| **Simple YAML**: Highly readable syntax that is easy for beginners to learn.           |   **No State Tracking**: Does not actively track infrastructure drift like Terraform does.   | 
| **Idempotent**: Safely re-runs playbooks without altering already-configured systems.    |  **Clunky Logic**: Advanced programming logic (like complex loops) is difficult to write in YAML.   | 
| **Huge Ecosystem**: Thousands of pre-built modules and community roles ready to use. |  **Windows Setup**: Configuring WinRM for Windows targets is more complex than Linux SSH   | 


## Step 1: Install Ansible

Ansible only needs to be installed on the control node (the machine from which you will run commands). The target nodes do not need Ansible installed, as long as they have Python and SSH running.

### On Ubuntu / Debian
```bash
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible
```

### On RHEL / Rocky Linux / AlmaLinux:
```bash
sudo dnf install -y epel-release
sudo dnf install -y ansible
```

Verify the installation by running:
```bash
ansible --version
```

## Step 2: Setup dedicated user

For security and organization, it is recommended to create a dedicated user named `ansible` on both the control machine and all target machines.

Run these commands on both the control node and the target nodes:
```bash
sudo adduser ansible
```

After running this command, you will be prompted to privide the password and some optional details

## Step 3: Setup NOPASSWD for Ansible user on Target Machines

To allow the `ansible` user to execute administrative commands (like installing packages) without prompting for a password, you need to configure sudo privileges on the **target machines**.

1. Create a dedicated file using `visudo`:
```bash
sudo visudo -f /etc/sudoers.d/ansible
```
2. Add the configuration line:
```bash
ansible ALL=(ALL) NOPASSWD:ALL
```
3. Save and exit by typing `:wq`

## Step 4: Edit hosts file from the Controller Machine

This is like an optional step, but very useful. The purpose is to use hostnames instead of IP addresses to reffer to the target machines.

1. Edit the `hosts` file on the *controller machine*:
```bash
sudo vim /etc/hosts
```

2. Add your hosts at the end of the file:
```bash
# Ansible target machines
10.0.0.2 host1
10.0.0.3 host2
```
<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

> Please note: The first column contains the IP addresses from the host you use as targets. You can use both public or private addresses depending on where are those VMs deployed.
{: .prompt-warning }

<!-- markdownlint-restore -->

## Step 5: Generate SSH Keys and Copy them to Target Machines

Ansible uses SSH to communicate with target machines. You should set up SSH key-based authentication so the control machine can log in without typing a password.

1. On the **control machine**, switch to the `ansible` user:
```bash
sudo su - ansible
```

2. Generate an SSH key pair (press `Enter` to accept the defaults and leave the passphrase empty):
```bash
ssh-keygen -t ed25519 -C "ansible_key"
```
3. Copy the public key to your target machine(s):

```bash
# Directly by IP address
ssh-copy-id ansible@<target_ip_address>

# Or if you have add the targets on your /etc/hosts file
ssh-copy-id ansible@<target_hostname>
```
## Step 6: How to Test the Connection with an Ad-Hoc Ping

An ad-hoc command is a quick, one-line Ansible command used to perform a single task.
To test if your control machine can communicate with the target, run the `ping` module (this is an **Ansible-specific ping**, not an ICMP ping):

```bash
ansible all -i <target_ip_address>, -u ansible -m ping
```
<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

> Note: The comma after the IP address is required if you are passing a single IP directly without an inventory file.
{: .prompt-tip }

<!-- markdownlint-restore -->

If successful, you will see a response containing:
```json
{
    "changed": false,
    "ping": "pong"
}
```

## Step 7: Test Other Ad-Hoc Commands

Ad-hoc commands are highly useful for quick tasks. The syntax structure is:
```bash
ansible <hosts> -i <inventory> -m <module> -a <arguments>
```
Here are some few examples:

- Check disk space (using the shell module):
```bash
ansible all -i <target_ip_address>, -u ansible -m shell -a "df -h"
```
- Check system uptime (using the command module):
```bash
ansible all -i <target_ip_address>, -u ansible -m command -a "uptime"
```

## Step 8: Setup basic ansible.cfg, Inventory, and Playbook

When managing multiple servers, you should group your settings into files. Create a directory on your control node (as the ansible user) to host these files:
```bash
mkdir ansible-training && cd ansible-training
```
#### The Inventory File
This file lists your target servers. Create a file named `inventory.ini`:
```ini
[webservers]
host1 ansible_python_interpreter=/usr/bin/python3
host2 ansible_python_interpreter=/usr/bin/python3
```
<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

> Please note: This will work if you have specified the IP addresses on the `/etc/hosts` file, if not you would need to add the IP addresses in the following manner:
```ini
[webservers]
host1 ansible_host=<target_ip_address> ansible_python_interpreter=/usr/bin/python3
host2 ansible_host=<target_ip_address> ansible_python_interpreter=/usr/bin/python3
```
{: .prompt-warning }

<!-- markdownlint-restore -->

#### The Configuration File (`ansible.cfg`)
This file defines default settings so you do not have to type them into the command line every time.
Create `ansible.cfg` with this content:
```conf
[defaults]
inventory=inventory.ini
remote_user=ansible
[privilege_escalation]
become=true
become_method=sudo
become_user=root
```
#### The Playbook File

An Ansible playbook is a YAML file containing a list of different componets.

This is the general overview of a playbook:
```yaml
# Ansible playbooks are written in YAML format and always begin with three dashes.
---
- name: "Description of what this entire play does"
  hosts: "target_host_or_group"          # Defined in your inventory file (e.g., webservers, database, all)
  become: true                           # (Optional) Run tasks as root/sudo (true or false)
  become_user: root                      # (Optional) Specify which user to become (defaults to root)
  gather_facts: true                     # (Optional) Gather system information (CPU, RAM, OS, IP) before starting

  # 1. Variables Section (Optional)
  # Define values you want to reuse throughout your tasks
  vars:
    variable_name_1: "some_value"
    variable_name_2: "another_value"

  vars_files:
    - "../path_to_file/external_variables.yaml"  # (Optional) Import variables from an external file

  # 2. Roles Section (Optional)
  # Import pre-packaged, reusable structures of tasks, variables, and files
  roles:
    - common_setup_role
    - webserver_role

  # 3. Tasks Section (Required)
  # The actual steps to execute on your target machines, run in sequential order
  tasks:
    - name: "Description of Task 1"
      module_name:                      # e.g., apt, yum, copy, service, template
        parameter_1: "value_1"
        parameter_2: "value_2"
      register: output_variable          # (Optional) Saves the output/results of this task

    - name: "Description of Task 2 (Using a Condition, Loop, and Notification)"
      module_name:
        parameter_1: "{% raw %}{{ item }}{% endraw %}"       # Loops through the list below
      loop:                             # (Optional) Execute this task multiple times
        - "item_value_A"
        - "item_value_B"
      when: output_variable.changed     # (Optional) Only run this task if the condition is met
      notify: Trigger Handler Name       # (Optional) Triggers a handler if this task makes a change

  # 4. Handlers Section (Optional)
  # Tasks that only run if they are "notified" by a task that successfully changed something.
  # Handlers run once at the very end of the play (useful for restarting services).
  handlers:
    - name: "Trigger Handler Name"       # Must match the 'notify' name in your task exactly
      module_name:
        parameter_1: "value"
```
<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

> Note: This represents a brief overview about the structure of a general ansible playbook. It is highly recommended to check the  [official Ansible documentation - builtin modules](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/index.html#ansible-builtin) to have a better understanding about the builtin functionalities and other components.
{: .prompt-tip }

<!-- markdownlint-restore -->

- Create `apt.yaml` to update and upgrade all packages (this example assumes an Ubuntu/Debian target):

```yaml
---
- name: update apt packages
  hosts: "*"
  become: yes
  tasks:
    - name: Update and upgrade apt packages
      apt:
        update_cache: yes
        upgrade: 'yes'
```
- Create `apache2.yaml` to install and start the Apache web server (this example assumes an Ubuntu/Debian target):

```yaml
---
- name: Setup Web Server
  hosts: webservers
  become: true
  tasks:
    - name: Ensure Apache is installed
      ansible.builtin.apt:
        name: apache2
        state: present
        update_cache: yes

    - name: Ensure Apache is running and enabled
      ansible.builtin.service:
        name: apache2
        state: started
        enabled: yes
```

#### Running the Playbook
To run your playbook, use the `ansible-playbook` command:
```bash
ansible-playbook playbook.yaml
```

## How to translate a Shell Script to an Ansible Playbook

### Case 1: Nginx setup

#### Scenario Description

Let's say we have the following bash script:
```bash
#!/bin/bash
sudo apt update
sudo apt install -y nginx
echo "<h1>Welcome to my website</h1>" | sudo tee /var/www/html/index.html
sudo systemctl restart nginx
```
This bash script updates the package list, installs Nginx, writes an HTML file, and restarts Nginx.

What would be the equivalent ansible playbook for this bash script?

Let's have a look here:
```yaml
---
- name: Deploy Nginx Web Server
  hosts: webservers
  become: true
  tasks:
    - name: Install Nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Create index.html file
      ansible.builtin.copy:
        content: "<h1>Welcome to my website</h1>"
        dest: /var/www/html/index.html
        owner: www-data
        group: www-data
        mode: '0644'

    - name: Start and enable Nginx service
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: yes
```
While you can use the shell module in Ansible, the best practice is to use dedicated modules. Dedicated modules are *idempotent*, meaning they will only make changes if the system is not already in the desired state.

### Case 2: Managing Custom Application Logs

#### Scenario Description
The objective is to do the following:
1. Create a dedicated directory for a custom application's logs (`/var/log/custom_service`) and set its ownership so the system logger can write to it.
2. Initialize an empty log file to avoid starting errors.
3. Deploy a custom rotation policy to `/etc/logrotate.d/custom_service` so logs are rotated daily, compressed, and cleaned up after 7 days.
4. Run a command-line check to validate the logrotate configuration without actually modifying the system state. 

#### The Bash Script Version (`setup_log_rotate.sh`)
In Bash, you write multiline files using "here documents" (`cat << 'EOF'`), and verifying configuration syntax usually requires manually executing the command and inspecting the terminal output.

```bash
#!/bin/bash

# 1. Create the log directory and set correct system permissions
sudo mkdir -p /var/log/custom_service
sudo chown syslog:adm /var/log/custom_service
sudo chmod 755 /var/log/custom_service

# 2. Initialize an empty log file
sudo touch /var/log/custom_service/service.log
sudo chown syslog:adm /var/log/custom_service/service.log
sudo chmod 640 /var/log/custom_service/service.log

# 3. Create the logrotate configuration rule
sudo cat << 'EOF' > /etc/logrotate.d/custom_service
/var/log/custom_service/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 syslog adm
}
EOF

# Ensure logrotate file has secure root-only write permissions
sudo chmod 644 /etc/logrotate.d/custom_service

# 4. Dry-run test the configuration to make sure there are no syntax errors
sudo logrotate -d /etc/logrotate.d/custom_service
```

#### The Ansible Playbook Version (`log_rotate.yaml`)
Because we are not using variables, we can embed the configuration file's text directly inside the playbook using YAML's multiline string indicator (`|`).

We also use `changed_when: false` on the final command to ensure that running a syntax test doesn't flag the play as "changed" in our final Ansible report.

```yaml
---
- name: Configure Custom Log Rotation Policy
  hosts: webservers
  become: true

  tasks:
    # 1. Manage Directory and File State
    - name: Ensure custom log directory exists with correct system permissions
      ansible.builtin.file:
        path: /var/log/custom_service
        state: directory
        owner: syslog
        group: adm
        mode: '0755'

    - name: Initialize the empty active log file
      ansible.builtin.file:
        path: /var/log/custom_service/service.log
        state: touch
        owner: syslog
        group: adm
        mode: '0640'
        modification_time: preserve
        access_time: preserve

    # 2. Write the multi-line config block directly to a destination file
    - name: Deploy custom logrotate configuration
      ansible.builtin.copy:
        content: |
          /var/log/custom_service/*.log {
              daily
              rotate 7
              compress
              delaycompress
              missingok
              notifempty
              create 0640 syslog adm
          }
        dest: /etc/logrotate.d/custom_service
        owner: root
        group: root
        mode: '0644'

    # 3. Validation task (Dry-run command)
    - name: Validate logrotate configuration syntax
      ansible.builtin.command: logrotate -d /etc/logrotate.d/custom_service
      # Because this command only reads the file to verify it (and makes no changes),
      # we tell Ansible to always report this task's status as green/unchanged.
      changed_when: false
```

#### Key highlights
1. Multiline Copying (`|`):
Using `content: |` in a `copy` block allows them to write clean, multi-line configuration files directly within their task list.

2. The `touch` State:
The playbook demonstrates how the `file` module can safely initialize a file if it does not exist, without modifying timestamps if the file is already there (`modification_time: preserve`).

3. The Importance of `changed_when: false`
By default, running a `command` in Ansible will always show a status of "Changed" in yellow, even if the command was just a diagnostic test. Introducing `changed_when: false` is an excellent intermediate best practice, as it ensures Ansible's final run summary remains accurate.

### Case 3: Log auditing

#### Scenario Description
The objective is to scan a custom application log (`/var/log/myapp.log`) for any entries containing `ERROR` or `CRITICAL`.

- **If errors are found**: Loop through the matching lines, prefix them with an `[ALERT]` tag, write them to an audit file (`/var/log/myapp_errors.log`), and ensure the success status file is removed.
- **If no errors are found**: Write a "healthy" status message with the current date to /var/log/myapp_status.txt and ensure the old error log is cleaned up.
- **Self-containment**: If the dummy log doesn't exist, create one so the code can run immediately.

#### The Bash Script Version (`audit.sh`)

In procedural scripting, you use commands like `grep`, test brackets `[ -n "$VAR" ]`, standard bash `while` loops, and `if/else` statements.

```bash
#!/bin/bash
LOG_FILE="/var/log/myapp.log"
ERROR_FILE="/var/log/myapp_errors.log"
STATUS_FILE="/var/log/myapp_status.txt"

# 1. Condition: Create a dummy log if it doesn't exist yet
if [ ! -f "$LOG_FILE" ]; then
    echo "Creating dummy log file..."
    sudo mkdir -p /var/log
    echo "2026-07-15 10:00:00 INFO Application started" | sudo tee -a "$LOG_FILE"
    echo "2026-07-15 10:05:00 ERROR Database connection failed" | sudo tee -a "$LOG_FILE"
    echo "2026-07-15 10:10:00 WARNING High memory usage" | sudo tee -a "$LOG_FILE"
    echo "2026-07-15 10:15:00 CRITICAL Out of memory" | sudo tee -a "$LOG_FILE"
fi

# 2. Check for matching errors using Regex (Grep)
# Store matching lines in a variable
ERRORS=$(grep -E "ERROR|CRITICAL" "$LOG_FILE")

# 3. Conditional Logic: IF errors were found
if [ -n "$ERRORS" ]; then
    echo "Errors detected! Extracting to $ERROR_FILE..."
    
    # Initialize/overwrite file with header
    echo "=== Error Log Audit - $(date) ===" | sudo tee "$ERROR_FILE"
    
    # LOOP: Iterate through each matched line to prepend an alert tag
    echo "$ERRORS" | while read -r line; do
        echo "[ALERT] $line" | sudo tee -a "$ERROR_FILE"
    done
    
    # Clean up the healthy status file if it existed previously
    sudo rm -f "$STATUS_FILE"

# ELSE: If no errors were found
else
    echo "No errors detected. Updating status file..."
    echo "System healthy as of $(date)" | sudo tee "$STATUS_FILE"
    
    # Clean up old error logs
    sudo rm -f "$ERROR_FILE"
fi
```
#### The Ansible Playbook Version (`audit.yml`)
In Ansible, we replace procedural commands with descriptive tasks. We use **variables, conditionals (`when`)**, and **loops (`loop`)** to control the flow.


```yaml
---
- name: Log Audit and Report Generator
  hosts: webservers
  become: true
  gather_facts: true # Enabled to get native date and time variables
  
  tasks:
    # 1. Ansible equivalent of: if [ ! -f "$LOG_FILE" ]
    - name: Ensure dummy log file exists (only if missing)
      ansible.builtin.copy:
        content: |
          2026-07-15 10:00:00 INFO Application started
          2026-07-15 10:05:00 ERROR Database connection failed
          2026-07-15 10:10:00 WARNING High memory usage
          2026-07-15 10:15:00 CRITICAL Out of memory
        dest: /var/log/myapp.log
        force: no # "force: no" ensures we don't overwrite it if it exists
        mode: '0644'

    # 2. Extract errors and save output to an Ansible variable using 'register'
    - name: Scan log file for ERROR or CRITICAL entries
      ansible.builtin.command: grep -E "ERROR|CRITICAL" /var/log/myapp.log
      register: log_search
      # Grep exits with code 1 if no matches are found. 
      # We tell Ansible not to fail the playbook when this happens.
      failed_when: false
      changed_when: false

    # ----------------- BRANCH A: Errors exist -----------------

    # Initialize the error file only if errors are found
    - name: Create or clear the audit file with header
      ansible.builtin.copy:
        content: "=== Error Log Audit ===\n"
        dest: /var/log/myapp_errors.log
        mode: '0644'
      when: log_search.stdout_lines | length > 0

    # LOOP: Iterate over each matched line found in log_search
    - name: Loop through errors and write formatted lines to audit file
      ansible.builtin.lineinfile:
        path: /var/log/myapp_errors.log
        line: "[ALERT] {% raw %}{{ item }}{% endraw %}"
        create: yes
      loop: {% raw %}"{{ log_search.stdout_lines }}" {% endraw %}      
      when: log_search.stdout_lines | length > 0

    - name: Ensure healthy status file is absent when errors exist
      ansible.builtin.file:
        path: /var/log/myapp_status.txt
        state: absent
      when: log_search.stdout_lines | length > 0

    # ----------------- BRANCH B: No errors exist -----------------

    - name: Create status file if system is healthy
      ansible.builtin.copy:
        content: "System healthy as of {{ ansible_date_time.date }} {{ ansible_date_time.time }}\n"
        dest: /var/log/myapp_status.txt
        mode: '0644'
      when: log_search.stdout_lines | length == 0

    - name: Ensure old error audit file is absent when system is healthy
      ansible.builtin.file:
        path: /var/log/myapp_errors.log
        state: absent
      when: log_search.stdout_lines | length == 0
```
#### Key highlights
1. **Variables & Registering Output (`register`)**:
In bash, you save command output with `ERRORS=$(command)`. In Ansible, you use the `register` keyword. This saves the entire execution metadata (including stdout, stderr, and exit codes) into a dictionary variable (`log_search`) that you can query in later tasks.
2. **Handling Command Failure (`failed_when`)**:
Usually, if a shell command returns a non-zero exit code, Ansible assumes something went wrong and halts execution on that host. Since `grep` returns exit code `1` when it finds zero matches, we use `failed_when: false` to allow the playbook to continue running normally.
3. **Conditionals (`when`)**:
Bash uses `if [ ... ]; then`. Ansible uses the `when` parameter attached to individual tasks. The task is skipped entirely unless the condition evaluates to `true`. In this playbook, we check the length of the registered line list: `log_search.stdout_lines | length > 0`.
4. **Loops (`loop`)**:
Instead of a bash `while read` block, Ansible features a built-in `loop` parameter. When you provide a list to `loop`, the task executes once for each item, and you access the current value using the special {% raw %}`{{ item }}` {% endraw %} placeholder.