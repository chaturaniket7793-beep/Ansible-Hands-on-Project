# Ansible Automated Node.js Deployment

## Project Overview

This project demonstrates an automated **Node.js application deployment using Ansible on AWS EC2**.

The project uses a **Controller-Worker architecture**, where Ansible is installed on an Ubuntu 24.04 LTS Controller instance and remotely manages an Ubuntu 24.04 LTS Worker instance in the same AWS VPC.

Ansible automates the complete application deployment process, including:

* Installing required packages
* Installing Node.js 20
* Creating the application directory
* Cloning the Node.js application from GitHub
* Installing application dependencies
* Starting the Node.js application

---

## Architecture

```text
                         AWS VPC
                            |
             +--------------+--------------+
             |                             |
             |                             |
     +---------------+             +---------------+
     |   Controller  |             |    Worker     |
     | Ubuntu 24.04  |             | Ubuntu 24.04  |
     |               |   SSH       |               |
     |   Ansible     | ----------> | Node.js App   |
     +---------------+             +---------------+
             ^
             |
             | SSH
             |
       Local Machine
```

---

## Project Structure

```text
ansible-node-app/
│
├── inventory.ini
├── playbook.yml
├── README.md
└── .gitignore
```

### Files Description

| File            | Description                                                 |
| --------------- | ----------------------------------------------------------- |
| `inventory.ini` | Defines the Worker EC2 instance and SSH configuration       |
| `playbook.yml`  | Automates Node.js application deployment                    |
| `README.md`     | Project documentation                                       |
| `.gitignore`    | Prevents private files such as SSH keys from being uploaded |

---

# AWS Infrastructure

## 1. Controller Instance

Create an Ubuntu 24.04 LTS EC2 instance.

The Controller is responsible for:

* Installing Ansible
* Managing the Worker
* Executing the Ansible playbook
* Connecting to the Worker through SSH

## 2. Worker Instance

Create another Ubuntu 24.04 LTS EC2 instance in the **same VPC**.

The Worker is responsible for:

* Running Node.js
* Hosting the application
* Receiving deployment instructions from Ansible

---

# Security Groups

## Controller Security Group

Allow:

| Type | Port | Source                |
| ---- | ---: | --------------------- |
| SSH  |   22 | Your local machine IP |

Example:

```text
SSH
TCP
22
My IP
```

---

## Worker Security Group

Allow:

| Type       | Port | Source                    |
| ---------- | ---: | ------------------------- |
| SSH        |   22 | Controller Security Group |
| Custom TCP | 3000 | Your IP                   |

The Worker should accept SSH connections from the **Controller Security Group**, rather than exposing SSH to the entire internet.

Port `3000` is used for testing the Node.js application.

---

# Step 1: Connect to Controller

From your local machine, connect to the Controller:

```bash
ssh -i aws-key.pem ubuntu@CONTROLLER_PUBLIC_IP
```

---

# Step 2: Install Ansible

On the Controller:

```bash
sudo apt update
sudo apt install ansible -y
```

Verify:

```bash
ansible --version
```

Example:

```text
ansible [core ...]
```

---

# Step 3: Configure SSH Key

The Controller needs the SSH private key to connect to the Worker.

Copy your AWS `.pem` key to the Controller.

For example:

```text
/home/ubuntu/aws-key.pem
```

Set the correct permissions:

```bash
chmod 400 /home/ubuntu/aws-key.pem
```

Test SSH connectivity:

```bash
ssh -i /home/ubuntu/aws-key.pem ubuntu@WORKER_PRIVATE_IP
```

If successful, exit the Worker:

```bash
exit
```

---

# Step 4: Create Project Directory

On the Controller:

```bash
mkdir ~/ansible-node-app
cd ~/ansible-node-app
```

Create the required files:

```bash
touch inventory.ini playbook.yml README.md .gitignore
```

---

# Step 5: Configure Ansible Inventory

Edit:

```bash
nano inventory.ini
```

Add:

```ini
[webservers]
WORKER_PRIVATE_IP ansible_user=ubuntu ansible_ssh_private_key_file=/home/ubuntu/aws-key.pem
```

Replace:

```text
WORKER_PRIVATE_IP
```

with the private IP address of your Worker EC2 instance.

Example:

```ini
[webservers]
10.0.135.1 ansible_user=ubuntu ansible_ssh_private_key_file=/home/ubuntu/aws-key.pem
```

---

# Step 6: Test Ansible Connectivity

Run:

```bash
ansible webservers -i inventory.ini -m ping
```

Expected result:

