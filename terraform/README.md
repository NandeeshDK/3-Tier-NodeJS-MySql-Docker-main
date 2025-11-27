# Terraform Infrastructure Configuration

Infrastructure as Code (IaC) for provisioning AWS EKS cluster and related resources.

## 📋 Overview

This Terraform configuration creates a complete AWS EKS cluster with:
- VPC with public subnets across 2 availability zones
- Internet Gateway and Route Tables
- Security Groups for cluster and nodes
- EKS Cluster with managed control plane
- EKS Node Group with EC2 instances
- IAM Roles and Policies for EKS

## 🏗️ Architecture

```
AWS Region (us-east-1)
│
├── VPC (10.0.0.0/16)
│   ├── Subnet 1 (10.0.0.0/24) - us-east-1a
│   ├── Subnet 2 (10.0.1.0/24) - us-east-1b
│   └── Internet Gateway
│
├── EKS Cluster
│   ├── Control Plane (Managed by AWS)
│   └── Security Group
│
├── EKS Node Group
│   ├── 3 x t2.medium instances
│   └── Security Group
│
└── IAM Roles
    ├── Cluster Role (AmazonEKSClusterPolicy)
    └── Node Role (AmazonEKSWorkerNodePolicy, CNI, ECR)
```

## 📁 Files

- **`main.tf`** - Main Terraform configuration
- **`variables.tf`** - Input variables
- **`output.tf`** - Output values
- **`README.md`** - This file

## 🔧 Prerequisites

### Required Tools

```bash
# Terraform (1.0+)
terraform version

# AWS CLI (configured with credentials)
aws --version
aws configure

# kubectl
kubectl version --client
```

### AWS Permissions

Your AWS IAM user/role needs permissions for:
- EKS (CreateCluster, DescribeCluster, etc.)
- EC2 (VPC, Subnets, Security Groups, Instances)
- IAM (CreateRole, AttachRolePolicy)

## 🚀 Quick Start

### 1. Configure AWS Credentials

```bash
# Configure AWS CLI
aws configure
# Enter:
# - AWS Access Key ID
# - AWS Secret Access Key
# - Default region: us-east-1
# - Default output format: json

# Verify
aws sts get-caller-identity
```

### 2. Initialize Terraform

```bash
cd terraform

# Initialize Terraform
terraform init

# This will:
# - Download AWS provider
# - Initialize backend
# - Prepare modules
```

### 3. Review Configuration

```bash
# Format code
terraform fmt

# Validate configuration
terraform validate

# Review the plan
terraform plan
```

### 4. Deploy Infrastructure

```bash
# Apply configuration
terraform apply

# Review the plan and type 'yes' to confirm
# This will take 10-15 minutes to create the EKS cluster
```

### 5. Configure kubectl

```bash
# Update kubeconfig
aws eks update-kubeconfig --region us-east-1 --name devopsshack-cluster

# Verify connection
kubectl get nodes
```

## 📊 Resource Details

### VPC Configuration

```hcl
CIDR Block: 10.0.0.0/16
Subnets:
  - 10.0.0.0/24 (us-east-1a)
  - 10.0.1.0/24 (us-east-1b)
DNS: Enabled
Public IPs: Enabled for subnets
```

### EKS Cluster

```hcl
Name: devopsshack-cluster
Version: Latest stable
Region: us-east-1
Endpoint: Public
```

### Node Group

```hcl
Instance Type: t2.medium
Desired Size: 3
Min Size: 3
Max Size: 3
AMI: Amazon Linux 2 (EKS optimized)
SSH Key: awslogin (configure in variables.tf)
```

### Security Groups

#### Cluster Security Group
- Egress: Allow all
- Ingress: Port 443 from node security group

#### Node Security Group
- Egress: Allow all
- Ingress:
  - Self (all ports)
  - Port 1025-65535 from cluster
  - Port 443 from cluster
  - Port 22 from anywhere (SSH)

## 🔧 Customization

### Variables

Edit `variables.tf` or pass values via command line:

```bash
# Change SSH key
terraform apply -var="ssh_key_name=your-key-name"

# Change region
terraform apply -var="region=ap-south-1"

# Change cluster name
terraform apply -var="cluster_name=my-cluster"
```

### Modify Node Group

Edit `main.tf`:

```hcl
scaling_config {
  desired_size = 5  # Change number of nodes
  max_size     = 10
  min_size     = 2
}

instance_types = ["t3.medium"]  # Change instance type
```

### Change Region

Update `main.tf`:

```hcl
provider "aws" {
  region = "ap-south-1"  # Change to your preferred region
}

# Also update availability zones
availability_zone = element(["ap-south-1a", "ap-south-1b"], count.index)
```

## 📊 Outputs

After successful deployment:

```bash
terraform output

# You'll see:
cluster_id     = eks cluster ID
node_group_id  = node group ID
vpc_id         = VPC ID
subnet_ids     = List of subnet IDs
```

