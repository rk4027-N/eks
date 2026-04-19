# EKS Cluster Terraform Project - Clear Explanation

## 📋 Overview

This is a **Terraform infrastructure-as-code project** for deploying a **production-ready Amazon EKS (Elastic Kubernetes Service) cluster** on AWS. It's part of Day 20 of a 30-day AWS learning challenge.

---

## 🎯 What Does This Project Do?

Imagine you need to set up a Kubernetes cluster on AWS. Normally, you'd click through the AWS console for hours. This project **automates everything** using Terraform, so you can deploy a complete EKS cluster with one command.

### What Gets Created:

1. **VPC (Virtual Private Cloud)** - A private network on AWS
   - 3 public subnets (for load balancers, gateways)
   - 3 private subnets (for Kubernetes nodes)
   - Internet Gateway + NAT Gateway for connectivity

2. **EKS Cluster** - The Kubernetes control plane
   - Manages your containerized applications
   - Kubernetes version 1.31
   - Encrypted secrets using KMS

3. **Two Types of Worker Nodes**:
   - **General Node Group** (2 nodes): Stable, always-on instances (c7i-flex.large)
   - **Spot Node Group** (1 node): Cheaper but interruptible instances (t3.micro/t3.small)

4. **IAM Roles & Permissions** - Security configuration
   - Cluster role (for EKS control plane)
   - Node role (for worker nodes)
   - IRSA enabled (for pod-level permissions)

5. **Secrets Manager** (optional) - Secure credential storage
   - Database credentials
   - API keys
   - Application configs

---

## 📁 Project Structure

```
eks-main/
├── README.md              # Main project info
├── task.md                # Day 20 tasks & requirements
└── code/                  # Terraform code
    ├── main.tf            # Main configuration (calls all modules)
    ├── variables.tf       # Input variables (customizable)
    ├── outputs.tf         # Output values (what you get after deployment)
    ├── provider.tf        # AWS provider config
    ├── DEMO_GUIDE.md      # Step-by-step deployment guide
    ├── CUSTOM_MODULES.md  # Documentation of custom modules
    └── modules/           # Reusable Terraform modules
        ├── vpc/           # VPC creation
        ├── iam/           # IAM roles & policies
        ├── eks/           # EKS cluster
        └── secrets-manager/ # Secrets storage
```

---

## 🔧 Key Components Explained

### 1. **main.tf** - The Orchestrator

This is the master file that ties everything together:

```hcl
module "vpc" {
  source = "./modules/vpc"
  # Creates the network infrastructure
}

module "iam" {
  source = "./modules/iam"
  # Creates IAM roles for security
}

module "eks" {
  source = "./modules/eks"
  # Creates the Kubernetes cluster
  # Uses the VPC from vpc module
  # Uses IAM roles from iam module
}

module "secrets_manager" {
  source = "./modules/secrets-manager"
  # Stores sensitive data securely
}
```

**Think of it like a recipe**: Each module is an ingredient, and main.tf is the chef combining them.

---

### 2. **variables.tf** - Customization

These are the settings you can change:

| Variable | Default | Purpose |
|----------|---------|---------|
| `aws_region` | us-east-1 | Which AWS region to use |
| `cluster_name` | day20-eks | Name of your cluster |
| `kubernetes_version` | 1.31 | K8s version |
| `vpc_cidr` | 10.0.0.0/16 | Network range |
| `enable_db_secret` | false | Store DB credentials? |
| `enable_api_secret` | false | Store API keys? |

**Example usage**: You can create a `terraform.tfvars` file to override defaults:
```hcl
aws_region = "eu-west-1"
cluster_name = "my-production-cluster"
```

---

### 3. **modules/** - Reusable Building Blocks

#### **A. VPC Module** (`modules/vpc/`)
- Creates a complete network
- 3 Public subnets (facing the internet)
- 3 Private subnets (for secure nodes)
- NAT Gateway (allows private instances to reach the internet)
- Required Kubernetes tags for load balancer discovery

#### **B. IAM Module** (`modules/iam/`)
- **Cluster Role**: Permissions for EKS control plane to:
  - Manage EC2 instances
  - Write logs
  - Manage security groups
  
- **Node Role**: Permissions for worker nodes to:
  - Pull container images
  - Write logs to CloudWatch
  - Access EBS volumes

#### **C. EKS Module** (`modules/eks/`)
**This is the heart of the project.** It creates:

- **EKS Cluster**: The Kubernetes control plane
  - Managed by AWS (you don't manage it)
  - Handles scheduling, networking, storage
  - Automatic updates and patches

- **Security Groups**: Firewalls for the cluster
  - Cluster SG: Controls access to API server
  - Node SG: Controls traffic between nodes

- **Two Node Groups**:
  ```
  General Group:
  - Instance type: c7i-flex.large
  - Desired: 2 nodes
  - Type: ON_DEMAND (always available)
  
  Spot Group:
  - Instance types: t3.micro, t3.small
  - Desired: 1 node
  - Type: SPOT (cheaper but can be interrupted)
  - Tainted (pods must tolerate being interrupted)
  ```

- **Add-ons** (Essential Kubernetes components):
  - **CoreDNS**: DNS resolution within the cluster
  - **kube-proxy**: Network routing
  - **VPC CNI**: Assigns IP addresses to pods
  - **EBS CSI**: Manages persistent storage

- **IRSA** (IAM Roles for Service Accounts):
  - Allows individual pods to have AWS permissions
  - More secure than giving all nodes same permissions

- **KMS Encryption**: Encrypts secrets at rest

#### **D. Secrets Manager Module** (`modules/secrets-manager/`)
- Stores sensitive data securely
- Database credentials
- API keys
- Optional (disabled by default)

---

## 🚀 How to Deploy This

### Step 1: Prerequisites
```bash
# Install Terraform
terraform -v

# Install AWS CLI
aws --version

# Install kubectl
kubectl version --client

# Configure AWS credentials
aws configure  # Enter your AWS access key & secret
```

### Step 2: Initialize Terraform
```bash
cd code
terraform init
# Downloads required plugins and prepares the environment
```

### Step 3: Review What Will Be Created
```bash
terraform plan
# Shows all resources that will be created
# Safe - doesn't actually create anything yet
```

### Step 4: Deploy the Cluster
```bash
terraform apply
# Type 'yes' when prompted
# Takes 15-20 minutes to complete
```

### Step 5: Connect to Your Cluster
```bash
# After deployment completes, configure kubectl
aws eks --region us-east-1 update-kubeconfig --name day20-eks-cluster

# Or use the Terraform output
$(terraform output -raw configure_kubectl)

# Verify connection
kubectl cluster-info
kubectl get nodes
```

### Step 6: Deploy a Sample Application
```bash
# Create a deployment
kubectl create deployment nginx --image=nginx

# Expose it with a load balancer
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# Check the service
kubectl get svc
```

### Step 7: Clean Up (Important!)
```bash
# Delete the load balancer first
kubectl delete svc --all

# Then destroy all AWS resources
terraform destroy
# Type 'yes' to confirm
```

---

## 📊 Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    AWS Account                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │           VPC (10.0.0.0/16)                      │  │
│  │                                                  │  │
│  │  Public Subnets (3)          Private Subnets (3)│  │
│  │  ┌─────────────────┐         ┌────────────────┐│  │
│  │  │ Internet Gateway │         │ EKS Node Group ││  │
│  │  │ NAT Gateway     │         │ (6 nodes total)││  │
│  │  │ Load Balancer   │         │                ││  │
│  │  └─────────────────┘         │ General: 2x    ││  │
│  │                              │ Spot: 1x       ││  │
│  │                              └────────────────┘│  │
│  │                                                 │  │
│  │  ┌────────────────────────────────────────────┐│  │
│  │  │   EKS Cluster Control Plane                ││  │
│  │  │ - API Server                               ││  │
│  │  │ - Scheduler                                ││  │
│  │  │ - Controller Manager                       ││  │
│  │  │ - KMS Encryption (Secrets)                 ││  │
│  │  │ - CloudWatch Logging                       ││  │
│  │  └────────────────────────────────────────────┘│  │
│  │                                                 │  │
│  │  Add-ons:                                       │  │
│  │  - CoreDNS (DNS)                               │  │
│  │  - VPC CNI (Networking)                        │  │
│  │  - kube-proxy (Service routing)                │  │
│  │  - EBS CSI (Storage)                           │  │
│  │                                                  │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
│  Security:                                              │
│  - IAM Roles for cluster and nodes                     │
│  - Security Groups (firewalls)                         │
│  - KMS Encryption                                      │
│  - IRSA for pod permissions                           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 💰 Cost Breakdown

**Approximate monthly costs**:

| Component | Cost/Hour | Cost/Month |
|-----------|-----------|------------|
| EKS Control Plane | $0.10 | ~$73 |
| 2x c7i-flex.large (on-demand) | $0.10 | ~$73 |
| 1x Spot instance | $0.01 | ~$7 |
| NAT Gateway | $0.045 | ~$33 |
| **Total** | **~$0.245** | **~$180** |

⚠️ **Don't forget to destroy resources when done!**

---

## 🔐 Security Features

1. **Private Subnets**: Worker nodes are not directly accessible from the internet
2. **IAM Roles**: Fine-grained permissions for cluster and nodes
3. **IRSA**: Individual pods get only the AWS permissions they need
4. **KMS Encryption**: Secrets encrypted at rest
5. **Security Groups**: Firewall rules for cluster and nodes
6. **CloudWatch Logs**: Audit logs for API, authenticator, scheduler
7. **IMDSv2**: Secure metadata service (prevents certain attacks)

---

## 📝 What This Project Teaches

This is a real-world example of:

✅ **Infrastructure as Code** - Define cloud resources in code  
✅ **Terraform Modules** - Reusable, organized code  
✅ **AWS Networking** - VPC, subnets, security groups  
✅ **Container Orchestration** - Kubernetes basics  
✅ **IAM & Security** - Cloud security best practices  
✅ **Cost Optimization** - Mix of on-demand and spot instances  

---

## ❓ Common Questions

**Q: Do I need to know Kubernetes?**
A: Not for deployment - Terraform handles that. But knowing K8s helps when deploying apps.

**Q: Can I use a different AWS region?**
A: Yes, change `aws_region` variable in variables.tf

**Q: How do I add more nodes?**
A: Modify `desired_size` in main.tf and run `terraform apply`

**Q: Can I use RDS/databases with this?**
A: Yes, but you'd need to add that as a separate module

**Q: How do I backup the cluster?**
A: Cluster configuration is defined in code (backed up). Applications need separate backup solutions.

---

## 📚 Additional Resources

- [Terraform AWS Provider Docs](https://registry.terraform.io/providers/hashicorp/aws)
- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/)
- [Kubernetes Official Docs](https://kubernetes.io/docs/)
- [Terraform EKS Module](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws)

---

## ✅ Summary

This project demonstrates how to:
1. **Define cloud infrastructure as code** using Terraform
2. **Organize code into reusable modules**
3. **Deploy a production-grade Kubernetes cluster** on AWS
4. **Implement security best practices** (IAM, encryption, networks)
5. **Manage costs** (spot instances, right-sizing)

It's a complete, real-world example suitable for learning or as a base for actual production deployments.