```text
10.0.135.1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

This confirms that the Controller can communicate with the Worker using Ansible.

---

# Step 7: Create the Ansible Playbook

Edit:

```bash
nano playbook.yml
```

Use the deployment playbook:

```yaml
---
- name: Deploy Node.js Application
  hosts: webservers
  become: yes

  vars:
    repo_url: "https://github.com/rat9615/simple-nodejs-app"
    app_directory: "/opt/node-app"
    nodejs_version: "20"

  tasks:

    - name: Update apt package cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install required dependencies
      apt:
        name:
          - git
          - curl
        state: present

    - name: Install Node.js repository
      shell: |
        curl -fsSL https://deb.nodesource.com/setup_{{ nodejs_version }}.x | bash -
      args:
        creates: /etc/apt/sources.list.d/nodesource.list

    - name: Install Node.js
      apt:
        name: nodejs
        state: present
        update_cache: yes

    - name: Create application directory
      file:
        path: "{{ app_directory }}"
        state: directory
        mode: '0755'

    - name: Clone Node.js application
      git:
        repo: "{{ repo_url }}"
        dest: "{{ app_directory }}"
        version: main

    - name: Install Node.js application dependencies
      npm:
        path: "{{ app_directory }}"
        state: present

    - name: Start Node.js application
      shell: |
        nohup npm start > /var/log/node-app.log 2>&1 &
      args:
        chdir: "{{ app_directory }}"
```

---

# Step 8: Validate Playbook Syntax

Before deployment, run:

```bash
ansible-playbook -i inventory.ini playbook.yml --syntax-check
```

Expected:

```text
playbook: playbook.yml
```

---

# Step 9: Deploy the Application

Run:

```bash
ansible-playbook -i inventory.ini playbook.yml
```

The playbook will automatically:

```text
1. Update apt
       ↓
2. Install Git and curl
       ↓
3. Configure Node.js repository
       ↓
4. Install Node.js 20
       ↓
5. Create /opt/node-app
       ↓
6. Clone GitHub repository
       ↓
7. Install npm dependencies
       ↓
8. Start Node.js application
```

---

# Step 10: Verify Node.js

Check the Node.js version:

```bash
ansible webservers -i inventory.ini -m shell -a "node --version"
```

Expected:

```text
v20.x.x
```

---

# Step 11: Verify Application Process

Run:

```bash
ansible webservers -i inventory.ini -m shell -a "ps aux | grep node"
```

You should see the Node.js process running.

---

# Step 12: Check Application Log

Run:

```bash
ansible webservers -i inventory.ini -m shell -a "tail -n 20 /var/log/node-app.log"
```

This helps verify whether the Node.js application started successfully.

---

# Step 13: Verify Application Port

On the Worker:

```bash
sudo ss -lntp | grep 3000
```

If the application listens on port 3000, you should see something similar to:

```text
LISTEN 0 511 0.0.0.0:3000
```

---

# Step 14: Test from Browser

Open:

```text
http://WORKER_PUBLIC_IP:3000
```

Replace:

```text
WORKER_PUBLIC_IP
```

with the Worker EC2 public IP address.

If the application is running correctly, the Node.js application should be displayed.

---

# GitHub Security

**Do not upload your AWS private key to GitHub.**

Add the following to `.gitignore`:

```gitignore
*.pem
*.key
credentials.txt
.env
```

This prevents private credentials and sensitive configuration from being committed.

---

# Git Commands

Initialize the repository:

```bash
git init
```

Add files:

```bash
git add inventory.ini playbook.yml README.md .gitignore
```

Check:

```bash
git status
```

Commit:

```bash
git commit -m "Add Ansible Node.js deployment project"
```

Add your GitHub repository:

```bash
git remote add origin https://github.com/YOUR_USERNAME/ansible-node-app.git
```

Rename the branch:

```bash
git branch -M main
```

Push:

```bash
git push -u origin main
```

---

# Final Deployment Flow

```text
Local Machine
      |
      | SSH
      v
AWS EC2 Controller
Ubuntu 24.04
      |
      | Ansible + SSH
      v
AWS EC2 Worker
Ubuntu 24.04
      |
      | Clone
      v
GitHub Node.js Repository
      |
      | npm install
      v
Node.js Application
      |
      | Port 3000
      v
Web Browser
```

---

# Technologies Used

* AWS EC2
* Ubuntu 24.04 LTS
* Ansible
* Node.js 20
* npm
* Git
* GitHub
* SSH

---

# Project Outcome

This project demonstrates how **Ansible can automate Node.js application deployment on AWS EC2** using a Controller-Worker architecture.

The deployment process is repeatable and reduces manual configuration by automating server preparation, application installation, dependency installation, and application startup.
