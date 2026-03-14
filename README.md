# CI/CD Pipeline for Containerized Flask Application on AWS EC2

> A complete DevOps project demonstrating Infrastructure as Code, containerization, and automated CI/CD pipeline deployment.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Phase 1 — Application & Docker Setup](#phase-1--application--docker-setup)
- [Phase 2 — AWS & Terraform Infrastructure](#phase-2--aws--terraform-infrastructure)
- [Phase 3 — EC2 Server Setup](#phase-3--ec2-server-setup)
- [Phase 4 — Jenkins CI/CD Pipeline](#phase-4--jenkins-cicd-pipeline)
- [Phase 5 — GitHub Webhook Automation](#phase-5--github-webhook-automation)
- [Problems Faced & How I Fixed Them](#problems-faced--how-i-fixed-them)
- [How to Run This Project](#how-to-run-this-project)
- [Interview Q&A](#interview-qa)
- [Cleanup](#cleanup)

---

## Project Overview

This project demonstrates a complete DevOps workflow — from writing a Flask web application to deploying it automatically on AWS using a CI/CD pipeline. Every `git push` to the main branch triggers Jenkins to automatically build and deploy the latest version of the app on an AWS EC2 instance.

**What the app does:** A simple Task Manager where users can add tasks, mark them done, and delete them. Data is stored in MySQL and persists across deployments.

**What the DevOps setup does:**
- Terraform provisions AWS infrastructure (EC2, security groups, networking) as code
- Docker containerizes the Flask app and MySQL database
- Docker Compose orchestrates both containers together
- Jenkins automates the build and deployment on every code push
- GitHub webhook triggers Jenkins automatically — no manual intervention needed

---

## Tech Stack

| Category | Tool |
|---|---|
| Application | Python Flask |
| Database | MySQL 8.0 |
| Containerization | Docker + Docker Compose |
| Infrastructure as Code | Terraform |
| CI/CD | Jenkins |
| Cloud | AWS EC2 (ap-south-1 / Mumbai) |
| Version Control | GitHub |
| OS | Ubuntu 24.04 LTS |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Developer Machine                         │
│                                                                  │
│   ┌──────────┐    git push    ┌──────────────────────────────┐  │
│   │  VS Code  │ ────────────► │         GitHub Repo          │  │
│   └──────────┘                └──────────────┬───────────────┘  │
│                                              │                   │
│   ┌──────────┐                               │ webhook trigger   │
│   │ Terraform │ ──── provisions ────►        │                   │
│   └──────────┘        AWS EC2               ▼                   │
└─────────────────────────────────────────────────────────────────┘
                                    ┌──────────────────────────────┐
                                    │        AWS EC2 Instance       │
                                    │      (ap-south-1 Mumbai)      │
                                    │                               │
                                    │  ┌─────────────────────────┐ │
                                    │  │     Jenkins Server       │ │
                                    │  │    (port 8080)           │ │
                                    │  │                          │ │
                                    │  │  Stage 1: Clone Code     │ │
                                    │  │  Stage 2: Build Image    │ │
                                    │  │  Stage 3: Docker Compose │ │
                                    │  │  Stage 4: Verify Deploy  │ │
                                    │  └───────────┬─────────────┘ │
                                    │              │ deploys        │
                                    │              ▼                │
                                    │  ┌───────────────────────┐   │
                                    │  │   Docker Network      │   │
                                    │  │   (two-tier)          │   │
                                    │  │                       │   │
                                    │  │ ┌─────────────────┐  │   │
                                    │  │ │  Flask Container │  │   │
                                    │  │ │   port 5000      │  │   │
                                    │  │ └────────┬────────┘  │   │
                                    │  │          │            │   │
                                    │  │          ▼            │   │
                                    │  │ ┌─────────────────┐  │   │
                                    │  │ │  MySQL Container │  │   │
                                    │  │ │   port 3306      │  │   │
                                    │  │ └─────────────────┘  │   │
                                    │  └───────────────────────┘   │
                                    └──────────────────────────────┘
                                              │
                                              │ public access
                                              ▼
                                    http://<EC2-IP>:5000
```

---

## Project Structure

```
flask-todo-app/
├── app.py                  # Flask application (routes, DB connection)
├── requirements.txt        # Python dependencies
├── Dockerfile              # Docker image definition for Flask app
├── docker-compose.yml      # Orchestrates Flask + MySQL containers
├── Jenkinsfile             # CI/CD pipeline definition (4 stages)
├── README.md               # This file
├── .gitignore              # Excludes .venv, terraform state, binaries
├── templates/
│   └── index.html          # Frontend (HTML + CSS + JS)
└── terraform/
    ├── main.tf             # AWS resources (EC2, security group)
    ├── variables.tf        # Input variables (region, instance type, key)
    └── outputs.tf          # Output values (public IP, SSH command)
```

---

## Phase 1 — Application & Docker Setup

### Step 1 — The Flask Application

The app is a simple Task Manager built with Flask and MySQL. It has 5 routes:

| Route | Method | Purpose |
|---|---|---|
| `/` | GET | Show all tasks |
| `/add` | POST | Add a new task |
| `/toggle/<id>` | GET | Mark task done/undone |
| `/delete/<id>` | GET | Delete a task |
| `/health` | GET | Health check endpoint |

The `/health` route is important — Docker uses it in the healthcheck to know when the container is ready.

Flask connects to MySQL using environment variables:

```python
conn = mysql.connector.connect(
    host=os.environ.get("MYSQL_HOST", "localhost"),
    user=os.environ.get("MYSQL_USER", "root"),
    password=os.environ.get("MYSQL_PASSWORD", "root"),
    database=os.environ.get("MYSQL_DB", "devops")
)
```

These environment variables are passed in by Docker Compose — keeping configuration separate from code.

---

### Step 2 — Dockerfile

```dockerfile
FROM python:3.9-slim

WORKDIR /app

RUN apt-get update && apt-get install -y gcc default-libmysqlclient-dev pkg-config && \
    rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["python", "app.py"]
```

**Key decisions:**
- `python:3.9-slim` — lightweight base image, smaller than full Python image
- `gcc` and `default-libmysqlclient-dev` — needed to compile the `mysql-connector-python` package
- `COPY requirements.txt` before `COPY . .` — Docker caches layers, so dependencies are only reinstalled when requirements.txt changes, not on every code change

---

### Step 3 — Docker Compose

```yaml
version: "3.8"

services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: "devops"
      MYSQL_ROOT_PASSWORD: "root"
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - two-tier
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-proot"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 60s

  flask:
    build:
      context: .
    ports:
      - "5000:5000"
    environment:
      - MYSQL_HOST=mysql
      - MYSQL_USER=root
      - MYSQL_PASSWORD=root
      - MYSQL_DB=devops
    networks:
      - two-tier
    depends_on:
      mysql:
        condition: service_healthy

volumes:
  mysql-data:

networks:
  two-tier:
```

**Key concepts:**

- **Docker Network (`two-tier`)** — Both containers are on the same private network. Flask connects to MySQL using the container name `mysql` as the hostname, not `localhost`. This is how Docker container networking works.

- **Healthcheck** — `depends_on: condition: service_healthy` means Flask only starts after MySQL passes its health check. Without this, Flask starts before MySQL is ready and crashes.

- **Volume (`mysql-data`)** — MySQL data is stored in a named volume, not inside the container. This means data persists even if the container is stopped, restarted, or replaced during deployment.

---

### Step 4 — Test Locally

```bash
docker compose up -d --build
docker ps  # verify both containers are running
```

Verify data is saving to MySQL:
```bash
docker exec -it mysql mysql -uroot -proot devops
SELECT * FROM tasks;
```

---

## Phase 2 — AWS & Terraform Infrastructure

### Step 5 — Create IAM User

Never use root AWS credentials for programmatic access. Create a dedicated IAM user:

1. AWS Console → IAM → Users → Create User
2. Username: `terraform-user`
3. Attach policy: `AdministratorAccess`
4. Security credentials → Create access key → CLI use case
5. Download the access key CSV

### Step 6 — Configure AWS CLI

```bash
brew install awscli
aws configure
```

Enter:
- AWS Access Key ID
- AWS Secret Access Key
- Default region: `ap-south-1` (Mumbai — closest to India)
- Output format: `json`

Verify:
```bash
aws sts get-caller-identity
```

This returns your account ID and user ARN confirming credentials work.

---

### Step 7 — Terraform Files

**`variables.tf`** — stores configurable values:

```hcl
variable "aws_region" {
  default = "ap-south-1"
}

variable "instance_type" {
  default = "t2.micro"
}

variable "key_name" {
  description = "Your EC2 key pair name"
}
```

**`main.tf`** — defines AWS resources:

```hcl
provider "aws" {
  region = var.aws_region
}

data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

resource "aws_security_group" "flask_sg" {
  name        = "flask-jenkins-sg"
  description = "Allow SSH, Jenkins, and Flask"

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "Jenkins"
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "Flask App"
    from_port   = 5000
    to_port     = 5000
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "flask_server" {
  ami                         = "ami-0f58b397bc5c1f2e8"  # Ubuntu 22.04 ap-south-1
  instance_type               = var.instance_type
  key_name                    = var.key_name
  vpc_security_group_ids      = [aws_security_group.flask_sg.id]
  subnet_id                   = tolist(data.aws_subnets.default.ids)[0]
  associate_public_ip_address = true

  root_block_device {
    volume_size = 20
  }

  tags = {
    Name = "flask-jenkins-server"
  }
}
```

**`outputs.tf`** — prints useful info after apply:

```hcl
output "ec2_public_ip" {
  value = aws_instance.flask_server.public_ip
}

output "ssh_command" {
  value = "ssh -i ~/.ssh/${var.key_name}.pem ubuntu@${aws_instance.flask_server.public_ip}"
}
```

---

### Step 8 — Create Key Pair

In AWS Console → EC2 → Key Pairs → Create:
- Name: `flask-key`
- Type: RSA
- Format: `.pem`

Move and secure it:
```bash
mv ~/Downloads/flask-key.pem ~/.ssh/
chmod 400 ~/.ssh/flask-key.pem
```

`chmod 400` is required — SSH refuses to use keys readable by others.

---

### Step 9 — Terraform Commands

```bash
cd terraform
terraform init    # downloads AWS provider plugin
terraform plan    # preview what will be created
terraform apply   # create the infrastructure
```

After apply, Terraform prints:
```
ec2_public_ip = "3.110.85.1"
ssh_command = "ssh -i ~/.ssh/flask-key.pem ubuntu@3.110.85.1"
```

---

## Phase 3 — EC2 Server Setup

### Step 10 — SSH Into EC2

```bash
ssh -i ~/.ssh/flask-key.pem ubuntu@<EC2-IP>
```

### Step 11 — Install Docker

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install docker.io docker-compose-v2 -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu
newgrp docker
```

`systemctl enable` makes Docker start automatically on reboot.
`usermod -aG docker ubuntu` lets ubuntu user run Docker without sudo.

### Step 12 — Install Jenkins

```bash
# Install Java (Jenkins requires it)
sudo apt install openjdk-17-jdk -y

# Add Jenkins GPG key and repository
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | \
  gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/jenkins.gpg > /dev/null

echo "deb [signed-by=/etc/apt/trusted.gpg.d/jenkins.gpg] \
  https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update --allow-insecure-repositories
sudo apt install jenkins -y --allow-unauthenticated

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Give Jenkins Docker permissions
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### Step 13 — Add Swap Memory

t2.micro only has 1GB RAM. Running Jenkins + Docker + MySQL together requires more:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

`/etc/fstab` makes swap survive reboots.

### Step 14 — Jenkins Initial Setup

1. Get initial admin password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

2. Open `http://<EC2-IP>:8080`
3. Paste the password
4. Click "Install suggested plugins"
5. Create an admin user
6. Save and start using Jenkins

---

## Phase 4 — Jenkins CI/CD Pipeline

### Step 15 — Jenkinsfile

```groovy
pipeline {
    agent any

    stages {
        stage('Clone Code') {
            steps {
                git branch: 'main', url: 'https://github.com/YOUR_USERNAME/flask-todo-app.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t flask-todo-app:latest .'
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                sh 'docker compose down || true'
                sh 'docker compose up -d --build'
            }
        }

        stage('Deployment Status') {
            steps {
                sh 'docker ps'
                echo 'Deployment successful! App running on port 5000.'
            }
        }
    }
}
```

**What each stage does:**

| Stage | What Happens |
|---|---|
| Clone Code | Jenkins pulls latest code from GitHub onto EC2 |
| Build Docker Image | Builds fresh Flask Docker image from Dockerfile |
| Deploy with Docker Compose | Stops old containers, starts new ones |
| Deployment Status | Confirms containers are running, prints success |

### Step 16 — Create Pipeline in Jenkins

1. Jenkins Dashboard → New Item → Pipeline → name it `flask-todo-pipeline`
2. Scroll to Pipeline section
3. Definition: `Pipeline script from SCM`
4. SCM: `Git`
5. Repository URL: `https://github.com/YOUR_USERNAME/flask-todo-app.git`
6. Branch: `*/main`
7. Script Path: `Jenkinsfile`
8. Save → Build Now

---

## Phase 5 — GitHub Webhook Automation

### Step 17 — Configure Webhook

**In GitHub:**
1. Repo Settings → Webhooks → Add webhook
2. Payload URL: `http://<EC2-IP>:8080/github-webhook/`
3. Content type: `application/json`
4. Event: "Just the push event"
5. Save

**In Jenkins:**
1. Pipeline → Configure
2. Build Triggers → check "GitHub hook trigger for GITScm polling"
3. Save

**Result:** Every `git push` to main branch now automatically triggers Jenkins to build and deploy. No manual clicking needed.

---

## Problems Faced & How I Fixed Them

These are real problems encountered during this project — not a tutorial that worked perfectly.

### Problem 1 — Port 3306 already in use locally
**Error:** `ports are not available: exposing port TCP 0.0.0.0:3306`

**Why:** Local Mac had MySQL already running on port 3306.

**Fix:** Changed docker-compose.yml to map `3307:3306` — Mac uses 3307 externally, but inside Docker containers still use 3306.

---

### Problem 2 — No default subnets in AWS account
**Error:** `No subnets found for the default VPC`

**Why:** New AWS accounts sometimes don't have default subnets created.

**Fix:**
```bash
aws ec2 create-default-subnet --availability-zone ap-south-1a
```

---

### Problem 3 — EC2 had no public IP
**Symptom:** `ec2_public_ip = ""`

**Why:** `associate_public_ip_address` was not set in Terraform.

**Fix:** Added `associate_public_ip_address = true` to `aws_instance` in `main.tf`, then ran `terraform apply`.

---

### Problem 4 — Jenkins disk full, EC2 froze
**Error:** `Usage of /: 99.8% of 6.71GB`

**Why:** Default EC2 root volume is 8GB. Docker images (MySQL ~600MB, Flask image, build cache) filled it completely.

**Fix:** Added `root_block_device { volume_size = 20 }` to Terraform and ran `terraform apply`. Then expanded the filesystem:
```bash
sudo growpart /dev/xvda 1
sudo resize2fs /dev/root
```

---

### Problem 5 — Jenkins timing out on startup
**Error:** `Job for jenkins.service failed because a timeout was exceeded`

**Why:** Default systemd timeout is 90 seconds. Jenkins on t2.micro needs ~3 minutes to start.

**Fix:**
```bash
sudo mkdir -p /etc/systemd/system/jenkins.service.d
echo -e "[Service]\nTimeoutStartSec=300" | \
  sudo tee /etc/systemd/system/jenkins.service.d/override.conf
sudo systemctl daemon-reload
sudo systemctl restart jenkins
```

---

### Problem 6 — Flask cannot connect to MySQL
**Error:** `Host '172.18.0.3' is not allowed to connect to this MySQL server`

**Why:** MySQL 8.0 by default only allows root to connect from `localhost`. Flask container has a different IP.

**Fix:**
```bash
docker exec -it mysql mysql -uroot -proot -e \
  "GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'root'; FLUSH PRIVILEGES;"
docker restart flask-app
```

---

## How to Run This Project

### Prerequisites
- Docker Desktop installed
- Terraform installed
- AWS CLI configured
- AWS account with IAM user

### Run Locally
```bash
git clone https://github.com/YOUR_USERNAME/flask-todo-app.git
cd flask-todo-app
docker compose up -d --build
# Visit http://localhost:5000
```

### Deploy to AWS
```bash
cd terraform
terraform init
terraform apply
# SSH into EC2 (use printed ssh_command)
# Install Docker + Jenkins (see Phase 3)
# Configure Jenkins pipeline (see Phase 4)
```

---


## Cleanup

To avoid AWS charges when not using the project:

```bash
cd terraform
terraform destroy
```

This deletes the EC2 instance and security group. Because infrastructure is defined as code, you can recreate everything in minutes whenever needed — just run `terraform apply` again.

---

*Built and deployed by Shlok Bam — March 2026*