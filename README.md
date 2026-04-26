# EKS Terraform Project - Complete Analysis & Production-Grade Enhancements

## 📋 Executive Summary

This is a **production-ready Amazon EKS (Elastic Kubernetes Service) cluster** Infrastructure-as-Code project using Terraform. It automates the deployment of a complete Kubernetes infrastructure with networking, security, identity management, and secrets management.

---

## 🎯 What This Project Creates

### 1. **Network Infrastructure (VPC Module)**
- **1 Virtual Private Cloud (VPC)** with CIDR `10.0.0.0/16`
- **3 Public Subnets** (for load balancers, NAT gateways, internet access)
- **3 Private Subnets** (for EKS worker nodes - not directly accessible from internet)
- **1 Internet Gateway** (for public subnet internet connectivity)
- **1 NAT Gateway** (allows private subnets to reach the internet securely)
- Proper Kubernetes-required tags for service discovery

### 2. **Kubernetes Cluster Control Plane (EKS Module)**
- **AWS-managed EKS Cluster** (Kubernetes v1.31)
- **KMS Encryption** for secrets at rest
- **CloudWatch Logging** for audit trails (API, authenticator, scheduler logs)
- **Multiple availability zones** for high availability
- **Public & Private API endpoints** for secure access
- **IRSA (IAM Roles for Service Accounts)** for fine-grained pod permissions

### 3. **Worker Nodes (2 Node Groups)**

#### **General Node Group (Production-grade)**
- **Instance Type**: m7i-flex.large (2 vCPU, 8 GB RAM)
- **Capacity**: 2 nodes (desired) → 2-4 nodes (autoscaling range)
- **Type**: ON_DEMAND (always available, reliable)
- **Purpose**: Running critical workloads
- **EBS Volume**: 20 GB gp3

#### **Spot Node Group (Cost-optimized)**
- **Instance Type**: m7i-flex.large (with fallback support)
- **Capacity**: 1 node (desired) → 1-3 nodes (autoscaling range)
- **Type**: SPOT (up to 70% cheaper but can be interrupted)
- **Taint**: `spot=true:NoSchedule` (pods must opt-in to run here)
- **Purpose**: Non-critical workloads, batch jobs, development

### 4. **Security & IAM (IAM Module)**
- **Cluster Role**: Permissions for EKS control plane
  - EC2 instance management
  - CloudWatch logs
  - VPC security management
  - KMS encryption
- **Node Role**: Permissions for worker nodes
  - ECR image pulling
  - CloudWatch logs
  - EBS/EFS access
  - S3 access
  - CloudWatch metrics

### 5. **Secrets Management (Optional)**
- **AWS Secrets Manager** integration
- Store:
  - Database credentials
  - API keys
  - Application configurations
- Optionally disabled (enabled via variables)

### 6. **Add-ons (Essential Kubernetes Components)**
- **CoreDNS**: DNS resolution within cluster
- **VPC CNI**: Pod networking and IP assignment
- **kube-proxy**: Service routing
- **EBS CSI**: Persistent storage management

---

## 🔍 Production-Grade Assessment

### ✅ **Already Production-Ready:**
1. **Multi-AZ Deployment** - Resources spread across 3 availability zones
2. **Encryption** - KMS encryption for secrets at rest
3. **IAM Security** - Role-based access with least privilege
4. **Network Isolation** - Private subnets for worker nodes
5. **Logging** - CloudWatch integration for audit trails
6. **High Availability** - Managed EKS control plane (AWS responsibility)
7. **Auto-scaling** - Node groups can scale based on demand
8. **Cost Optimization** - Mix of on-demand and spot instances
9. **IRSA** - Pod-level IAM permissions (best practice)
10. **Tag Management** - Comprehensive tagging strategy

