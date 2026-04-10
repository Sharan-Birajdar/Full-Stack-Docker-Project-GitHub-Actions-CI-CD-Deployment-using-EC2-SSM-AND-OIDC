# Full-Stack-Docker-Project-GitHub-Actions-CI-CD-Deployment-using-EC2-SSM-AND-OIDC


## Complete Setup Guide

GitHub Actions + Docker Hub + AWS EC2 + OIDC + SSM

---

## 🧩 Components

| Component        | Technology     |
| ---------------- | -------------- |
| CI/CD Platform   | GitHub Actions |
| Containerization | Docker         |
| Image Registry   | Docker Hub     |
| Cloud Platform   | AWS EC2        |
| Authentication   | OIDC (Keyless) |
| Remote Execution | AWS SSM        |

---

# 📌 1. Prerequisites

## 1.1 Accounts & Services

* GitHub account with repository
* Docker Hub account (free tier)
* AWS account with billing enabled

---

## 1.2 Local Tools

* Git installed
* Docker Desktop installed
* AWS CLI v2 installed
* VS Code recommended

---

## 1.3 AWS EC2 Requirements

* EC2 instance (Ubuntu 22.04 / Amazon Linux 2)
* Docker installed
* AWS SSM Agent running
* IAM role with SSM permissions
* Security group open:

  * Port 80
  * Port 5000

---

# ☁️ 2. AWS Setup

## 2.1 Install Docker on EC2

```bash
sudo yum update -y
sudo yum install docker -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user
```

⚠️ Re-login after adding docker group.

---

## 2.2 Install & Start SSM Agent

```bash
sudo yum install -y amazon-ssm-agent
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
```

Check status:

```bash
sudo systemctl status amazon-ssm-agent
```

---

## 2.3 Create IAM Role for OIDC (GitHub Actions)

### Step 1: Add OIDC Provider

* IAM → Identity Providers → Add Provider
* Type: OpenID Connect
* URL: [https://token.actions.githubusercontent.com](https://token.actions.githubusercontent.com)
* Audience: sts.amazonaws.com

---

### Step 2: Create IAM Role

* IAM → Roles → Create Role
* Trusted Entity: Web Identity
* Provider: GitHub OIDC

---

### 🔐 Trust Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_GITHUB_USERNAME/YOUR_REPO:*"
        }
      }
    }
  ]
}
```

---

### Step 3: Attach Permissions

* AmazonSSMFullAccess
* AmazonEC2ReadOnlyAccess

---

# 🐳 3. Docker Hub Setup

## 3.1 Create Access Token

* Docker Hub → Account Settings → Security
* Generate Access Token
* Copy token

---

## 3.2 Create Repositories

* frontend-app
* backend-app

---

# 🔐 4. GitHub Repository Setup

## 4.1 GitHub Secrets

| Secret          | Value               |
| --------------- | ------------------- |
| DOCKER_USERNAME | Docker Hub username |
| DOCKERHUB_TOKEN | Docker Hub token    |
| AWS_ROLE_ARN    | IAM Role ARN        |
| AWS_REGION      | ap-south-1          |
| INSTANCE_ID     | EC2 Instance ID     |

---

## 4.2 Workflow File

```bash
.github/workflows/ci-cd.yml
```

---

## 📁 4.3 Project Structure

```bash
your-repo/
├── .github/workflows/ci-cd.yml
├── frontend/
│   ├── Dockerfile
│   └── code
├── backend/
│   ├── Dockerfile
│   └── code
└── README.md
```

---

# 🧱 5. Dockerfile Examples

## 5.1 Frontend (React + Nginx)

```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

## 5.2 Backend (Node.js)

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 5000
CMD ["node", "server.js"]
```

---

# 🚀 6. Run Pipeline

## 6.1 Trigger

```bash
git add .
git commit -m "trigger ci/cd pipeline"
git push origin main
```

---

## 6.2 Monitor

* GitHub → Actions tab
* Watch workflow run

---

## 6.3 Verify EC2

```bash
docker ps
docker logs frontend-app
docker logs backend-app
```

---

## 🌍 Access App

* Frontend: http://EC2_PUBLIC_IP
* Backend: http://EC2_PUBLIC_IP:5000

---

# 🛠️ 7. Troubleshooting

| Problem           | Cause              | Fix                  |
| ----------------- | ------------------ | -------------------- |
| Image not found   | Wrong tag          | Fix docker push name |
| SSM not working   | Agent stopped      | Start SSM agent      |
| OIDC failure      | Trust policy wrong | Fix repo name        |
| Port not open     | SG issue           | Open ports 80/5000   |
| Docker pull fails | No internet        | Attach public IP     |

---

## 🔍 Debug Commands

```bash
aws ssm list-commands --region YOUR_REGION
docker ps -a
docker logs frontend-app
```

---

# 🎯 Final Result

✔ Fully automated CI/CD pipeline
✔ Docker container deployment
✔ AWS SSM remote execution
✔ Secure OIDC authentication (no secrets stored)


