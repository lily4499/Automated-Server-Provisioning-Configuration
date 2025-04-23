# Automated-Server-Provisioning-Configuration


```markdown
# 🔧 Automated Server Provisioning & Configuration

This project automates the deployment of a full-stack web server (NGINX + Node.js + MongoDB) using:

- **Bash** for provisioning an Ubuntu EC2 instance
- **Python** for application setup
- **Ansible** for configuration management

## ✅ Outcome

Push-button provisioning and configuration of a production-ready stack on AWS EC2.

---

## 📁 Project Structure

```bash

automated-deployment/
│
├── provision.sh                  # Bash script to provision EC2
├── setup_app.py                  # Python script to install Node.js app
├── ansible/
│   ├── inventory                 # Ansible inventory file
│   ├── playbook.yml              # Main Ansible playbook
│   └── roles/
│       ├── nginx/
│       │   └── tasks/main.yml
│       ├── mongodb/
│       │   └── tasks/main.yml
│       └── nodeapp/
│           └── tasks/main.yml


### `setup-files.py`

```python
import os

# Directory structure
structure = [
    "automated-deployment/provision.sh",
    "automated-deployment/setup_app.py",
    "automated-deployment/ansible/inventory",
    "automated-deployment/ansible/playbook.yml",
    "automated-deployment/ansible/roles/nginx/tasks/main.yml",
    "automated-deployment/ansible/roles/mongodb/tasks/main.yml",
    "automated-deployment/ansible/roles/nodeapp/tasks/main.yml"
]

# Create directories and empty files
for path in structure:
    os.makedirs(os.path.dirname(path), exist_ok=True)
    with open(path, "w") as f:
        f.write("")  # Create an empty file

"Files and directories have been created."

### Run:
```bash
python3 setup-files.py

---

## 🔰 Prerequisites

- AWS CLI configured
- SSH key pair (`.pem`) created in EC2
- Python 3 installed
- Ansible installed
- IAM role with EC2 permissions

---

## 🧱 Step 1: Provision EC2 with Bash

### `provision.sh`

```bash
#!/bin/bash

AMI_ID="ami-084568db4383264d4"
INSTANCE_TYPE="t2.micro"
KEY_NAME="ec2-devops-key"
SECURITY_GROUP="web-sg"
REGION="us-east-1"

aws ec2 create-security-group --group-name $SECURITY_GROUP --description "Web server SG" --region $REGION

aws ec2 authorize-security-group-ingress --group-name $SECURITY_GROUP --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-name $SECURITY_GROUP --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-name $SECURITY_GROUP --protocol tcp --port 3000 --cidr 0.0.0.0/0

INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --count 1 \
  --instance-type $INSTANCE_TYPE \
  --key-name $KEY_NAME \
  --security-groups $SECURITY_GROUP \
  --region $REGION \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "Waiting for instance to start..."
aws ec2 wait instance-running --instance-ids $INSTANCE_ID

PUBLIC_IP=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)

echo "Instance is running at IP: $PUBLIC_IP"
```

### Run:
```bash
chmod +x provision.sh
./provision.sh
```

---

## 🐍 Step 2: App Setup with Python

### `setup_app.py`

```python
import subprocess

def run(command):
    subprocess.run(command, shell=True, check=True)

def main():
    run("sudo apt update")
    run("sudo apt install -y git")
    run("git clone https://github.com/lily4499/devops-demo.git")
    run("curl -fsSL https://deb.nodesource.com/setup_16.x | sudo -E bash -")
    run("sudo apt install -y nodejs")
    run("cd devops-demo && npm install")
    print("Node.js app setup complete.")

if __name__ == "__main__":
    main()
```

### Run:
```bash
scp -i /home/lilia/ec2.pem setup_app.py ubuntu@<PUBLIC_IP>:~
ssh -i /home/lilia/ec2.pem ubuntu@<PUBLIC_IP>
python3 setup_app.py
cd devops-demo
node app.js

```

---

## ⚙️ Step 3: Configuration with Ansible

### `ansible/inventory`

```ini
webserver ansible_host=<PUBLIC_IP> ansible_user=ubuntu ansible_ssh_private_key_file=/home/lilia/ec2.pem
```

### `ansible/playbook.yml`

```yaml
- name: Configure Web Server
  hosts: webserver
  become: true
  roles:
    - nginx
    - mongodb
    - nodeapp
```

### Role: `nginx/tasks/main.yml`

 vim ansible/roles/nginx/tasks/main.yml

```yaml
- name: Install NGINX
  apt:
    name: nginx
    state: present

- name: Start NGINX
  service:
    name: nginx
    state: started
    enabled: true

- name: Copy NGINX reverse proxy config
  copy:
    dest: /etc/nginx/sites-available/default
    content: |
      server {
          listen 80;
          server_name localhost;

          location / {
              proxy_pass http://localhost:3000;
              proxy_http_version 1.1;
              proxy_set_header Upgrade $http_upgrade;
              proxy_set_header Connection 'upgrade';
              proxy_set_header Host $host;
              proxy_cache_bypass $http_upgrade;
          }
      }

- name: Reload NGINX
  service:
    name: nginx
    state: reloaded

```

### Role: `mongodb/tasks/main.yml`

vim ansible/roles/mongodb/tasks/main.yml

```yaml
- name: Import MongoDB key
  apt_key:
    url: https://www.mongodb.org/static/pgp/server-4.4.asc
    state: present

- name: Add MongoDB repository
  apt_repository:
    repo: deb [ arch=amd64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/4.4 multiverse
    state: present

- name: Install MongoDB
  apt:
    name: mongodb-org
    state: present
    update_cache: yes

- name: Start MongoDB
  service:
    name: mongod
    state: started
    enabled: true
```

### Role: `nodeapp/tasks/main.yml`

vim ansible/roles/nodeapp/tasks/main.yml

```yaml
- name: Install Node.js
  shell: |
    curl -fsSL https://deb.nodesource.com/setup_16.x | sudo -E bash -
    apt install -y nodejs
  args:
    executable: /bin/bash

- name: Clone Node.js app
  git:
    repo: https://github.com/lily4499/devops-demo.git
    dest: /home/ubuntu/nodeapp

- name: Install app dependencies
  shell: npm install
  args:
    chdir: /home/ubuntu/nodeapp

- name: Start the app
  shell: nohup node app.js &
  args:
    chdir: /home/ubuntu/nodeapp


```

### Run:
```bash
sudo apt install ansible-core
cd ansible
ansible-playbook -i inventory playbook.yml
```

---

## 🌐 Final Output

Visit your app at:

```
http://<PUBLIC_IP>
```

You should see your Node.js app served by NGINX, connected to MongoDB.

---

## 🧰 Tools Used

| Tool      | Purpose                          |
|-----------|----------------------------------|
| Bash      | EC2 provisioning                 |
| Python    | App setup and scripting          |
| Ansible   | Configuration management         |
| AWS EC2   | Virtual server (Ubuntu 22.04)    |

---

## 🧠 Author

Liliane Konissi — [GitHub](https://github.com/lily4499/Automated-Server-Provisioning-Configuration
)

```