### ⚠️ **Missing for Full Production:**
1. **No Remote State Backend** - Uses local `terraform.tfstate` (CRITICAL ISSUE)
2. **No State Locking** - Team deployments can corrupt state
3. **No Backup Strategy** - Terraform state not backed up
4. **No Monitoring** - EKS logs enabled but no alerting configured
5. **No Ingress Controller** - Kubernetes load balancer service not pre-configured
6. **No Service Mesh** - No traffic management or observability
7. **No RBAC Configuration** - Basic cluster but no pod security policies
8. **No GitOps Integration** - No ArgoCD or Flux for continuous deployment
9. **No SIEM Integration** - CloudWatch logs created but not centralized

---

## 📊 Architecture Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                      AWS Account                             │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │            VPC (10.0.0.0/16)                         │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │                                                      │   │
│  │  ┌─ PUBLIC SUBNETS ─────────┐                       │   │
│  │  │ 10.0.101-103.0/24        │                       │   │
│  │  │ Internet Gateway (IGW)    │                       │   │
│  │  │ NAT Gateway (AZ1)         │                       │   │
│  │  │ Load Balancer IPs         │                       │   │
│  │  └─────────────────────────────┐                     │   │
│  │                                │                     │   │
│  │  ┌─ PRIVATE SUBNETS ────┐      │                     │   │
│  │  │ 10.0.1-3.0/24        │      │                     │   │
│  │  │                      │      │                     │   │
│  │  │  ┌────────────────┐  │      │                     │   │
│  │  │  │  General Nodes │  │      │ Routes through      │   │
│  │  │  │  2x m7i-flex.lg│◄┼──────┤ NAT Gateway         │   │
│  │  │  │  ON_DEMAND     │  │      │                     │   │
│  │  │  └────────────────┘  │      │                     │   │
│  │  │                      │      │                     │   │
│  │  │  ┌────────────────┐  │      │                     │   │
│  │  │  │  Spot Nodes    │  │      │                     │   │
│  │  │  │  1x m7i-flex.lg│◄┼──────┘                     │   │
│  │  │  │  SPOT (cheap)  │  │                           │   │
│  │  │  └────────────────┘  │                           │   │
│  │  │                      │                           │   │
│  │  │ EKS Control Plane    │                           │   │
│  │  │ (AWS Managed)        │                           │   │
│  │  │ - API Server         │                           │   │
│  │  │ - Scheduler          │                           │   │
│  │  │ - etcd (KMS encrypted)                          │   │
│  │  │ - Controller Manager │                           │   │
│  │  └──────────────────────┘                           │   │
│  │                                                      │   │
│  │ Add-ons: CoreDNS | VPC CNI | kube-proxy | EBS CSI  │   │
│  │                                                      │   │
│  │ Security Groups:                                     │   │
│  │ - Cluster SG (API access)                           │   │
│  │ - Node SG (inter-node & pod traffic)               │   │
│  │                                                      │   │
│  │ IAM Roles:                                           │   │
│  │ - Cluster Role (EKS control plane)                 │   │
│  │ - Node Role (worker nodes)                         │   │
│  │ - IRSA OIDC Provider (pod-level IAM)               │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  Additional Resources:                                       │
│  - CloudWatch Log Group: /aws/eks/cluster/logs             │
│  - KMS Key: EKS encryption key                             │
│  - Secrets Manager: (optional) DB creds, API keys         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 💰 Estimated Monthly Costs (US-EAST-1)

| Component | Quantity | Unit Cost | Monthly Cost |
|-----------|----------|-----------|--------------|
| EKS Control Plane | 1 cluster | $0.10/hour | $73 |
| m7i-flex.large (ON_DEMAND) | 2 nodes | $0.063/hour each | $92 |
| m7i-flex.large (SPOT) | 1 node | $0.019/hour | $14 |
| NAT Gateway | 1 gateway | $0.045/hour | $33 |
| NAT Gateway Data Transfer | ~100GB | $0.045/GB | $4.50 |
| CloudWatch Logs | ~2-5GB/month | $0.50/GB | $2.50 |
| KMS Key Usage | 1 key | $1.00/month | $1 |
| **TOTAL** | | | **~$220/month** |