### Get EKS Endpoint

```bash
aws eks describe-cluster --name devopsshack-cluster --query "cluster.endpoint" --output text
```

## 🔍 Verification

### Check Cluster Status

```bash
# Cluster status
aws eks describe-cluster --name devopsshack-cluster --query "cluster.status"

# Node group status
aws eks describe-nodegroup \
  --cluster-name devopsshack-cluster \
  --nodegroup-name devopsshack-node-group \
  --query "nodegroup.status"
```

### Verify Nodes

```bash
# Get nodes
kubectl get nodes

# Node details
kubectl describe nodes

# Check node resources
kubectl top nodes
```

### Test Cluster

```bash
# Deploy test pod
kubectl run nginx --image=nginx

# Check pod
kubectl get pods

# Delete test pod
kubectl delete pod nginx
```

## 🔐 Security Considerations

### 1. SSH Key Management

```bash
# Create SSH key if not exists
aws ec2 create-key-pair --key-name awslogin --query 'KeyMaterial' --output text > awslogin.pem
chmod 400 awslogin.pem
```

### 2. Security Group Rules

**Restrict SSH Access:**

Edit `main.tf`:

```hcl
ingress {
  description = "Allow SSH access"
  from_port   = 22
  to_port     = 22
  protocol    = "tcp"
  cidr_blocks = ["YOUR_IP/32"]  # Replace with your IP
}
```

### 3. IAM Best Practices

- Use IAM roles instead of access keys
- Enable MFA for AWS account
- Follow principle of least privilege
- Rotate credentials regularly

### 4. Cluster Security

```bash
# Enable cluster logging
aws eks update-cluster-config \
  --name devopsshack-cluster \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
```

## 💰 Cost Considerations

### Estimated Monthly Costs (us-east-1)

```
EKS Control Plane:  $73/month ($0.10/hour)
EC2 Instances:      $75/month (3 x t2.medium @ $0.0464/hour)
EBS Volumes:        $15/month (120GB total)
Data Transfer:      Variable
Load Balancer:      $16-25/month

Total: ~$180-190/month
```

### Cost Optimization

```bash
# Use t3.medium instead of t2.medium (better performance, similar cost)
# Reduce number of nodes when not in use
terraform apply -var="desired_size=1"

# Delete cluster when not needed
terraform destroy
```

## 🗑️ Cleanup

### Destroy Infrastructure

```bash
# WARNING: This will delete all resources!

# Review what will be destroyed
terraform plan -destroy

# Destroy resources
terraform destroy

# Type 'yes' to confirm
```

### Manual Cleanup (if needed)

```bash
# Delete any LoadBalancers created by Kubernetes
kubectl delete svc --all -n webapps

# Wait 5 minutes, then run
terraform destroy
```

## 🔧 Troubleshooting

### Issue: Terraform Init Fails

```bash
# Clear cache
rm -rf .terraform
rm .terraform.lock.hcl

# Re-initialize
terraform init
```

### Issue: Cluster Creation Timeout

```bash
# Check AWS CloudTrail for errors
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateCluster

# Increase timeout
# Add to resource block in main.tf:
timeouts {
  create = "30m"
  delete = "30m"
}
```

### Issue: Cannot Connect to Cluster

```bash
# Update kubeconfig
aws eks update-kubeconfig --region us-east-1 --name devopsshack-cluster

# Verify AWS credentials
aws sts get-caller-identity

# Check cluster status
aws eks describe-cluster --name devopsshack-cluster
```

### Issue: Nodes Not Joining Cluster

```bash
# Check node group status
aws eks describe-nodegroup --cluster-name devopsshack-cluster --nodegroup-name devopsshack-node-group

# Check IAM roles
aws iam get-role --role-name devopsshack-node-group-role

# View node logs (SSH into node)
ssh -i awslogin.pem ec2-user@<node-ip>
sudo journalctl -u kubelet
```

## 📚 State Management

### Terraform State

By default, state is stored locally in `terraform.tfstate`.

### Remote State (Recommended for Production)

Configure S3 backend in `main.tf`:

```hcl
terraform {
  backend "s3" {
    bucket = "your-terraform-state-bucket"
    key    = "eks-cluster/terraform.tfstate"
    region = "us-east-1"
    encrypt = true
    dynamodb_table = "terraform-lock"
  }
}
```

## 🔄 Upgrades

### Upgrade Cluster Version

```bash
# Check available versions
aws eks describe-addon-versions --kubernetes-version 1.27

# Upgrade cluster
aws eks update-cluster-version --name devopsshack-cluster --kubernetes-version 1.28

# Update node group (create new, delete old)
```

## 🔗 Related Documentation

- [Kubernetes Manifests](../kubernetes/README.md)
- [CI/CD Pipeline](../ci-cd/README.md)
- [RBAC Configuration](../RBAC/README.md)
- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)

---

For questions or issues, please create an issue in the repository.
