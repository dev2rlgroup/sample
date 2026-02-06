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

---

## 🕵️ Under the Hood: The SSH Connection Flow

Understanding this breakdown will help you debug pipeline issues.

### ⛓️ The SSH Workflow (Visualized)

#### 1. The Jenkins Container (The Source)
Jenkins doesn't just "send" the file; it uses the **Ansible Plugin** to start an SSH session.

*   **Commands Run Behind the Scenes**: When you click build, Jenkins executes: `ansible-playbook -i hosts.ini web-app.yml --extra-vars "..."`
*   **The Credentials**: Ansible looks at your `hosts.ini`. Because you provided `ansible_user=jenkins` and `ansible_password=***`, it uses a tool called `sshpass` (which we installed) to feed the password into the SSH connection automatically.
*   **The Security Bypass**: Because we added `-o StrictHostKeyChecking=no`, Jenkins ignores the "Unknown Host" warning and proceeds to connect.

#### 2. The Prod-Server (The Target)
This is the Ubuntu container where the "Reception" happens.

*   **The SSH Daemon (`sshd`)**: When you ran `service ssh start`, you opened **Port 22**. The container is now "listening" for Jenkins to knock on the door.
*   **The User (`jenkins`)**: When the connection hits, the server checks `/etc/passwd`. It finds the `jenkins` user you created and verifies the password you set.
*   **The Work Area (`/home/jenkins`)**: Once logged in, Ansible needs to run Python scripts. It looks for `/home/jenkins/.ansible/tmp`. Since we ran `mkdir` and `chown`, the user has permission to write these temporary files.

#### 3. The "Become" Process (Escalating Power)
This is how your file actually gets into the protected `/var/www/html` folder.

*   **Standard User**: Jenkins logs in as a normal user (`jenkins`). Normal users cannot write to `/var/www/html`.
*   **The Sudo Request**: Your playbook says `become: true`. Ansible sends a command: `sudo cp ...`
*   **The Sudoers Check**: The server looks at `/etc/sudoers`. It finds the line you added: `jenkins ALL=(ALL) NOPASSWD:ALL`
*   **The Action**: Because of that line, the server says "Okay, you don't need a password to be root," and the file is copied successfully.

### 📋 Summary of Key Setup Commands

If you ever have to rebuild this from scratch, these are your "Magic" commands:

**On Jenkins Container:**
```bash
apt-get install -y sshpass
export ANSIBLE_HOST_KEY_CHECKING=False
```

**On Prod-Server (Ubuntu) Container:**
```bash
# Setup User & SSH
useradd -m -s /bin/bash jenkins
echo "jenkins:yourpassword" | chpasswd
apt install -y openssh-server sudo
service ssh start

# Setup Permissions
echo "jenkins ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
chown -R jenkins:jenkins /home/jenkins
```

---

## 🧹 Keep It Clean: Best Practices

It is a smart move to keep your production server clean. If you don't clean up, old files from previous versions might stay in the folder and cause confusion or security risks.

Here is how to add a Cleanup task and a Python Interpreter fix to make your playbook professional.

### 🛠️ Updated web-app.yml (With Cleanup)

Open your playbook with `nano /var/jenkins_home/playbooks/web-app.yml` and update it to this:

```yaml
---
- name: Upload new code and ReStart the service
  hosts: prod-server
  vars:
    app: apache2
    # This line removes the pink "Python Discovery" warning
    ansible_python_interpreter: /usr/bin/python3
    
  tasks:
    - name: Remove old index file
      become: true
      file:
        path: /var/www/html/index.html
        state: absent

    - name: Place custom HTML File
      become: true
      copy:
        src: /var/jenkins_home/workspace/web-app/index.html
        dest: /var/www/html/index.html

    - name: Start Apache service manually
      become: true
      shell: "apache2ctl restart"
```

### 🔍 What Changed?
*   **`state: absent`**: This tells Ansible: "Check if this file exists; if it does, delete it." This ensures that you are always starting with a fresh slate before the copy task runs.
*   **`ansible_python_interpreter`**: By explicitly telling Ansible where Python is, you save a few seconds during the "Gathering Facts" stage and keep your Jenkins logs clean.

### 🛡️ One Final Security Tip: User Passwords
In your `hosts.ini`, you currently have your password written in plain text: `ansible_password=YOUR_PASSWORD`

As you get more comfortable, the next step is to use Jenkins Credentials. You would:
1.  Store the password in Jenkins (**Manage Jenkins -> Credentials**).
2.  Give it an ID like `prod-server-pass`.
3.  In your Ansible build step, select that credential instead of writing it in the file.

### 🏁 Verify the Automatic Build
Since you set up SCM Polling (`* * * * *`):
1.  Go to your GitHub repo.
2.  Edit `index.html`.
3.  Save/Commit.
4.  Wait 60 seconds.
5.  Watch Jenkins start **Build #8** (or whichever number is next) automatically!