⚠️ **Don't forget to destroy the cluster when not in use!**

---

## 🏗️ Project Structure

```
eks-main/
├── README.md                          # Project documentation
├── task.md                            # Day 20 tasks
└── code/
    ├── main.tf                        # Orchestrator (imports all modules)
    ├── variables.tf                   # Input variables & customization
    ├── outputs.tf                     # Cluster endpoints & outputs
    ├── provider.tf                    # AWS provider config
    ├── backend.tf                     # 🆕 Remote backend configuration
    ├── terraform.tfvars              # 🆕 Example variables file
    ├── .terraform.lock.hcl           # Provider versions (lock file)
    ├── DEMO_GUIDE.md                 # Deployment walkthrough
    ├── CUSTOM_MODULES.md             # Module documentation
    └── modules/
        ├── vpc/                       # VPC, subnets, NAT, IGW
        │   ├── main.tf
        │   ├── variables.tf
        │   └── outputs.tf
        ├── iam/                       # IAM roles & policies
        │   ├── main.tf
        │   ├── variables.tf
        │   └── outputs.tf
        ├── eks/                       # EKS cluster & node groups
        │   ├── main.tf
        │   ├── variables.tf
        │   ├── outputs.tf
        │   └── templates/
        │       └── userdata.sh
        └── secrets-manager/           # Secrets Manager (optional)
            ├── main.tf
            ├── variables.tf
            └── outputs.tf
```

---

## 🚨 CRITICAL: Local State vs. Remote Backend Problem

### The Problem with Local State

```
# Current setup (INSECURE for production):
❌ terraform.tfstate file stored locally on your machine
❌ State file contains sensitive data (encrypted passwords, keys)
❌ Team members can't collaborate (merge conflicts)
❌ No automatic backup
❌ No state locking (multiple people could apply simultaneously)
❌ If you lose your machine, you lose the cluster metadata
```

### Why This Matters

