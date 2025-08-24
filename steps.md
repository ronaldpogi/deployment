# 🚀 DEPLOY LARAVEL APPLICATION INTO ECS (CodePipeline / CodeBuild)

---

## 1. Amazon ECR
**Steps:**
1. Click **Create a repository**
2. Inside Create private repository:
   - Repository name: (your app name)
   - Image tag mutability: Mutable (checked)
   - Then click create

**Suggestions:**
- Use **Immutable** tags instead of mutable → safer for production (keeps old images).
- Enable **image scan on push** to detect vulnerabilities.

---

## 2. Amazon ECS Cluster
**Steps:**
1. Click **Create cluster**
2. Inside Create cluster page:
   - Cluster name: `my-dev-cluster`
   - Infrastructure: Amazon EC2 instances (checked)
   - Auto Scaling group (ASG): Create new ASG
   - Provisioning model: Spot (checked)
   - Allocation strategy: Price capacity optimized (recommended)
   - Container instance AMI: Amazon Linux 2 (kernel 5.10)
   - EC2 instance type: `t2.micro`
   - EC2 instance role: Create new role
   - Desired capacity: Minimum = 1 | Maximum = 2
   - SSH Key pair: Create new if you don’t have
   - Subnets: Select 2 subnets (public subnets) | `us-east-1b` & `us-east-1a`
   - Security group: Use an existing SG (checked)
   - Then click create

**Suggestions:**
- Prefer **Fargate** unless you specifically need EC2.
- Use **private subnets + NAT gateway** (not public subnets) for better security.

---

## 3. Task Definition
**Steps:**
1. Click **Create new task definition**
2. Inside Create new task definition:
   - Task definition family: `dev-td`
   - Launch type: Amazon EC2 instances
   - OS/Arch: Linux/x86_64
   - Network mode: bridge
   - CPU: leave empty
   - Memory: leave empty
   - Task execution role: Create new role
   - Container details:
     - Name: (your ECR name)
     - Image URI: (your ECR URI with placeholder, e.g. :IMAGE_TAG — replaced dynamically in buildspec)
     - Essential Container: Yes
     - Host port: 0
     - Container port: 8000 (Laravel default)
     - Memory hard limit: 0.5
     - Memory soft limit: 0.3
   - Then click create

**Suggestions:**
- Use **awsvpc networking mode** instead of `bridge`.
- Set **explicit CPU/Memory values** for stability.
- Use **SSM Parameter Store / Secrets Manager** for env vars instead of `.env` in S3.

---

## 4. Load Balancer
**Steps:**
1. Click **Create new load balancer**
2. Inside Create Application Load Balancer:
   - Name: `dev-alb`
   - Scheme: internet-facing (checked)
   - IP type: IPv4 (checked)
   - Availability Zones: `us-east-1a` & `us-east-1b` (checked)
   - Create target group
   - Default action: Use target group created
   - Then click create
3. Inside target group setup:
   - Choose a target type: instances
   - Name: `dev-tg`
   - Protocol/Port: HTTP:80
   - IP type: IPv4
   - Health checks: `/`
   - Then click next
4. Click Create Load balancer

**Suggestions:**
- Laravel root (`/`) may redirect or return 302/403 → create a `/health` route returning **200 OK**.
- Use **HTTPS (TLS cert in ACM)** for production.

---

## 5. ECS Service
**Steps:**
1. Go to **ECS → Clusters**
2. Click the cluster (`dev-cluster`)
3. Click **Create**
4. Inside create service:
   - Family: (select your task definition)
   - Revision: LATEST
   - Service name: `dev-service`
   - Compute options: Launch type (checked)
   - Launch type: EC2
   - Load balancing: Application Load Balancer
   - Load balancer: (your ALB)
   - Listener: Use existing (80:HTTP)
   - Target group: Use existing (`dev-tg`)
   - Then click create

**Suggestions:**
- Deployment config → Min healthy percent = 100 / Max = 200 for zero-downtime deploys.
- Enable **Service Auto Scaling** (CPU/Memory).

---

## 6. Identity and Access Management (IAM)
**Steps:**
1. Go to **IAM → Roles**
2. Click `codebuild-production-build-service-role`
3. Add permissions → Attach policies:
   - `AmazonEC2ContainerRegistryFullAccess`
   - `AmazonS3FullAccess`
4. Click create

**Suggestions:**
- Add `AmazonECSTaskExecutionRolePolicy` to ECS task role.
- Avoid `FullAccess` in production; restrict to specific repo/bucket.

---

## 7. Amazon S3
**Steps:**
1. If not available, create a bucket
2. Upload your `.env` file

**Suggestions:**
- Use **Secrets Manager or SSM Parameter Store** instead of S3 for `.env` secrets.

---

## 8. CodePipeline
**Steps:**
1. Click **Create pipeline**
2. Choose creation option: Create pipeline from template → Next
3. Pipeline settings:
   - Name: `dev-pipeline` (or prod-pipeline)
   - Execution mode: Queued (Pipeline type V2)
   - Service role: New role (first time)
   - Allow AWS CodePipeline to create resources (checked)
   - Next
4. Source stage:
   - Provider: GitHub (V2)
   - Connect to GitHub
   - Repo: (your repo)
   - Branch: (branch to deploy)
   - Output format: CodePipeline default
   - Retry on failure: checked
   - Trigger type: Push → Branch filter → (branch)
5. Build stage:
   - Provider: AWS CodeBuild
   - Project: Create new project
     - Name: `dev-build`
     - Public build access: unchecked
     - Use buildspec file: checked
   - Env variables (from buildspec):
     - AWS_DEFAULT_REGION: `us-east-1`
     - ECR_REPOSITORY: (your repo name)
     - CONTAINER_NAME: (your container name)
   - Retry on failure: checked
6. Deploy stage:
   - Provider: Amazon ECS
   - Region: `us-east-1`
   - Cluster: (select cluster)
   - Service: (select service)
   - Image definitions file: `imagedefinitions.json` (auto-generated with commit SHA as tag)
   - Next → Create pipeline

**Suggestions:**
- Add **manual approval stage** before deploy (for production).
- Use **commit SHA tags** in buildspec instead of `latest`.
- Ensure `CONTAINER_NAME` in CodeBuild matches ECS container name.

---
