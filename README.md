# Full-Stack-Docker-Project-GitHub-Actions-CI-CD-Deployment-using-EC2-SSM-AND-OIDC



Full Stack Docker CI/CD Pipeline
Complete Setup Guide
GitHub Actions + Docker Hub + AWS EC2 + OIDC + SSM


Component	Technology
CI/CD Platform	GitHub Actions
Containerization	Docker
Image Registry	Docker Hub
Cloud Platform	AWS EC2
Authentication	OIDC (Keyless)
Remote Execution	AWS SSM


1. Prerequisites
Before starting, ensure the following are available and configured:

1.1 Accounts & Services
•	GitHub account with a repository containing your app code
•	Docker Hub account (free tier is sufficient)
•	AWS account with billing enabled

1.2 Local Tools
•	Git installed and configured
•	Docker Desktop installed locally for testing
•	AWS CLI v2 installed (for initial IAM setup)
•	A code editor (VS Code recommended)

1.3 AWS EC2 Requirements
•	EC2 instance running (Amazon Linux 2 or Ubuntu 22.04 recommended)
•	Docker installed on the instance
•	AWS SSM Agent installed and running
•	IAM instance profile with SSM permissions attached
•	Security group with port 80 and 5000 open to internet (0.0.0.0/0)

 
2. AWS Setup
2.1 Install Docker on EC2
SSH into your EC2 instance and run:
sudo yum update -y                          # Amazon Linux
sudo yum install docker -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user

⚠️ Note: Log out and back in after adding ec2-user to docker group for changes to take effect.

2.2 Verify SSM Agent
sudo systemctl status amazon-ssm-agent
If not running, install it:
sudo yum install -y amazon-ssm-agent
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent

2.3 Create IAM Role for OIDC (GitHub Actions)
This allows GitHub Actions to authenticate with AWS without storing static credentials.

Step 1: Add GitHub OIDC Provider
•	Go to AWS Console → IAM → Identity Providers
•	Click Add Provider
•	Select "OpenID Connect"
•	Provider URL: https://token.actions.githubusercontent.com
•	Audience: sts.amazonaws.com
•	Click Add Provider

Step 2: Create IAM Role
•	Go to IAM → Roles → Create Role
•	Select "Web identity" as trusted entity
•	Select the GitHub OIDC provider you just created
•	Audience: sts.amazonaws.com

Add the following trust policy (replace with your GitHub username and repo):
{
  "Version": "2012-10-17",
  "Statement": [{
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
  }]
}

Step 3: Attach Permissions
Attach the following AWS managed policies to the role:
•	AmazonSSMFullAccess — to send SSM commands to EC2
•	AmazonEC2ReadOnlyAccess — to describe instances (optional but useful)

💡 Tip: Name the role something memorable like github-actions-deploy-role. Copy the Role ARN — you will need it as a GitHub Secret.

 
3. Docker Hub Setup
3.1 Create Docker Hub Access Token
1.	Log in to hub.docker.com
2.	Go to Account Settings → Security → Access Tokens
3.	Click "New Access Token"
4.	Give it a name like "github-actions" and set permissions to Read/Write/Delete
5.	Copy the generated token (you will not see it again)

⚠️ Note: Store the token securely. It will be added as the DOCKERHUB_TOKEN GitHub Secret.

3.2 Create Repositories on Docker Hub
Create two public repositories on Docker Hub:
•	YOUR_USERNAME/frontend-app
•	YOUR_USERNAME/backend-app

💡 Tip: You can also let Docker Hub auto-create them on the first push — no manual creation needed for public repos.

 
4. GitHub Repository Setup
4.1 Add GitHub Secrets
Go to your GitHub repository → Settings → Secrets and variables → Actions → New repository secret.
Add all of the following secrets:

Secret Name	Value	Where to Find
DOCKER_USERNAME	Your Docker Hub username	hub.docker.com profile
DOCKERHUB_TOKEN	Docker Hub access token	Account Settings → Security
AWS_ROLE_ARN	IAM Role ARN	IAM → Roles → your role → ARN
AWS_REGION	AWS region (e.g. us-east-1)	EC2 instance region
INSTANCE_ID	EC2 instance ID	EC2 Console → instance details

4.2 Create Workflow File
Create the file .github/workflows/ci-cd.yml in your repository with the workflow code provided in the README.

4.3 Project Structure
Ensure your repository follows this structure:
your-repo/
├── .github/
│   └── workflows/
│       └── ci-cd.yml
├── frontend/
│   ├── Dockerfile
│   └── ... (your frontend code)
├── backend/
│   ├── Dockerfile
│   └── ... (your backend code)
└── README.md

 
5. Dockerfile Examples
5.1 Frontend Dockerfile (React/Nginx)
# Stage 1: Build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: Serve with Nginx
FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

5.2 Backend Dockerfile (Node.js/Express)
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 5000
CMD ["node", "server.js"]

6. Running the Pipeline
6.1 Trigger the Pipeline
Push any code change to the main branch:
git add .
git commit -m "trigger ci/cd pipeline"
git push origin main

6.2 Monitor Pipeline
•	Go to your GitHub repository
•	Click the "Actions" tab
•	Select the running workflow
•	Watch build-and-push and deploy jobs complete

6.3 Verify on EC2
After deployment completes, SSH into your EC2 and verify:
docker ps                       # Both containers should be running
docker logs frontend-app        # Check frontend logs
docker logs backend-app         # Check backend logs

Access your app:
•	Frontend: http://YOUR_EC2_PUBLIC_IP
•	Backend API: http://YOUR_EC2_PUBLIC_IP:5000

 
7. Troubleshooting

Problem	Cause	Fix
Image not found on push	Build tag doesn't match push tag	Ensure -t name matches docker push name
SSM command not executing	SSM agent not running on EC2	Run: sudo systemctl start amazon-ssm-agent
OIDC auth failure	Trust policy mismatch	Check repo name in IAM trust policy
Containers not starting	Wrong image name in SSM command	Match image names to what was built & pushed
Port not accessible	Security group blocking port	Add inbound rules for port 80 and 5000
Docker pull failing on EC2	EC2 has no internet access	Attach public IP or NAT gateway to EC2

7.1 Useful Debug Commands
# View SSM command history
aws ssm list-commands --region YOUR_REGION

# Check running containers
docker ps -a

# Force re-pull and restart
docker pull YOUR_USERNAME/frontend-app:latest
docker stop frontend-app && docker rm frontend-app
docker run -d -p 80:80 --name frontend-app YOUR_USERNAME/frontend-app:latest

Full Stack Docker CI/CD Pipeline — Setup Guide