The `terraform.tfstate` file is like a blueprint of your entire infrastructure:
- **Contains secrets** (database passwords, API keys, credentials)
- **Required to make changes** (without it, Terraform can't know what exists)
- **Shared among team members** (production changes need collaboration)
- **Must be available** (losing it means losing the ability to manage resources)

---

## ✅ SOLUTION: Production-Grade Remote Backend Configuration

### Option 1: **S3 + DynamoDB Backend (Recommended for AWS)**

This is the most common production setup for AWS-based teams.

#### Step 1: Create S3 Bucket & DynamoDB Table (Run Once)

```bash
# Create a separate directory for backend setup
mkdir -p terraform-backend
cd terraform-backend

# Create backend.tf
cat > backend-setup.tf << 'EOF'
# This file creates the infrastructure for storing Terraform state
# Run this ONCE before your main infrastructure

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"  # Same region as your cluster
}

# S3 bucket for Terraform state
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-company-terraform-state-${data.aws_caller_identity.current.account_id}"

  tags = {
    Name        = "Terraform State Bucket"
    Environment = "production"
    Purpose     = "Terraform state storage"
  }
}

# Enable versioning for state recovery
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Block public access
resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Enable encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# DynamoDB table for state locking
resource "aws_dynamodb_table" "terraform_locks" {
  name           = "terraform-locks"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name        = "Terraform Lock Table"
    Environment = "production"
    Purpose     = "State locking"
  }
}

# Data source to get current AWS account ID
data "aws_caller_identity" "current" {}

# Outputs
output "s3_bucket_name" {
  value       = aws_s3_bucket.terraform_state.id
  description = "Name of the S3 bucket for Terraform state"
}

output "dynamodb_table_name" {
  value       = aws_dynamodb_table.terraform_locks.name
  description = "Name of the DynamoDB table for state locking"
}
EOF

# Deploy the backend infrastructure
terraform init
terraform plan
terraform apply

# Save the outputs (you'll need them next)
terraform output -json > backend-outputs.json
```

#### Step 2: Update Your EKS Project with Remote Backend

Create `code/backend.tf`:

```hcl
# Remote Backend Configuration
# Stores Terraform state in S3 with DynamoDB locking
# This ensures state is backed up, encrypted, and prevents conflicts

terraform {
  backend "s3" {
    # Replace these with your actual values
    bucket         = "my-company-terraform-state-123456789"  # From backend setup output
    key            = "eks/cluster/terraform.tfstate"         # Unique path for this infrastructure
    region         = "us-east-1"                             # Same as cluster region
    encrypt        = true                                    # Server-side encryption
    dynamodb_table = "terraform-locks"                       # Prevents concurrent changes
  }
}
```

#### Step 3: Migrate Existing State (if applicable)

```bash
cd code

# Copy local state to backend
terraform init

# When prompted about moving state, type 'yes'
# Terraform will automatically migrate your local state to S3
```

#### Step 4: Verify Backend is Working

```bash
# Check the S3 bucket
aws s3 ls s3://my-company-terraform-state-123456789/

# You should see: eks/cluster/terraform.tfstate
# Plus: eks/cluster/terraform.tfstate.backup (versioning backup)

# Check DynamoDB
aws dynamodb scan --table-name terraform-locks --region us-east-1
```

---

### Option 2: **Terraform Cloud Backend (Easiest for Teams)**

If you want the simplest setup with built-in best practices:

#### Step 1: Create Terraform Cloud Account

```bash
# Go to https://app.terraform.io
# Sign up for free account
# Create an organization (e.g., "my-company")
```

#### Step 2: Create API Token

```bash
# In Terraform Cloud:
# 1. Go to Account Settings → Tokens
# 2. Create API Token
# 3. Copy the token

# Add to your machine
cat ~/.terraformrc
# Should contain:
# credentials "app.terraform.io" {
#   token = "YOUR_TOKEN_HERE"
# }
```

#### Step 3: Update Backend Configuration

Create `code/backend.tf`:

```hcl
terraform {
  cloud {
    organization = "my-company"  # Your organization name
    
    workspaces {
      name = "eks-production"    # Workspace name
    }
  }
}
```

#### Step 4: Initialize and Migrate

```bash
cd code
terraform init

# When prompted about migrating state, type 'yes'
```

**Advantages:**
- ✅ Free tier includes remote state
- ✅ Automatic backups
- ✅ State locking included
- ✅ UI for viewing state
- ✅ Runs terraform commands on Terraform Cloud servers
- ✅ Team management built-in

---

### Option 3: **Azure Backend (for hybrid teams)**

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state"
    storage_account_name = "tfstate123456"
    container_name       = "tfstate"
    key                  = "eks/cluster/terraform.tfstate"
  }
}
```

---

## 📝 Enhanced Configuration Files

### New File: `code/terraform.tfvars.example`

```hcl
# This file shows what variables you can customize
# Copy to terraform.tfvars and update with your values

# Cluster Configuration
aws_region       = "us-east-1"
cluster_name     = "production-eks"
kubernetes_version = "1.31"
environment      = "production"

# Network Configuration
vpc_cidr        = "10.0.0.0/16"
private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

# Secrets Manager Configuration (Optional)
enable_db_secret         = false
enable_api_secret        = false
enable_app_config_secret = false

# Database Credentials (only if enable_db_secret = true)
db_username = "admin"
db_password = "ChangeMe123!"  # Never commit real passwords!
db_engine   = "postgres"
db_host     = "my-db.123456.us-east-1.rds.amazonaws.com"
db_port     = 5432
db_name     = "myapp"

