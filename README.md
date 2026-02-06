# Jenkins + Ansible CI/CD Pipeline Documentation

This repository documents the CI/CD flow configured for deploying `index.html` to a production server using Jenkins and Ansible.

## 🏗️ Architecture Overview (The Flow)

The deployment process follows a "relay race" model with three stages:

1.  **The Trigger**:
    -   Code is pushed to GitHub.
    -   Jenkins detects the change and pulls the latest `index.html` into its Workspace.

2.  **The Hand-off**:
    -   Jenkins triggers an Ansible build step.
    -   It provides Ansible with the "Map" (`hosts.ini`) and the "Instructions" (`web-app.yml`).

3.  **The Deployment**:
    -   Ansible calculates the differences and connects to the `prod-server` via SSH.
    -   It copies the file to the app directory and instructs Apache to refresh if necessary.

### ⚙️ Jenkins Build Step Configuration
When configuring the "Invoke Ansible Playbook" build step in Jenkins, use the following details:

*   **Playbook Path**: `/var/jenkins_home/playbooks/web-app.yml`
    *   *Why?* This is the "Instructions" file. We ask Jenkins to look here because we mounted our `web-app.yml` into this specific directory inside the container. It tells Ansible *what* to do (install Apache, copy index.html).
*   **Inventory (File or Host List)**: `/var/jenkins_home/playbooks/hosts.ini`
    *   *Why?* This is the "Map". It tells Ansible *where* to go (the IP address of the prod-server). Without this, Ansible wouldn't know which server to contact.

#### 🔐 Extra Variables (Hidden/Concealed)
Under the "Advanced" or "Extra Variables" section of the build step, we inject sensitive or complex arguments so they aren't hardcoded in the playbook.

*   **Key**: `ansible_become_pass` | **Value**: *(Concealed)*
    *   *Why?* This passes the `sudo` password (usually `jenkins` in this lab setup) to Ansible. Even though we configured `visudo`, passing this ensures Ansible can definitely escalate privileges to restart the Apache service without getting stuck at a password prompt.
*   **Key**: `ansible_ssh_common_args` | **Value**: *(Concealed)*
    *   *Why?* This usually contains flags like `-o StrictHostKeyChecking=no`. It prevents the build from failing if the Docker container is recreated and the SSH fingerprint changes (which would normally trigger a "Host Key Verification" error).

---

## 🔑 Credentials & Connectivity

The communication between Jenkins and the Production Server relies on **SSH Key-Based Authentication** (no passwords).

| Component | Detail |
| :--- | :--- |
| **User & Pass of prod-server ** | `jenkins` `jenkins` (Running commands inside containers) |
| **SSH Key** | `/var/jenkins_home/.ssh/id_rsa` |
| **Target Host** | `prod-server` (IP defined in `playbooks/hosts.ini`) |
| **App Path** | `/var/www/html` |
| **Sudo Access** | Passwordless (Configured via `visudo` for `become: true`) |

---

## 💻 Essential Commands Cheatsheet

### 1. Manual Ansible Testing (Run from Jenkins Container)
Use these commands to verify connectivity and configuration without running a full pipeline build.

*   **The "Heartbeat" Check (Ping):**
    ```bash
    ansible prod-server -i /var/jenkins_home/playbooks/hosts.ini -m ping
    ```

*   **Check Disk Space on Prod:**
    ```bash
    ansible prod-server -i /var/jenkins_home/playbooks/hosts.ini -a "df -h"
    ```

*   **Manually Trigger the Playbook:**
    ```bash
    ansible-playbook /var/jenkins_home/playbooks/web-app.yml -i /var/jenkins_home/playbooks/hosts.ini
    ```

### 2. SSH Maintenance (Run from Jenkins Container)

*   **Manual Login:**
    ```bash
    ssh prod-server
    ```

*   **Fix "Host Key Verification" Errors:**
    If the `prod-server` container is recreated, its fingerprint changes. Update `known_hosts`:
    ```bash
    ssh-keyscan -H prod-server >> ~/.ssh/known_hosts
    ```

### 3. Troubleshooting the Production Server (Run from Prod-Server Container)
If deployment succeeds but the site isn't loading, check the server directly.

*   **Check Apache Status:**
    ```bash
    service apache2 status
    # OR if service command fails:
    ps aux | grep apache
    ```

*   **View Apache Error Logs:**
    ```bash
    tail -f /var/log/apache2/error.log
    ```

*   **Verify File Permissions:**
    ```bash
    ls -la /var/www/html/
    ```

---

## 🐳 Docker Architecture & Ports

Current container status as referenced in this setup:

*   **Production Server (`prod-server`)**:
    *   Image: `ubuntu:latest`
    *   Port Mapping: `8093:80` (Access web application at `localhost:8093`)
    *   Container ID: `e37bd2d95960` (example)

*   **Jenkins Controller (`my-jenkins-9001`)**:
    *   Image: `jenkins/jenkins:lts`
    *   Port Mapping: `9001:8080` (Access Jenkins UI at `localhost:9001`)
    *   Container ID: `c08f756d9ca0` (example)
