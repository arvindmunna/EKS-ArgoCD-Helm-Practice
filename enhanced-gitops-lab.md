# 🚀 Enhanced GitOps Practice Lab: AWS EKS + ArgoCD
## Complete End-to-End Guide with Architecture & Flow Diagrams

> **Learning Objective**: Build a production-ready GitOps pipeline from scratch, understanding each component's role in the architecture.

---

## 📋 Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Prerequisites & Tools](#prerequisites--tools)
3. [Phase 0: AWS Authentication & IAM Setup](#phase-0-aws-authentication--iam-setup)
4. [Phase 1: Container Registry Setup (ECR)](#phase-1-container-registry-setup-ecr)
5. [Phase 2: Kubernetes Cluster Creation (EKS)](#phase-2-kubernetes-cluster-creation-eks)
6. [Phase 3: Application Code Generation](#phase-3-application-code-generation)
7. [Phase 4: GitLab Repository Setup](#phase-4-gitlab-repository-setup)
8. [Phase 5: ArgoCD Installation & Configuration](#phase-5-argocd-installation--configuration)
9. [Phase 6: GitOps Update Workflow](#phase-6-gitops-update-workflow)
10. [Troubleshooting Guide](#troubleshooting-guide)

---

## 🏗️ Architecture Overview

### High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        YOUR LOCAL MACHINE                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   AWS CLI    │  │   kubectl    │  │     Git      │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
└─────────┼──────────────────┼──────────────────┼──────────────────┘
          │                  │                  │
          ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                          AWS CLOUD                               │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    Amazon ECR                               │ │
│  │  (Container Image Registry - Stores Docker Images)         │ │
│  │  practice-repo: v1, v2, v3...                              │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                   │
│                              │ Image Pull                        │
│                              ▼                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              Amazon EKS Cluster (demo-cluster)             │ │
│  │                                                            │ │
│  │  ┌──────────────────────────────────────────────────────┐ │ │
│  │  │              Control Plane (Managed by AWS)          │ │ │
│  │  │  • API Server  • Scheduler  • Controller Manager    │ │ │
│  │  └──────────────────────────────────────────────────────┘ │ │
│  │                           │                                │ │
│  │  ┌────────────────────────┼────────────────────────────┐  │ │
│  │  │           Worker Nodes (t4g.small - ARM64)          │  │ │
│  │  │                        │                             │  │ │
│  │  │  ┌──────────────┐  ┌──┴───────────┐                │  │ │
│  │  │  │   Node 1     │  │    Node 2    │                │  │ │
│  │  │  │  (2GB RAM)   │  │  (2GB RAM)   │                │  │ │
│  │  │  │              │  │              │                │  │ │
│  │  │  │ ┌──────────┐ │  │ ┌──────────┐ │                │  │ │
│  │  │  │ │ ArgoCD   │ │  │ │ Frontend │ │                │  │ │
│  │  │  │ │ Pods     │ │  │ │ App Pod  │ │                │  │ │
│  │  │  │ └──────────┘ │  │ └──────────┘ │                │  │ │
│  │  │  └──────────────┘  └──────────────┘                │  │ │
│  │  └─────────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │ Git Sync
                              │
                    ┌─────────┴─────────┐
                    │   GitLab Repo     │
                    │  (Source of Truth)│
                    │  • Helm Charts    │
                    │  • K8s Manifests  │
                    └───────────────────┘
```

### GitOps Workflow Flow

```
Developer Workflow:
┌────────────┐     ┌────────────┐     ┌────────────┐     ┌────────────┐
│ 1. Update  │────▶│ 2. Build & │────▶│ 3. Update  │────▶│ 4. Git Push│
│ index.html │     │ Push Docker│     │ values.yaml│     │ to GitLab  │
│            │     │ Image (v2) │     │ tag: v2    │     │            │
└────────────┘     └────────────┘     └────────────┘     └────────────┘
                         │                                      │
                         ▼                                      ▼
                   ┌──────────┐                          ┌──────────┐
                   │   ECR    │                          │  GitLab  │
                   │practice- │                          │   Repo   │
                   │  repo:v2 │                          │          │
                   └──────────┘                          └────┬─────┘
                         ▲                                    │
                         │                                    ▼
                         │                              ┌──────────┐
                         │                              │ ArgoCD   │
                         │                              │ Detects  │
                         │                              │ Change   │
                         │                              └────┬─────┘
                         │                                   │
                         │                                   ▼
                         │                            ┌──────────────┐
                         └────────────────────────────│ ArgoCD Syncs │
                                                      │ & Pulls v2   │
                                                      └──────┬───────┘
                                                             │
                                                             ▼
                                                      ┌─────────────┐
                                                      │ New Pod     │
                                                      │ Deployed    │
                                                      │ (Version 2) │
                                                      └─────────────┘
```

### Component Interaction Map

```
┌─────────────────────────────────────────────────────────────────┐
│                   COMPONENT RESPONSIBILITIES                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  AWS CLI ────────────▶ Manages AWS resources (ECR, EKS, IAM)    │
│                                                                  │
│  kubectl ────────────▶ Interacts with Kubernetes API            │
│                                                                  │
│  Docker ─────────────▶ Builds ARM64 images                      │
│                                                                  │
│  ECR ────────────────▶ Stores versioned container images        │
│                                                                  │
│  EKS ────────────────▶ Runs Kubernetes workloads                │
│                                                                  │
│  ArgoCD ─────────────▶ Monitors Git, syncs cluster state        │
│                                                                  │
│  GitLab ─────────────▶ Version control (single source of truth) │
│                                                                  │
│  Helm ───────────────▶ Templates Kubernetes manifests           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔧 Prerequisites & Tools

### Required Software
Install these tools on your local machine before starting:

```bash
# 1. AWS CLI (v2 recommended)
# Mac:
brew install awscli

# Linux:
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify installation
aws --version
# Expected output: aws-cli/2.x.x

# 2. kubectl (Kubernetes CLI)
# Mac:
brew install kubectl

# Linux:
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify installation
kubectl version --client
# Expected output: Client Version: v1.x.x

# 3. eksctl (EKS management tool)
# Mac:
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl

# Linux:
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Verify installation
eksctl version
# Expected output: 0.x.x

# 4. Docker Desktop (with buildx support)
# Download from: https://www.docker.com/products/docker-desktop
# After installation, verify:
docker --version
docker buildx version

# 5. Git
# Mac:
brew install git

# Linux:
sudo apt-get install git

# Verify installation
git --version
```

### AWS Account Requirements
- **Active AWS Account** with billing enabled
- **Credit card on file** (EKS and ECR have costs)
- **No service limits** blocking EKS or EC2 in `ap-south-1` region

### Estimated Costs
- **EKS Control Plane**: ~$0.10/hour ($72/month)
- **t4g.small instances (2x)**: ~$0.034/hour ($24/month total)
- **ECR Storage**: First 500MB free, then $0.10/GB/month
- **Data Transfer**: Minimal for this lab

> **💡 Cost-Saving Tip**: Delete resources immediately after completing the lab to avoid charges.

---

## 🔐 Phase 0: AWS Authentication & IAM Setup

### Understanding IAM Authentication

**What is IAM?**
- Identity and Access Management system
- Controls who (authentication) can do what (authorization) in AWS

**Why do we need this?**
- Your terminal needs proof of identity to create AWS resources
- Access keys = username + password for programmatic access

### Architecture: Authentication Flow

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│ Your Terminal│         │  AWS IAM     │         │ AWS Services │
│              │         │  Service     │         │ (EKS, ECR)   │
└──────┬───────┘         └──────┬───────┘         └──────┬───────┘
       │                        │                        │
       │ 1. aws configure       │                        │
       │ (stores credentials)   │                        │
       │◀───────────────────────┤                        │
       │                        │                        │
       │ 2. aws ecr create...   │                        │
       ├───────────────────────▶│                        │
       │                        │ 3. Validate Access Key │
       │                        │    Check Permissions   │
       │                        │◀───────────────────────┤
       │                        │ 4. Authorized ✓        │
       │                        ├───────────────────────▶│
       │                        │                        │
       │ 5. Success Response    │ 6. Create Resource     │
       │◀───────────────────────┼────────────────────────│
       │                        │                        │
```

### Step-by-Step IAM Setup

#### Step 1: Create IAM User (AWS Console)

```bash
# Navigate to AWS Console in your browser:
# https://console.aws.amazon.com/iam/

# Follow these exact clicks:
# 1. Click "Users" in left sidebar
# 2. Click "Create user" button (top right)
# 3. Enter username: gitops-admin
# 4. Click "Next"
# 5. Select "Attach policies directly"
# 6. Search for: AdministratorAccess
# 7. Check the checkbox next to AdministratorAccess
# 8. Click "Next"
# 9. Click "Create user"
```

**Understanding the Policy:**
- `AdministratorAccess` = Full permissions to all AWS services
- **Production Alternative**: Use least-privilege policies
  ```json
  Required Minimum Policies:
  - AmazonEKSClusterPolicy
  - AmazonEKSWorkerNodePolicy
  - AmazonEC2ContainerRegistryFullAccess
  - IAMFullAccess (for creating roles)
  ```

#### Step 2: Create Access Keys

```bash
# In AWS Console:
# 1. Click on "gitops-admin" user (from users list)
# 2. Click "Security credentials" tab
# 3. Scroll to "Access keys" section
# 4. Click "Create access key"
# 5. Select use case: "Command Line Interface (CLI)"
# 6. Check acknowledgment checkbox
# 7. Click "Next"
# 8. (Optional) Add description: "GitOps Lab Access"
# 9. Click "Create access key"

# ⚠️ CRITICAL: Copy both values NOW
# Access Key ID: AKIA... (example: AKIAIOSFODNN7EXAMPLE)
# Secret Access Key: wJalr... (example: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY)

# Store them temporarily in a text file - you'll need them in the next step
```

**Security Best Practices:**
- Never commit access keys to Git repositories
- Never share them in screenshots or logs
- Rotate keys every 90 days
- Delete keys when no longer needed

#### Step 3: Configure AWS CLI

```bash
# Run this command in your terminal:
aws configure

# You'll be prompted for 4 values. Enter them exactly:

# Prompt 1:
AWS Access Key ID [None]: AKIA...
# ↑ Paste your Access Key ID from previous step

# Prompt 2:
AWS Secret Access Key [None]: wJalr...
# ↑ Paste your Secret Access Key from previous step

# Prompt 3:
Default region name [None]: ap-south-1
# ↑ Mumbai region (use this for consistency with the guide)

# Prompt 4:
Default output format [None]: json
# ↑ Makes output easier to read and parse

# Configuration is stored in: ~/.aws/credentials and ~/.aws/config
```

#### Step 4: Verify Authentication

```bash
# Test 1: Check current identity
aws sts get-caller-identity

# Expected output:
# {
#     "UserId": "AIDAI...",
#     "Account": "123456789012",  ← Your 12-digit AWS Account ID
#     "Arn": "arn:aws:iam::123456789012:user/gitops-admin"
# }

# Test 2: List existing ECR repositories (should be empty initially)
aws ecr describe-repositories --region ap-south-1

# Expected output if no repos exist:
# {
#     "repositories": []
# }

# If you see this, authentication is working! ✓
```

**Troubleshooting Authentication:**
```bash
# Error: "Unable to locate credentials"
# Fix: Run aws configure again

# Error: "The security token is expired"
# Fix: Your access key might be invalid. Create a new one.

# Error: "Access Denied"
# Fix: Verify AdministratorAccess policy is attached to your IAM user
```

---

## 📦 Phase 1: Container Registry Setup (ECR)

### Understanding ECR Architecture

**What is Amazon ECR?**
- Elastic Container Registry = Docker Hub equivalent in AWS
- Stores Docker images privately
- Integrated with EKS (Kubernetes can pull images seamlessly)

**Why ECR instead of Docker Hub?**
- ✅ Private by default (secure)
- ✅ IAM-based authentication (no separate credentials)
- ✅ Low latency (same region as EKS)
- ✅ No rate limiting issues

### ECR Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    Your Local Machine                        │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Docker Build Process                                   │ │
│  │                                                         │ │
│  │  1. Dockerfile ──▶ 2. Build Image ──▶ 3. Tag Image    │ │
│  │                      (ARM64)                            │ │
│  └────────────────────────┬────────────────────────────────┘ │
│                           │                                  │
│                           │ 4. docker push                   │
└───────────────────────────┼──────────────────────────────────┘
                            │
                            ▼
            ┌───────────────────────────────┐
            │        Amazon ECR             │
            │  (ap-south-1 region)          │
            │                               │
            │  Repository: practice-repo    │
            │                               │
            │  Images:                      │
            │  ├─ <account>.dkr.ecr...:v1  │
            │  ├─ <account>.dkr.ecr...:v2  │
            │  └─ <account>.dkr.ecr...:v3  │
            └───────────────┬───────────────┘
                            │
                            │ 5. EKS pulls images
                            ▼
            ┌───────────────────────────────┐
            │      EKS Worker Nodes         │
            │  (Pull images to run pods)    │
            └───────────────────────────────┘
```

### Step-by-Step ECR Setup

#### Step 1: Create the Repository

```bash
# Create ECR repository with detailed output
aws ecr create-repository \
    --repository-name practice-repo \
    --region ap-south-1 \
    --image-scanning-configuration scanOnPush=true \
    --encryption-configuration encryptionType=AES256

# Expected Output:
# {
#     "repository": {
#         "repositoryArn": "arn:aws:ecr:ap-south-1:123456789012:repository/practice-repo",
#         "registryId": "123456789012",  ← YOUR ACCOUNT ID (save this!)
#         "repositoryName": "practice-repo",
#         "repositoryUri": "123456789012.dkr.ecr.ap-south-1.amazonaws.com/practice-repo",
#         "createdAt": "2024-01-15T10:30:00+00:00"
#     }
# }
```

**⚠️ CRITICAL: Save Your Account ID**
```bash
# Extract and save your AWS Account ID
aws sts get-caller-identity --query Account --output text

# Example output: 123456789012

# Store it in an environment variable for easy reuse:
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo $AWS_ACCOUNT_ID  # Verify it's saved

# You'll use this in image URIs like:
# 123456789012.dkr.ecr.ap-south-1.amazonaws.com/practice-repo:v1
```

#### Step 2: Understanding Image URIs

**ECR Image URI Structure:**
```
[ACCOUNT_ID].dkr.ecr.[REGION].amazonaws.com/[REPO_NAME]:[TAG]
│           │       │         │              │            │
│           │       │         │              │            └─ Version (v1, v2, latest)
│           │       │         │              └─────────────── Repository name
│           │       │         └────────────────────────────── AWS region
│           │       └──────────────────────────────────────── ECR service domain
│           └──────────────────────────────────────────────── Docker registry
└──────────────────────────────────────────────────────────── Your AWS Account ID

Example:
123456789012.dkr.ecr.ap-south-1.amazonaws.com/practice-repo:v1
```

#### Step 3: Verify Repository Creation

```bash
# List all ECR repositories in your region
aws ecr describe-repositories --region ap-south-1

# Expected output:
# {
#     "repositories": [
#         {
#             "repositoryName": "practice-repo",
#             "repositoryUri": "123456789012.dkr.ecr.ap-south-1.amazonaws.com/practice-repo"
#         }
#     ]
# }

# Check repository details
aws ecr describe-repositories \
    --repository-names practice-repo \
    --region ap-south-1

# Verify no images exist yet (we haven't pushed any)
aws ecr list-images \
    --repository-name practice-repo \
    --region ap-south-1

# Expected output:
# {
#     "imageIds": []  ← Empty, as expected
# }
```

**Understanding Repository Settings:**
- `scanOnPush=true`: Automatically scans images for vulnerabilities
- `encryptionType=AES256`: Encrypts images at rest
- Images are private by default (IAM authentication required)

---

## ⚙️ Phase 2: Kubernetes Cluster Creation (EKS)

### Understanding EKS Architecture

**What is Amazon EKS?**
- Elastic Kubernetes Service = Managed Kubernetes
- AWS manages the control plane (API server, scheduler, etc.)
- You manage worker nodes (EC2 instances)

**Why EKS over Self-Managed Kubernetes?**
- ✅ AWS handles control plane upgrades
- ✅ High availability built-in
- ✅ Integrates with AWS services (ECR, IAM, VPC)
- ✅ No need to maintain etcd or manage certificates

### EKS Cluster Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│                         AWS EKS Cluster                           │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │              Control Plane (Managed by AWS)                  │ │
│  │  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐  │ │
│  │  │ API Server  │  │  Scheduler   │  │ Controller Manager│  │ │
│  │  │ (endpoint)  │  │ (pod placing)│  │ (reconciliation)  │  │ │
│  │  └─────────────┘  └──────────────┘  └───────────────────┘  │ │
│  │         │                │                     │            │ │
│  │         └────────────────┼─────────────────────┘            │ │
│  └──────────────────────────┼───────────────────────────────────┘ │
│                             │                                     │
│                             ▼                                     │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │                     Data Plane (Your Nodes)                  │ │
│  │                                                              │ │
│  │  ┌─────────────────────────┐  ┌─────────────────────────┐  │ │
│  │  │   Worker Node 1         │  │   Worker Node 2         │  │ │
│  │  │   (t4g.small - ARM64)   │  │   (t4g.small - ARM64)   │  │ │
│  │  │   - 2 vCPU              │  │   - 2 vCPU              │  │ │
│  │  │   - 2 GB RAM            │  │   - 2 GB RAM            │  │ │
│  │  │                         │  │                         │  │ │
│  │  │  ┌──────────────────┐   │  │  ┌──────────────────┐   │  │ │
│  │  │  │  kubelet         │   │  │  │  kubelet         │   │  │ │
│  │  │  │  (node agent)    │   │  │  │  (node agent)    │   │  │ │
│  │  │  └──────────────────┘   │  │  └──────────────────┘   │  │ │
│  │  │  ┌──────────────────┐   │  │  ┌──────────────────┐   │  │ │
│  │  │  │  Container       │   │  │  │  Container       │   │  │ │
│  │  │  │  Runtime         │   │  │  │  Runtime         │   │  │ │
│  │  │  │  (containerd)    │   │  │  │  (containerd)    │   │  │ │
│  │  │  └──────────────────┘   │  │  └──────────────────┘   │  │ │
│  │  │                         │  │                         │  │ │
│  │  │  Pods:                  │  │  Pods:                  │  │ │
│  │  │  ├─ kube-proxy         │  │  ├─ coredns            │  │ │
│  │  │  ├─ aws-node (CNI)     │  │  ├─ frontend-app       │  │ │
│  │  │  └─ argocd-pods        │  │  └─ argocd-pods        │  │ │
│  │  └─────────────────────────┘  └─────────────────────────┘  │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                    │
│  Underlying Infrastructure:                                       │
│  ├─ VPC (10.0.0.0/16)                                            │
│  ├─ Public Subnets (2 AZs)                                       │
│  ├─ Private Subnets (2 AZs)                                      │
│  ├─ Internet Gateway                                             │
│  ├─ NAT Gateway                                                  │
│  └─ Security Groups                                              │
│                                                                    │
└───────────────────────────────────────────────────────────────────┘
```

### Step-by-Step EKS Setup

#### Step 1: Create the Cluster with eksctl

```bash
# This single command creates:
# - EKS control plane
# - 2 t4g.small worker nodes (ARM64/Graviton)
# - Complete VPC with subnets, route tables, IGW, NAT
# - IAM roles and policies
# - Node groups with auto-scaling configuration

eksctl create cluster \
  --name demo-cluster \
  --region ap-south-1 \
  --node-type t4g.small \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 4 \
  --with-oidc \
  --managed

# ⏱️ This will take 15-20 minutes. Output will show:
# [ℹ]  eksctl version 0.x.x
# [ℹ]  using region ap-south-1
# [ℹ]  setting availability zones to [ap-south-1a ap-south-1b]
# [ℹ]  subnets for ap-south-1a - public:10.0.0.0/24 private:10.0.128.0/24
# [ℹ]  subnets for ap-south-1b - public:10.0.1.0/24 private:10.0.129.0/24
# [ℹ]  creating EKS cluster "demo-cluster" in "ap-south-1" region
# [ℹ]  will create 2 separate CloudFormation stacks...
# ...
# [✔]  EKS cluster "demo-cluster" in "ap-south-1" region is ready
```

**Understanding the Flags:**
- `--name demo-cluster`: Your cluster identifier
- `--region ap-south-1`: Mumbai region (lower latency for India)
- `--node-type t4g.small`: ARM64 instance type
  - 2 vCPUs
  - 2 GB RAM
  - ~$0.017/hour per instance
  - **Why ARM64?** Lower cost, better performance per dollar
- `--nodes 2`: Initial node count
- `--nodes-min 2`: Minimum nodes (for high availability)
- `--nodes-max 4`: Maximum nodes (for auto-scaling)
- `--with-oidc`: Enables IAM roles for service accounts (IRSA)
- `--managed`: Uses EKS managed node groups (AWS handles updates)

#### Step 2: Configure kubectl Access

```bash
# This command updates ~/.kube/config with cluster credentials
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name demo-cluster

# Expected output:
# Added new context arn:aws:eks:ap-south-1:123456789012:cluster/demo-cluster to /home/user/.kube/config

# What this does:
# 1. Retrieves cluster endpoint from EKS API
# 2. Gets certificate authority data
# 3. Configures AWS IAM authentication
# 4. Sets demo-cluster as current context
```

**Understanding kubeconfig:**
```yaml
# Location: ~/.kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRU...  # CA cert
    server: https://ABCD1234.gr7.ap-south-1.eks.amazonaws.com  # API endpoint
  name: arn:aws:eks:ap-south-1:123456789012:cluster/demo-cluster
contexts:
- context:
    cluster: arn:aws:eks:ap-south-1:123456789012:cluster/demo-cluster
    user: arn:aws:eks:ap-south-1:123456789012:cluster/demo-cluster
  name: arn:aws:eks:ap-south-1:123456789012:cluster/demo-cluster
current-context: arn:aws:eks:ap-south-1:123456789012:cluster/demo-cluster
users:
- name: arn:aws:eks:ap-south-1:123456789012:cluster/demo-cluster
  user:
    exec:
      command: aws  # Uses AWS CLI for authentication
      args:
        - eks
        - get-token
        - --cluster-name
        - demo-cluster
```

#### Step 3: Add Access Entry (CRITICAL)

**Why is this step needed?**
In December 2023, AWS changed EKS authentication:
- **Old method**: ConfigMap-based authentication (aws-auth)
- **New method**: Access Entries (API-based)
- Your IAM user needs explicit cluster access

**Manual Method (AWS Console):**
```bash
# 1. Open AWS Console: https://console.aws.amazon.com/eks/
# 2. Select region: ap-south-1 (top-right dropdown)
# 3. Click "Clusters" in left sidebar
# 4. Click "demo-cluster"
# 5. Click "Access" tab
# 6. Under "Access entries", click "Create access entry"
# 7. IAM principal: arn:aws:iam::123456789012:user/gitops-admin
#    (or search and select your user)
# 8. Type: Standard
# 9. Click "Next"
# 10. Access policy: AmazonEKSClusterAdminPolicy
# 11. Access scope: Cluster (default)
# 12. Click "Next"
# 13. Review and click "Create"
```

**Automated Method (CLI):**
```bash
# Create access entry for your IAM user
aws eks create-access-entry \
  --cluster-name demo-cluster \
  --principal-arn arn:aws:iam::${AWS_ACCOUNT_ID}:user/gitops-admin \
  --type STANDARD \
  --region ap-south-1

# Associate admin policy
aws eks associate-access-policy \
  --cluster-name demo-cluster \
  --principal-arn arn:aws:iam::${AWS_ACCOUNT_ID}:user/gitops-admin \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster \
  --region ap-south-1

# Verify access entry
aws eks list-access-entries \
  --cluster-name demo-cluster \
  --region ap-south-1
```

#### Step 4: Verify Cluster Access

```bash
# Test 1: Check cluster info
kubectl cluster-info

# Expected output:
# Kubernetes control plane is running at https://ABCD1234.gr7.ap-south-1.eks.amazonaws.com
# CoreDNS is running at https://ABCD1234.gr7.ap-south-1.eks.amazonaws.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

# Test 2: List nodes (should show 2 nodes)
kubectl get nodes

# Expected output:
# NAME                                           STATUS   ROLES    AGE   VERSION
# ip-10-0-1-234.ap-south-1.compute.internal      Ready    <none>   5m    v1.28.3-eks-xyz
# ip-10-0-2-135.ap-south-1.compute.internal      Ready    <none>   5m    v1.28.3-eks-xyz

# Test 3: Check node architecture (should be ARM64)
kubectl get nodes -o wide

# Look for: INTERNAL-IP, ARCHITECTURE columns
# ARCHITECTURE should show: arm64

# Test 4: List all pods in all namespaces
kubectl get pods -A

# Expected output (system pods):
# NAMESPACE     NAME                       READY   STATUS    RESTARTS   AGE
# kube-system   aws-node-xxxxx             1/1     Running   0          6m
# kube-system   aws-node-yyyyy             1/1     Running   0          6m
# kube-system   coredns-xxxxx              1/1     Running   0          12m
# kube-system   coredns-yyyyy              1/1     Running   0          12m
# kube-system   kube-proxy-xxxxx           1/1     Running   0          6m
# kube-system   kube-proxy-yyyyy           1/1     Running   0          6m

# Test 5: Check cluster resources
kubectl get all -A

# Test 6: Verify you have admin permissions
kubectl auth can-i '*' '*'
# Expected output: yes

# Test 7: Check node capacity
kubectl describe nodes | grep -A 5 "Capacity:"
# Should show:
#   cpu:                2
#   memory:             2048Mi (approximately)
```

**Understanding System Pods:**
- `aws-node`: AWS VPC CNI plugin (networking)
- `coredns`: DNS server for service discovery
- `kube-proxy`: Network proxy on each node

---

## 💻 Phase 3: Application Code Generation

### Application Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  Application Structure                       │
│                                                              │
│  frontend-app/                                              │
│  ├── app-source/               ← Application code           │
│  │   ├── index.html           ← Custom HTML file           │
│  │   └── Dockerfile           ← Container build recipe     │
│  │                                                          │
│  ├── helm-chart/              ← Kubernetes deployment      │
│  │   ├── Chart.yaml          ← Helm metadata              │
│  │   ├── values.yaml         ← Configuration values       │
│  │   └── templates/          ← K8s manifest templates     │
│  │       ├── deployment.yaml ← Pod specification          │
│  │       └── service.yaml    ← Network exposure           │
│  │                                                          │
│  ├── argocd/                  ← GitOps configuration       │
│  │   └── application.yaml    ← ArgoCD app definition      │
│  │                                                          │
│  └── README.md                ← Instructions               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Understanding Each Component

**Dockerfile (Container Image Recipe):**
```dockerfile
# Base image: ARM64-compatible Nginx
FROM arm64v8/nginx:alpine

# Why arm64v8/nginx:alpine?
# - arm64v8: Runs on AWS Graviton (t4g.small) processors
# - nginx: Lightweight web server
# - alpine: Minimal Linux distro (5MB vs 100MB+)

# Copy custom HTML into Nginx's default serving directory
COPY index.html /usr/share/nginx/html/

# Nginx listens on port 80 by default
EXPOSE 80

# No CMD needed - inherits from base image
```

**index.html (Application Content):**
```html
<!DOCTYPE html>
<html>
<head>
    <title>GitOps Lab</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
        }
        .container {
            text-align: center;
            background: rgba(255, 255, 255, 0.1);
            padding: 50px;
            border-radius: 15px;
            box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
        }
        h1 { font-size: 3em; margin: 0; }
        p { font-size: 1.5em; margin-top: 20px; }
        .version { 
            font-size: 0.8em; 
            opacity: 0.8; 
            margin-top: 30px;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>🚀 GitOps Lab</h1>
        <p>Hello World!</p>
        <p>Frontend Deployment Successful</p>
        <div class="version">Version 1</div>
    </div>
</body>
</html>
```

**Helm Chart.yaml (Chart Metadata):**
```yaml
apiVersion: v2
name: frontend-app
description: A simple frontend application for GitOps practice
type: application
version: 1.0.0        # Chart version (not app version)
appVersion: "1.0.0"   # Application version
```

**Helm values.yaml (Configuration):**
```yaml
# This file contains all configurable values
# Helm templates reference these using {{ .Values.xxx }}

replicaCount: 2  # Number of pod replicas (for high availability)

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/practice-repo
  tag: v1  # ← THIS IS WHAT YOU'LL CHANGE for v2, v3, etc.
  pullPolicy: IfNotPresent  # Don't pull if image exists locally

service:
  type: LoadBalancer  # Exposes app to internet via AWS ELB
  port: 80           # External port
  targetPort: 80     # Container port (Nginx default)

resources:
  # Resource limits (CRITICAL for 2GB RAM nodes)
  requests:
    cpu: 100m        # 0.1 CPU cores
    memory: 64Mi     # 64 megabytes
  limits:
    cpu: 200m        # Max 0.2 CPU cores
    memory: 128Mi    # Max 128 megabytes
```

**Helm deployment.yaml Template:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name }}  # Replaced with "frontend-app"
spec:
  replicas: {{ .Values.replicaCount }}  # Replaced with 2
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
      - name: nginx
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        # Expands to: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/practice-repo:v1
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: {{ .Values.service.targetPort }}
        resources:
          requests:
            cpu: {{ .Values.resources.requests.cpu }}
            memory: {{ .Values.resources.requests.memory }}
          limits:
            cpu: {{ .Values.resources.limits.cpu }}
            memory: {{ .Values.resources.limits.memory }}
```

**Helm service.yaml Template:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Chart.Name }}-service
spec:
  type: {{ .Values.service.type }}  # LoadBalancer
  selector:
    app: {{ .Chart.Name }}  # Routes traffic to pods with this label
  ports:
  - protocol: TCP
    port: {{ .Values.service.port }}        # External: 80
    targetPort: {{ .Values.service.targetPort }}  # Container: 80
```

**ArgoCD application.yaml:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frontend-app
  namespace: argocd  # ArgoCD lives here
spec:
  project: default
  
  source:
    repoURL: https://gitlab.com/YOUR_USERNAME/gitops-repo.git
    # ↑ This is your GitLab repository (single source of truth)
    targetRevision: main  # Git branch to monitor
    path: helm-chart      # Folder containing Helm charts
  
  destination:
    server: https://kubernetes.default.svc  # Local cluster
    namespace: default  # Where to deploy the app
  
  syncPolicy:
    automated:
      prune: true     # Delete resources if removed from Git
      selfHeal: true  # Revert manual changes
    syncOptions:
    - CreateNamespace=true  # Create namespace if it doesn't exist
```

### Generate All Files with AI

**Fill in YOUR values:**  and run this in your terminal before running the prompt.
```bash
# Set your variables first:
export AWS_ACCOUNT_ID="123456789012"  # Your 12-digit AWS Account ID
export AWS_REGION="ap-south-1"
export ECR_REPO_NAME="practice-repo"
export EKS_CLUSTER_NAME="demo-cluster"
export GITLAB_USERNAME="your-gitlab-username"  # Replace with your GitLab username
```

**Copy this entire prompt to ChatGPT/Claude:**

```
Role: You are a Senior DevOps Engineer.

Task: Write a single, executable bash script named generate-gitops-zip.sh. When executed, this script must create a local directory structure for a Kubernetes GitOps project, populate it with Helm and ArgoCD manifests, create an HTML file, create a Dockerfile, generate a README guide, and compress the entire folder into a .zip file.

Strict Variable Injection: You MUST use the exact user variables provided below to pre-fill the generated files. Do NOT use generic placeholders like <YOUR_ACCOUNT_ID>.

Variables:
- AWS Account ID: ${AWS_ACCOUNT_ID}
- AWS Region: ${AWS_REGION}
- ECR Repository Name: ${ECR_REPO_NAME}
- EKS Cluster Name: ${EKS_CLUSTER_NAME}
- GitLab Username: ${GITLAB_USERNAME}

In values.yaml, construct the container image URI using: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}
Set the default image tag to v1.

In the ArgoCD manifest, construct the repoURL using: https://gitlab.com/${GITLAB_USERNAME}/gitops-repo.git

Strict Architectural Constraints:

Compute (ARM64/Graviton): The target environment is an AWS EKS cluster running t4g.small instances (AWS Graviton / ARM64 architecture). Add comments instructing the user to build the image using docker buildx build --platform linux/arm64.

Memory Limits: The nodes only have 2GB of RAM. You MUST NOT include Istio, Envoy sidecars, HashiCorp Vault, or any heavy service mesh/secret managers. Set resource requests to cpu: 100m, memory: 64Mi and limits to cpu: 200m, memory: 128Mi.

Networking (No Domain): Do NOT generate Ingress resources or Gateways. Use a standard Kubernetes Service manifest with type: LoadBalancer.

Application (Frontend ONLY): A single-tier application: an Nginx frontend (exposed on port 80).

The script must generate a custom index.html file locally in an app-source folder that says "GitOps Lab: Hello World! Frontend Deployment Successful (Version 1)" with nice CSS styling.

Generate a Dockerfile that uses an ARM64 Nginx base image (arm64v8/nginx:alpine) and copies this custom index.html into /usr/share/nginx/html.

Documentation Requirement (The Update Loop):

The script MUST generate a README.md file in the root directory.

This README must explain exactly what to do after the cluster is running to practice the GitOps lifecycle.

It must clearly instruct the user to:
1) Change the text in index.html to Version 2
2) Build and push a new Docker image tagged as v2
3) Open the values.yaml file and change the image tag from v1 to v2
4) Commit and push the changes to GitLab
5) Watch ArgoCD automatically sync the new version

Output Requirements:

Provide ONLY the bash script using cat << 'EOF' > filepath.
The script must generate: Chart.yaml, values.yaml, deployment.yaml, service.yaml, an ArgoCD Application manifest, the custom index.html, the Dockerfile, and the README.md.
The script must end with a command to zip the folder.
```

**Execute the generated script:**
```bash
# Save the AI output to a file
nano generate-gitops-zip.sh

# Paste the script, then save (Ctrl+O, Enter, Ctrl+X)

# Make it executable
chmod +x generate-gitops-zip.sh

# Run it
./generate-gitops-zip.sh

# Unzip the result
unzip frontend-app.zip
cd frontend-app

# Verify the structure
tree .
# Should show:
# .
# ├── app-source/
# │   ├── Dockerfile
# │   └── index.html
# ├── helm-chart/
# │   ├── Chart.yaml
# │   ├── values.yaml
# │   └── templates/
# │       ├── deployment.yaml
# │       └── service.yaml
# ├── argocd/
# │   └── application.yaml
# └── README.md
```

---

## 🔗 Phase 4: GitLab Repository Setup

### Understanding Git in GitOps

**Why GitLab?**
- Git repository = Single source of truth
- All changes go through Git (version controlled)
- ArgoCD monitors Git for changes
- Rollback = revert Git commit

**GitOps Flow:**
```
Developer → Git Commit → GitLab → ArgoCD Detection → Cluster Sync
```

### Step-by-Step GitLab Setup

#### Step 1: Create GitLab Account (if needed)

```bash
# Visit: https://gitlab.com/users/sign_up
# Enter email, username, password
# Verify email
# Complete profile setup
```

#### Step 2: Create New Repository

```bash
# In GitLab web interface:
# 1. Click "+" icon (top-right)
# 2. Select "New project/repository"
# 3. Click "Create blank project"
# 4. Project name: gitops-repo
# 5. Visibility: Private (recommended)
# 6. Initialize with README: Uncheck (we have our own files)
# 7. Click "Create project"
```

#### Step 3: Clone and Push Code

```bash
# Navigate to your frontend-app folder
cd frontend-app

# Initialize Git repository
git init

# Add all files
git add .

# Commit with descriptive message
git commit -m "Initial commit: v1 GitOps setup"

# Add GitLab as remote origin (REPLACE with your username)
git remote add origin https://gitlab.com/YOUR_USERNAME/gitops-repo.git

# Push to GitLab
git push -u origin main

# If prompted for credentials:
# Username: your-gitlab-username
# Password: use Personal Access Token (not your GitLab password)
```

#### Step 4: Create GitLab Personal Access Token

```bash
# In GitLab web interface:
# 1. Click your avatar (top-right)
# 2. Settings → Access Tokens
# 3. Click "Add new token"
# 4. Token name: argocd-access
# 5. Expiration: Set to 1 year from now
# 6. Scopes: Check "read_repository"
# 7. Click "Create personal access token"
# 8. Copy the token (starts with glpat-...)
# 9. Store it securely - you'll need it for ArgoCD
```

#### Step 5: Verify Repository

```bash
# View your repository in browser:
# https://gitlab.com/YOUR_USERNAME/gitops-repo

# Should show:
# - app-source/
#   - Dockerfile
#   - index.html
# - helm-chart/
#   - Chart.yaml
#   - values.yaml
#   - templates/
#     - deployment.yaml
#     - service.yaml
# - argocd/
#   - application.yaml
# - README.md

# Verify Git history
git log

# Should show your "Initial commit: v1 GitOps setup"
```

---

## 🔄 Phase 5: ArgoCD Installation & Configuration

### Understanding ArgoCD Architecture

**What is ArgoCD?**
- GitOps continuous delivery tool
- Monitors Git repositories for changes
- Automatically syncs Kubernetes cluster state with Git
- Provides visual dashboard for deployments

**ArgoCD Components:**
```
┌─────────────────────────────────────────────────────────┐
│                  ArgoCD Architecture                     │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │           ArgoCD Application Controller            │ │
│  │  (Monitors Git, reconciles cluster state)          │ │
│  └────────┬──────────────────────────┬─────────────────┘ │
│           │                          │                   │
│           ▼                          ▼                   │
│  ┌─────────────────┐      ┌──────────────────┐         │
│  │  Git Repository │      │ Kubernetes API   │         │
│  │  (Source)       │      │ (Destination)    │         │
│  └─────────────────┘      └──────────────────┘         │
│           │                          │                   │
│           ▼                          ▼                   │
│  ┌─────────────────┐      ┌──────────────────┐         │
│  │ Helm Charts     │──────▶│ Deployed Pods   │         │
│  │ (Desired State) │      │ (Actual State)   │         │
│  └─────────────────┘      └──────────────────┘         │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │              ArgoCD Web UI                         │ │
│  │  (Dashboard to visualize deployments)              │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Step-by-Step ArgoCD Installation

#### Step 1: Install ArgoCD

```bash
# Create ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD (official stable release)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# This creates multiple resources:
# - CustomResourceDefinitions (CRDs)
# - ServiceAccounts
# - Roles and RoleBindings
# - ConfigMaps
# - Secrets
# - Deployments (argocd-server, argocd-repo-server, argocd-application-controller)
# - Services

# Wait for all pods to be running (takes 2-3 minutes)
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s

# Verify installation
kubectl get pods -n argocd

# Expected output:
# NAME                                  READY   STATUS    RESTARTS   AGE
# argocd-application-controller-xxx     1/1     Running   0          2m
# argocd-dex-server-xxx                 1/1     Running   0          2m
# argocd-redis-xxx                      1/1     Running   0          2m
# argocd-repo-server-xxx                1/1     Running   0          2m
# argocd-server-xxx                     1/1     Running   0          2m
```

#### Step 2: Access ArgoCD UI

```bash
# Method 1: Port Forwarding (Recommended for this lab)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Keep this terminal open
# Access UI at: https://localhost:8080
# Note: Browser will show security warning (self-signed cert) - click "Advanced" → "Proceed"

Method 2: The "Direct" Way (Using NodePort)
Since your EKS nodes already have Public IPs (as seen in your kubectl get nodes output), you can expose ArgoCD directly on those IPs.

1. Change Service to NodePort
Run this command to change the service type:

Bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

2. Find the Assigned Port
Kubernetes will assign a random port (between 30000–32767).


Find it by running:
Bash
kubectl get svc argocd-server -n argocd
Look for the PORT(S) column. It will look like 443:3xxxx/TCP. Note the 3xxxx number.

3. Update AWS Security Group
You must allow traffic to your EKS Worker Nodes (not your Ubuntu EC2) on that 3xxxx port:

Go to EC2 Console > Instances.

Select one of your EKS nodes (e.g., ip-192-168-12-28...).

Click Security > Security Groups.

Add an Inbound Rule: Custom TCP, Port: 3xxxx (the one from step 2), Source: 0.0.0.0/0.

4. Access the UI
In your browser, go to:
https://13.200.205.255:3xxxx (Use either of the External IPs from your node list).

# Method 3: LoadBalancer (For production)
# kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
# Wait for external IP, then access via that IP
```

#### Step 3: Get Initial Admin Password

```bash
# ArgoCD generates a random initial password stored in a secret
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Example output: xK9fG3mP2nQ7vR1s
# Copy this password

# Login credentials:
# Username: admin
# Password: <the password from above>
```

#### Step 4: Login via ArgoCD UI

```bash
# 1. Open browser: https://localhost:8080
# 2. Click "Advanced" → "Proceed to localhost" (ignore cert warning)
# 3. Username: admin
# 4. Password: <paste password from Step 3>
# 5. Click "SIGN IN"

# You should now see the ArgoCD dashboard (empty initially)
```

#### Step 5: Configure Git Repository Connection

```bash
# Option A: Via ArgoCD UI
# 1. Click "Settings" (gear icon, left sidebar)
# 2. Click "Repositories"
# 3. Click "Connect Repo"
# 4. Connection method: HTTPS
# 5. Repository URL: https://gitlab.com/YOUR_USERNAME/gitops-repo.git
# 6. Username: YOUR_GITLAB_USERNAME
# 7. Password: <GitLab Personal Access Token from Phase 4>
# 8. Click "CONNECT"
# 9. Status should show "Successful"

# Option B: Via ArgoCD CLI (optional)
# Install ArgoCD CLI first:
# Mac: brew install argocd
# Linux: curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
# chmod +x /usr/local/bin/argocd

# Login to ArgoCD
argocd login localhost:8080 --username admin --password <password> --insecure

# Add repository
argocd repo add https://gitlab.com/YOUR_USERNAME/gitops-repo.git \
  --username YOUR_GITLAB_USERNAME \
  --password <GitLab Token>
```

#### Step 6: Build and Push Docker Image (v1)

```bash
# Navigate to app-source directory
cd app-source

# Login to ECR
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin \
  ${AWS_ACCOUNT_ID}.dkr.ecr.ap-south-1.amazonaws.com

# Build ARM64 image
docker buildx build \
  --platform linux/arm64 \
  -t ${AWS_ACCOUNT_ID}.dkr.ecr.ap-south-1.amazonaws.com/practice-repo:v1 \
  --push \
  .

# Expected output:
# [+] Building 45.2s (7/7) FINISHED
# => [internal] load build definition from Dockerfile
# => => transferring dockerfile: 123B
# => [internal] load .dockerignore
# => [1/2] FROM docker.io/arm64v8/nginx:alpine
# => [2/2] COPY index.html /usr/share/nginx/html/
# => exporting to image
# => => pushing layers
# => => pushing manifest for xxx.dkr.ecr.ap-south-1.amazonaws.com/practice-repo:v1

# Verify image in ECR
aws ecr list-images --repository-name practice-repo --region ap-south-1

# Expected output:
# {
#     "imageIds": [
#         {
#             "imageDigest": "sha256:abc123...",
#             "imageTag": "v1"
#         }
#     ]
# }
```

**Understanding the Build Command:**
- `docker buildx`: Multi-platform build tool
- `--platform linux/arm64`: Target architecture (matches t4g.small)
- `-t`: Tag the image
- `--push`: Push to registry immediately after build
- `.`: Build context (current directory)

#### Step 7: Deploy Application via ArgoCD

```bash
# Navigate back to frontend-app root
cd ..

# Apply ArgoCD Application manifest
kubectl apply -f argocd/application.yaml

# Verify application is created
kubectl get applications -n argocd

# Expected output:
# NAME           SYNC STATUS   HEALTH STATUS
# frontend-app   OutOfSync     Missing

# Watch ArgoCD sync the application
kubectl get applications -n argocd -w
# (Press Ctrl+C to stop watching)

# After a few seconds, status should change to:
# NAME           SYNC STATUS   HEALTH STATUS
# frontend-app   Synced        Healthy
```

#### Step 8: Verify Deployment

```bash
# Check if pods are running
kubectl get pods

# Expected output:
# NAME                            READY   STATUS    RESTARTS   AGE
# frontend-app-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
# frontend-app-xxxxxxxxxx-yyyyy   1/1     Running   0          2m

# Check service
kubectl get svc

# Expected output:
# NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP                                                               PORT(S)        AGE
# frontend-app-service   LoadBalancer   10.100.123.45   a1b2c3d4e5f6789.ap-south-1.elb.amazonaws.com                            80:31234/TCP   3m

# Get LoadBalancer URL (takes 2-3 minutes to provision)
kubectl get svc frontend-app-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# Example output: a1b2c3d4e5f6789.ap-south-1.elb.amazonaws.com

# Test the application
curl http://$(kubectl get svc frontend-app-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Or open in browser:
# http://a1b2c3d4e5f6789.ap-south-1.elb.amazonaws.com

# You should see: "GitOps Lab: Hello World! Frontend Deployment Successful (Version 1)"
```

#### Step 9: View in ArgoCD UI

```bash
# In ArgoCD UI (https://localhost:8080):
# 1. You should see "frontend-app" application card
# 2. Status: Synced (green checkmark)
# 3. Health: Healthy (heart icon)
# 4. Click on the application to see detailed view

# Detailed view shows:
# - Application tree (Deployment → ReplicaSet → Pods)
# - Service (LoadBalancer)
# - Resource health status
# - Git commit SHA
# - Sync status
```

---

## 🔄 Phase 6: GitOps Update Workflow

### Understanding the GitOps Loop

```
 ┌─────────────────────────────────────────────────────────┐
 │                    GitOps Cycle                          │
 │                                                          │
 │  1. Developer makes changes                             │
 │     ├─ Edit index.html (Version 1 → Version 2)         │
 │     └─ Build new Docker image (v1 → v2)                │
 │                                                          │
 │  2. Update configuration in Git                         │
 │     └─ Change values.yaml (tag: v1 → tag: v2)          │
 │                                                          │
 │  3. Git commit + push                                   │
 │     └─ Single source of truth updated                   │
 │                                                          │
 │  4. ArgoCD detects change                               │
 │     ├─ Polls Git repository every 3 minutes             │
 │     └─ Compares Git state vs Cluster state              │
 │                                                          │
 │  5. ArgoCD syncs cluster                                │
 │     ├─ Pulls new Helm values                            │
 │     ├─ Generates new Kubernetes manifests               │
 │     └─ Applies changes to cluster                       │
 │                                                          │
 │  6. Kubernetes rolls out update                         │
 │     ├─ Pulls new image from ECR                         │
 │     ├─ Creates new pods with v2                         │
 │     ├─ Terminates old pods with v1                      │
 │     └─ Zero-downtime deployment ✓                       │
 │                                                          │
 └─────────────────────────────────────────────────────────┘
```

### Step-by-Step Update Process

#### Step 1: Modify Application Code

```bash
# Edit index.html
cd app-source
nano index.html

# Find this line:
#   <div class="version">Version 1</div>

# Change it to:
#   <div class="version">Version 2 - Updated via GitOps!</div>

# Optionally, change other content:
#   <p>Hello World!</p>
# to:
#   <p>Hello World! This is an update!</p>

# Save: Ctrl+O, Enter, Ctrl+X
```

#### Step 2: Build and Push New Image (v2)

```bash
# Still in app-source directory

# Login to ECR (if not already logged in)
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin \
  ${AWS_ACCOUNT_ID}.dkr.ecr.ap-south-1.amazonaws.com

# Build and push v2 image
docker buildx build \
  --platform linux/arm64 \
  -t ${AWS_ACCOUNT_ID}.dkr.ecr.ap-south-1.amazonaws.com/practice-repo:v2 \
  --push \
  .

# Verify both v1 and v2 images exist
aws ecr list-images --repository-name practice-repo --region ap-south-1

# Expected output:
# {
#     "imageIds": [
#         {"imageTag": "v1"},
#         {"imageTag": "v2"}
#     ]
# }
```

#### Step 3: Update Helm Values

```bash
# Navigate to helm-chart directory
cd ../helm-chart

# Edit values.yaml
nano values.yaml

# Find:
# image:
#   repository: xxx.dkr.ecr.ap-south-1.amazonaws.com/practice-repo
#   tag: v1

# Change to:
# image:
#   repository: xxx.dkr.ecr.ap-south-1.amazonaws.com/practice-repo
#   tag: v2

# Save: Ctrl+O, Enter, Ctrl+X
```

#### Step 4: Commit and Push to Git

```bash
# Navigate back to repository root
cd ..

# Check what changed
git status

# Expected output:
# modified:   app-source/index.html
# modified:   helm-chart/values.yaml

# View differences
git diff

# Stage changes
git add .

# Commit with descriptive message
git commit -m "Update to v2: Changed version text and image tag"

# Push to GitLab
git push origin main
```

#### Step 5: Watch ArgoCD Sync

```bash
# Option 1: ArgoCD UI
# 1. Open https://localhost:8080
# 2. Click on "frontend-app"
# 3. You'll see "OutOfSync" status (orange)
# 4. Wait 3 minutes for auto-sync OR click "SYNC" button
# 5. Watch the sync process in real-time
# 6. Status changes to "Synced" (green)

# Option 2: kubectl (watch deployment)
kubectl get pods -w

# You'll see:
# frontend-app-xxxxxxxxxx-xxxxx   1/1     Running       0          10m  (v1)
# frontend-app-xxxxxxxxxx-yyyyy   1/1     Running       0          10m  (v1)
# frontend-app-zzzzzzzzzz-aaaaa   0/1     Pending       0          0s   (v2)
# frontend-app-zzzzzzzzzz-aaaaa   0/1     ContainerCreating   0          1s
# frontend-app-zzzzzzzzzz-aaaaa   1/1     Running             0          30s
# frontend-app-xxxxxxxxxx-xxxxx   1/1     Terminating         0          11m

# Press Ctrl+C to stop watching

# Option 3: ArgoCD CLI
argocd app get frontend-app

# Expected output:
# Name:               frontend-app
# Project:            default
# Server:             https://kubernetes.default.svc
# Namespace:          default
# URL:                https://localhost:8080/applications/frontend-app
# Repo:               https://gitlab.com/YOUR_USERNAME/gitops-repo.git
# Target:             main
# Path:               helm-chart
# SyncWindow:         Sync Allowed
# Sync Policy:        Automated (Prune)
# Sync Status:        Synced to main (abc123)
# Health Status:      Healthy

# Manual sync (if auto-sync disabled)
argocd app sync frontend-app
```

#### Step 6: Verify Update

```bash
# Get LoadBalancer URL
kubectl get svc frontend-app-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# Open in browser or curl:
curl http://$(kubectl get svc frontend-app-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# You should now see: "Version 2 - Updated via GitOps!"

# Check running pods image
kubectl describe pod $(kubectl get pods -l app=frontend-app -o jsonpath='{.items[0].metadata.name}') | grep Image:

# Expected output:
# Image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/practice-repo:v2
```

#### Step 7: Rollback (Optional Practice)

```bash
# GitOps makes rollback easy - just revert the Git commit!

# View commit history
git log --oneline

# Example output:
# def456 Update to v2: Changed version text and image tag
# abc123 Initial commit: v1 GitOps setup

# Revert to v1
git revert HEAD
# This creates a new commit that undoes the v2 changes

# Or use git reset (rewrites history):
# git reset --hard abc123  # Use commit hash from git log
# git push --force origin main

# Push the revert
git push origin main

# ArgoCD will detect the change and roll back to v1
# Watch in ArgoCD UI or kubectl get pods -w
```

### Understanding ArgoCD Sync Policies

**Manual Sync:**
```yaml
syncPolicy: {}  # No automated sync - requires manual "SYNC" button click
```

**Automated Sync:**
```yaml
syncPolicy:
  automated:
    prune: false  # Don't delete resources removed from Git
    selfHeal: false  # Don't revert manual kubectl changes
```

**Automated Sync with Prune & Self-Heal (Recommended):**
```yaml
syncPolicy:
  automated:
    prune: true  # Delete resources if removed from Git
    selfHeal: true  # Revert any manual changes (Git is truth)
```

**Our Configuration:**
We use automated sync with prune and self-heal. This means:
- ✅ Git is the single source of truth
- ✅ Changes in Git automatically deploy (within 3 minutes)
- ✅ Manual `kubectl` changes get reverted
- ✅ Deleted resources in Git get deleted in cluster

---

## 🧹 Cleanup: Delete All Resources

### Why Cleanup Matters
- **EKS Control Plane**: ~$72/month
- **EC2 Instances**: ~$24/month
- **LoadBalancers**: ~$16/month
- **Total**: ~$112/month if left running!

### Cleanup Order (IMPORTANT)

```bash
# Step 1: Delete ArgoCD Application (removes LoadBalancer)
kubectl delete application frontend-app -n argocd

# Wait for LoadBalancer to be deleted (2-3 minutes)
kubectl get svc -A -w
# Press Ctrl+C when frontend-app-service is gone

# Step 2: Delete ArgoCD
kubectl delete namespace argocd

# Step 3: Delete EKS Cluster (TAKES 10-15 MINUTES)
eksctl delete cluster --name demo-cluster --region ap-south-1

# This deletes:
# - All worker nodes
# - Node groups
# - VPC and subnets
# - Internet Gateway, NAT Gateway
# - Security groups
# - EKS control plane

# Step 4: Delete ECR Repository (and all images)
aws ecr delete-repository \
  --repository-name practice-repo \
  --region ap-south-1 \
  --force

# Step 5: (Optional) Delete IAM User
# In AWS Console:
# IAM → Users → gitops-admin → Delete user

# Step 6: Verify deletion
aws eks list-clusters --region ap-south-1
# Should return: "clusters": []

aws ecr describe-repositories --region ap-south-1
# Should return: "repositories": []
```

---

## ❌ Troubleshooting Guide

### Issue 1: "connection refused localhost:8080"

**Symptoms:**
- Cannot access ArgoCD UI
- Browser shows "ERR_CONNECTION_REFUSED"

**Diagnosis:**
```bash
# Check if port-forward is running
ps aux | grep "port-forward"

# Check if argocd-server pod is running
kubectl get pods -n argocd | grep argocd-server
```

**Solutions:**
```bash
# Solution 1: Restart port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Solution 2: Update kubeconfig
aws eks update-kubeconfig --region ap-south-1 --name demo-cluster

# Solution 3: Check if pods are ready
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
```

---

### Issue 2: "You must be logged in to the server (Unauthorized)"

**Symptoms:**
- `kubectl get nodes` returns error
- Any kubectl command fails with "Unauthorized"

**Diagnosis:**
```bash
# Check current context
kubectl config current-context

# Check if user is in kubeconfig
kubectl config view

# List access entries
aws eks list-access-entries --cluster-name demo-cluster --region ap-south-1
```

**Solutions:**
```bash
# Solution 1: Add access entry via Console
# AWS Console → EKS → demo-cluster → Access tab → Create access entry

# Solution 2: Add access entry via CLI
aws eks create-access-entry \
  --cluster-name demo-cluster \
  --principal-arn arn:aws:iam::${AWS_ACCOUNT_ID}:user/gitops-admin \
  --type STANDARD \
  --region ap-south-1

aws eks associate-access-policy \
  --cluster-name demo-cluster \
  --principal-arn arn:aws:iam::${AWS_ACCOUNT_ID}:user/gitops-admin \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster \
  --region ap-south-1

# Solution 3: Update kubeconfig
aws eks update-kubeconfig --region ap-south-1 --name demo-cluster
```

---

### Issue 3: "exec format error" in pod logs

**Symptoms:**
- Pod status: CrashLoopBackOff
- Pod logs show: `exec format error`

**Diagnosis:**
```bash
# Check pod logs
kubectl logs <pod-name>

# Check node architecture
kubectl get nodes -o wide
# Look for ARCHITECTURE column

# Check image architecture
docker buildx imagetools inspect ${AWS_ACCOUNT_ID}.dkr.ecr.ap-south-1.amazonaws.com/practice-repo:v1
```

**Root Cause:**
- Docker image built for x86_64 (Intel/AMD)
- EKS nodes are ARM64 (t4g.small)
- Binary formats incompatible

**Solutions:**
```bash
# Solution: Rebuild image for ARM64
cd app-source

docker buildx build \
  --platform linux/arm64 \
  -t ${AWS_ACCOUNT_ID}.dkr.ecr.ap-south-1.amazonaws.com/practice-repo:v1 \
  --push \
  .

# Restart pods
kubectl rollout restart deployment frontend-app
```

---

### Issue 4: Nodes not joining cluster

**Symptoms:**
- `kubectl get nodes` shows 0 nodes
- Cluster created but no worker capacity

**Diagnosis:**
```bash
# Check node groups
eksctl get nodegroup --cluster demo-cluster --region ap-south-1

# Check CloudFormation stacks
aws cloudformation list-stacks --region ap-south-1 | grep demo-cluster

# Check EC2 instances
aws ec2 describe-instances \
  --filters "Name=tag:eks:cluster-name,Values=demo-cluster" \
  --region ap-south-1
```

**Solutions:**
```bash
# Solution 1: Verify IAM roles
# Ensure NodeInstanceRole has:
# - AmazonEKSWorkerNodePolicy
# - AmazonEC2ContainerRegistryReadOnly
# - AmazonEKS_CNI_Policy

# Solution 2: Delete and recreate cluster
eksctl delete cluster --name demo-cluster --region ap-south-1
eksctl create cluster \
  --name demo-cluster \
  --region ap-south-1 \
  --node-type t4g.small \
  --nodes 2 \
  --managed
```

---

### Issue 5: ArgoCD shows "OutOfSync" but won't sync

**Symptoms:**
- Application stuck in OutOfSync status
- Manual sync fails
- Sync status shows errors

**Diagnosis:**
```bash
# Check application details
argocd app get frontend-app

# Check ArgoCD logs
kubectl logs -n argocd deployment/argocd-application-controller

# Check repository connection
argocd repo list
```

**Solutions:**
```bash
# Solution 1: Verify Git credentials
# ArgoCD UI → Settings → Repositories
# Test connection

# Solution 2: Hard refresh
argocd app get frontend-app --hard-refresh

# Solution 3: Delete and recreate application
kubectl delete application frontend-app -n argocd
kubectl apply -f argocd/application.yaml

# Solution 4: Check Helm syntax
cd helm-chart
helm lint .
helm template . --debug
```

---

### Issue 6: LoadBalancer stuck in "Pending"

**Symptoms:**
- `kubectl get svc` shows EXTERNAL-IP as `<pending>`
- Cannot access application

**Diagnosis:**
```bash
# Check service events
kubectl describe svc frontend-app-service

# Check AWS Load Balancer Controller
kubectl get pods -n kube-system | grep aws-load-balancer-controller
```

**Solutions:**
```bash
# Solution 1: Wait (can take 5 minutes)
kubectl get svc frontend-app-service -w

# Solution 2: Check service quotas
# AWS Console → Service Quotas → Elastic Load Balancing
# Ensure you haven't hit limits

# Solution 3: Verify VPC has public subnets
aws eks describe-cluster --name demo-cluster --region ap-south-1 | grep -A 10 resourcesVpcConfig

# Solution 4: Recreate service
kubectl delete svc frontend-app-service
kubectl apply -f helm-chart/templates/service.yaml
```

---

### Issue 7: Image pull errors

**Symptoms:**
- Pod stuck in ImagePullBackOff
- Pod logs: `Failed to pull image`

**Diagnosis:**
```bash
# Check pod events
kubectl describe pod <pod-name>

# Look for:
# - Failed to pull image
# - no basic auth credentials
# - 403 Forbidden
```

**Solutions:**
```bash
# Solution 1: Verify image exists in ECR
aws ecr list-images --repository-name practice-repo --region ap-south-1

# Solution 2: Check IAM permissions
# Ensure node IAM role has AmazonEC2ContainerRegistryReadOnly

# Solution 3: Verify image URI in values.yaml
cat helm-chart/values.yaml | grep repository

# Should match: <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/practice-repo

# Solution 4: Manual pull test
kubectl run test --image=${AWS_ACCOUNT_ID}.dkr.ecr.ap-south-1.amazonaws.com/practice-repo:v1 --rm -it -- sh
```

---

### Issue 8: Out of memory errors

**Symptoms:**
- Pods getting OOMKilled
- Node pressure warnings

**Diagnosis:**
```bash
# Check node resources
kubectl top nodes

# Check pod resources
kubectl top pods

# Check pod resource limits
kubectl describe pod <pod-name> | grep -A 10 "Limits:"
```

**Solutions:**
```bash
# Solution 1: Increase memory limits in values.yaml
# Edit helm-chart/values.yaml:
resources:
  limits:
    memory: 256Mi  # Increase from 128Mi

# Solution 2: Scale down replicas
# Edit helm-chart/values.yaml:
replicaCount: 1  # Reduce from 2

# Solution 3: Use larger nodes (costs more!)
eksctl scale nodegroup \
  --cluster demo-cluster \
  --region ap-south-1 \
  --name <nodegroup-name> \
  --nodes 0

# Then create new nodegroup with t4g.medium
```

---

## 📚 Learning Resources

### Kubernetes Concepts

- **Pods**: Smallest deployable unit (one or more containers)
- **Deployments**: Manages pod replicas, rolling updates
- **Services**: Exposes pods to network (ClusterIP, NodePort, LoadBalancer)
- **Namespaces**: Virtual clusters for resource isolation
- **ConfigMaps**: Store configuration data
- **Secrets**: Store sensitive data (encrypted)

### GitOps Principles

1. **Declarative**: Entire system state described declaratively
2. **Versioned**: System state versioned in Git
3. **Automated**: Changes automatically applied
4. **Reconciled**: Agents ensure desired state matches actual state

### Further Reading

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Helm Documentation](https://helm.sh/docs/)
- [AWS EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)
- [GitOps Principles](https://opengitops.dev/)

---

## ✅ Success Criteria

You have successfully completed this lab if you can:

1. ✅ Create AWS resources programmatically (IAM, ECR, EKS)
2. ✅ Build multi-architecture Docker images (ARM64)
3. ✅ Deploy applications using Helm charts
4. ✅ Implement GitOps workflow with ArgoCD
5. ✅ Perform zero-downtime updates via Git commits
6. ✅ Troubleshoot common Kubernetes issues
7. ✅ Clean up resources to avoid AWS charges

---

## 🎓 Next Steps

After mastering this lab, explore:

- **Helm Hooks**: Pre/post install scripts
- **Kustomize**: Alternative to Helm
- **Progressive Delivery**: Canary deployments, blue-green
- **Monitoring**: Prometheus, Grafana
- **Secrets Management**: External Secrets Operator
- **Multi-cluster**: Deploy to dev/staging/prod
- **CI/CD Integration**: GitHub Actions, GitLab CI

---

**Congratulations!** You've built a production-grade GitOps pipeline. 🎉