# API Keys (only if enable_api_secret = true)
api_key    = "sk-api-key-123"
api_secret = "sk-secret-789"
```

### New File: `code/.gitignore`

```
# Local Terraform files
.terraform/
.terraform.lock.hcl
terraform.tfvars
terraform.tfvars.json
*.tfvars
*.tfvars.json
crash.log
crash.*.log
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Local .terraform directories
**/.terraform/*

# .tfstate files
*.tfstate
*.tfstate.*
*.backup
*.tfbackup

# Ignore override files, as they are usually used to override resources locally
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Ignore plan files
*.tfplan

# Ignore lock files for local development
# .terraform.lock.hcl

# IDE
.idea/
*.swp
*.swo
*~
.vscode/
*.code-workspace

# OS
.DS_Store
Thumbs.db

# Environment files
.env
.env.local
.env.*.local
```

### Enhanced: `code/backend.tf`

```hcl
# ============================================================================
# TERRAFORM REMOTE BACKEND CONFIGURATION
# ============================================================================
# 
# This configuration stores your Terraform state in AWS S3 with DynamoDB locking.
# 
# Benefits:
# ✅ Secure - State encrypted in S3, no sensitive data on local machine
# ✅ Collaborative - Multiple team members can work on same infrastructure
# ✅ Backed up - S3 versioning keeps history of all state changes
# ✅ Locked - DynamoDB prevents concurrent modifications
# ✅ Reliable - AWS-managed, highly available storage
#
# ============================================================================

terraform {
  backend "s3" {
    # S3 bucket name (created by backend-setup.tf)
    # Update this with your actual bucket name
    bucket = "my-company-terraform-state-123456789"
    
    # Path within the bucket for this specific infrastructure
    # Allows storing multiple projects in same bucket
    key = "eks/production/terraform.tfstate"
    
    # AWS region (same as your cluster)
    region = "us-east-1"
    
    # Enable server-side encryption for state file
    # Protects sensitive data at rest
    encrypt = true
    
    # DynamoDB table for state locking
    # Prevents corruption from concurrent terraform apply
    dynamodb_table = "terraform-locks"
    
    # ACL for the S3 object (private is most secure)
    # acl = "private"
  }
}

# ============================================================================
# LOCAL STATE BACKEND (For development only - DO NOT USE IN PRODUCTION)
# ============================================================================
# 
# Uncomment the section below only for local development
# This stores state on your local machine
#
# ⚠️ WARNING: Do NOT use this in production!
# Risks:
# ❌ State file lost if machine fails
# ❌ Secrets stored unencrypted locally
# ❌ Difficult for team collaboration
# ❌ No state locking (merge conflicts possible)
#
# terraform {
#   backend "local" {
#     path = "terraform.tfstate"
#   }
# }
```

---

## 🔐 Security Hardening Checklist

After deploying with remote backend, add these:

### 1. **Enable MFA Delete on S3**
```bash
aws s3api put-bucket-versioning \
  --bucket my-company-terraform-state-123456789 \
  --region us-east-1 \
  --versioning-configuration Status=Enabled,MFADelete=Enabled \
  --mfa "SERIAL-NUMBER MFA-CODE"
```

### 2. **Enable S3 Block Public Access (Already done in setup)**
```bash
aws s3api put-public-access-block \
  --bucket my-company-terraform-state-123456789 \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

### 3. **Add Bucket Policy for Access Control**
```hcl
resource "aws_s3_bucket_policy" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Deny"
        Principal = "*"
        Action = "s3:*"
        Resource = [
          aws_s3_bucket.terraform_state.arn,
          "${aws_s3_bucket.terraform_state.arn}/*"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      }
    ]
  })
}
```

### 4. **Enable CloudTrail for Audit**
```bash
# Log all S3 access for compliance
aws cloudtrail create-trail \
  --name terraform-state-audit \
  --s3-bucket-name terraform-audit-logs \
  --is-multi-region-trail
```

---

## 🚀 Deployment Workflow

### For Individual Developer:
```bash
cd code
terraform init              # Downloads providers & configures backend
terraform plan              # Review what will be created
terraform apply             # Deploy infrastructure
```

### For Team:
```bash
# Team member A:
cd code
terraform init              # Connects to remote backend
terraform plan -out=plan1
# Sends plan for review...

# After code review approval:
terraform apply plan1       # Only applies what was reviewed

# Team member B (same infrastructure):
terraform init              # Pulls latest state from S3
# They see current state, not local copy
```

---

## 📋 Complete File List (Production Setup)

```
eks-main/code/
├── main.tf                          # Imports all modules
├── variables.tf                     # Input variables
├── outputs.tf                       # Outputs
├── provider.tf                      # AWS provider
├── backend.tf                       # 🆕 Remote backend
├── terraform.tfvars.example         # 🆕 Example variables
├── .gitignore                       # 🆕 Prevent committing secrets
├── .terraform.lock.hcl              # ✅ Commit this (provider versions)
├── modules/
│   ├── vpc/...
│   ├── iam/...
│   ├── eks/...
│   └── secrets-manager/...
├── terraform-backend/               # 🆕 For backend infrastructure
│   ├── backend-setup.tf
│   ├── terraform.tfstate
│   └── .terraform/
└── README.md
```

---

## ✅ Production Readiness Checklist

- [x] Multi-AZ deployment
- [x] KMS encryption for secrets
- [x] CloudWatch logging enabled
- [x] IAM roles configured
- [x] Security groups configured
- [x] IRSA enabled
- [x] Auto-scaling configured
- [x] Spot instances for cost optimization
- [ ] **Remote state backend configured** ← ADD THIS
- [ ] **State locking enabled** ← ADD THIS
- [ ] **State backup configured** ← ADD THIS
- [ ] **Team access controls** ← ADD THIS
- [ ] Monitoring/alerting configured
- [ ] Backup strategy defined
- [ ] Disaster recovery plan
- [ ] Documentation complete
- [ ] Security audit completed

---

## 🎯 Quick Implementation Guide

### Step 1: Set Up Backend Infrastructure
```bash
# In terraform-backend/ directory
terraform init
terraform apply
# Note the bucket name and table name from output
```

### Step 2: Update Main Configuration
```bash
# In code/ directory
# 1. Copy backend.tf (provided above)
# 2. Update bucket name and region
# 3. Add terraform.tfvars.example
# 4. Add .gitignore
```

### Step 3: Migrate State
```bash
cd code
terraform init
# Type 'yes' when asked about migrating existing state
```

### Step 4: Verify
```bash
aws s3 ls s3://my-company-terraform-state-123456789/
# You should see your tfstate file there now
```

---

## 🔍 Troubleshooting

### "Error: Backend initialization required"
```bash
# You haven't initialized with backend yet
terraform init
```

### "Error: Error acquiring the lock"
```bash
# Someone else is currently running terraform
# Wait for them to finish, or:
aws dynamodb scan --table-name terraform-locks
# Manually check DynamoDB to see who has the lock
```

### "Error: Insufficient permissions to access S3"
```bash
# Your AWS credentials don't have S3 permissions
# Add AmazonS3FullAccess and DynamoDBFullAccess policies
```

---

## 📚 Summary

This EKS project is **production-ready** for the infrastructure itself, but to make it **truly production-grade for teams**, add:

1. **✅ Remote S3 Backend** - Keeps state safe and shared
2. **✅ DynamoDB Locking** - Prevents concurrent changes
3. **✅ State Versioning** - Recovery capability
4. **✅ Access Controls** - IAM policies for team members
5. **✅ Audit Logging** - CloudTrail for compliance

The configurations provided above handle all of these!

---

## 📞 Support Resources

- [Terraform AWS Provider Docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/)
- [Terraform Remote State Docs](https://www.terraform.io/language/state/remote)
- [Kubernetes Docs](https://kubernetes.io/docs/)
