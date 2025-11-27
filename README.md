# 3-Tier NodeJS MySQL Docker Application

A production-ready, scalable 3-tier web application with Blue-Green deployment strategy on AWS EKS.

## 🚀 Project Overview

This project demonstrates a complete DevOps pipeline for a full-stack application:
- **Frontend**: HTML, CSS, JavaScript (Client-side rendered)
- **Backend**: Node.js with Express.js REST API
- **Database**: MySQL 
- **Containerization**: Docker
- **Orchestration**: Kubernetes (AWS EKS)
- **CI/CD**: Jenkins Pipeline
- **Deployment Strategy**: Blue-Green Deployment
- **Infrastructure as Code**: Terraform

## 📋 Table of Contents

- [Features](#features)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Deployment](#deployment)
- [CI/CD Pipeline](#cicd-pipeline)
- [Monitoring](#monitoring)
- [Contributing](#contributing)

## ✨ Features

### Application Features
- ✅ User Management (CRUD operations)
- ✅ RESTful API
- ✅ Responsive UI
- ✅ MySQL Database persistence
- ✅ Connection pooling

### DevOps Features
- ✅ Dockerized application
- ✅ Kubernetes manifests for production deployment
- ✅ Blue-Green deployment strategy
- ✅ Automated CI/CD with Jenkins
- ✅ Infrastructure provisioning with Terraform
- ✅ Security scanning (Trivy, SonarQube)
- ✅ RBAC for Kubernetes

## 🏗️ Architecture

### Application Architecture
```
┌─────────────────┐
│   Client (UI)   │
│  HTML/CSS/JS    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Node.js Server │
│   Express API   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  MySQL Database │
│  Persistent Vol │
└─────────────────┘
```

### Infrastructure Architecture
```
AWS EKS Cluster
├── Blue Deployment (v1)
├── Green Deployment (v2)
├── LoadBalancer Service
├── MySQL StatefulSet
└── Persistent Volumes
```

## 📁 Project Structure

```
.
├── client/                 # Frontend application
│   ├── src/               # Source files
│   ├── public/            # Static assets
│   └── package.json
│
├── server/                # Backend application
│   ├── config/           # Database configuration
│   ├── controllers/      # Business logic
│   ├── models/           # Data models
│   ├── routes/           # API routes
│   └── package.json
│
├── kubernetes/           # Kubernetes manifests
│   ├── app-deployment-blue.yml
│   ├── app-deployment-green.yml
│   ├── app-service.yml
│   ├── mysql-ds.yml
│   ├── pv-pvc.yml
│   └── README.md
│
├── ci-cd/               # CI/CD configuration
│   ├── Jenkinsfile
│   └── README.md
│
├── terraform/           # Infrastructure as Code
│   ├── main.tf
│   ├── variables.tf
│   ├── output.tf
│   └── README.md
│
├── RBAC/               # Kubernetes RBAC
│   ├── rbac.yml
│   └── README.md
│
├── docs/               # Documentation
│   ├── deployment-guide.md
│   ├── troubleshooting.md
│   └── architecture.md
│
├── Dockerfile          # Docker image definition
├── docker-compose.yml  # Local development setup
└── README.md          # This file
```

## 🔧 Prerequisites

### Required Tools
- **Docker** (20.10+)
- **Kubernetes** (1.24+)
- **kubectl** (configured for your cluster)
- **Terraform** (1.0+)
- **Node.js** (16+ for local development)
- **MySQL** (8.0+ for local development)

### AWS Requirements
- AWS Account with EKS access
- AWS CLI configured
- IAM permissions for EKS, EC2, VPC

### Jenkins Requirements
- Jenkins server with plugins:
  - Docker Pipeline
  - Kubernetes CLI
  - SonarQube Scanner
  - Trivy

## 🚀 Quick Start

### Local Development with Docker Compose

```bash
# Clone the repository
git clone https://github.com/NandeeshDK/3-Tier-NodeJS-MySql-Docker-main.git
cd 3-Tier-NodeJS-MySql-Docker-main

# Start the application
docker-compose up -d

# Access the application
# Frontend: http://localhost:80
# Backend API: http://localhost:5000
```

### Manual Setup

1. **Setup MySQL Database**
```bash
sudo apt install mysql-server
mysql -u root -p < database/schema.sql
```

2. **Install Server Dependencies**
```bash
cd server
npm install
npm start
```

3. **Build Client**
```bash
cd client
npm install
npm run build
```

## 🌐 Deployment

### Kubernetes Deployment

See [kubernetes/README.md](kubernetes/README.md) for detailed instructions.

```bash
# Create namespace
kubectl create namespace webapps

# Apply RBAC
kubectl apply -f RBAC/rbac.yml

# Deploy MySQL
kubectl apply -f kubernetes/pv-pvc.yml
kubectl apply -f kubernetes/mysql-ds.yml

# Deploy Application (Blue)
kubectl apply -f kubernetes/app-deployment-blue.yml
kubectl apply -f kubernetes/app-service.yml

# Get LoadBalancer URL
kubectl get svc app -n webapps
```

### Terraform Infrastructure

See [terraform/README.md](terraform/README.md) for EKS cluster setup.

```bash
cd terraform
terraform init
terraform plan
terraform apply
```

## 🔄 CI/CD Pipeline

### Jenkins Pipeline Stages

See [ci-cd/README.md](ci-cd/README.md) for detailed configuration.

1. **Git Checkout** - Clone repository
2. **SonarQube Analysis** - Code quality scan
3. **Trivy FS Scan** - Filesystem security scan
4. **Docker Build** - Build Docker image
5. **Trivy Image Scan** - Container security scan
6. **Docker Push** - Push to Docker Hub
7. **Deploy to Kubernetes** - Deploy selected environment
8. **Switch Traffic** - Blue-Green traffic switching
9. **Verify Deployment** - Health checks

### Pipeline Parameters
- `DEPLOY_ENV`: blue | green
- `DOCKER_TAG`: blue | green
- `SWITCH_TRAFFIC`: true | false

## 📊 Monitoring

### Check Application Status

```bash
# Get all pods
kubectl get pods -n webapps

# Check logs
kubectl logs -f deployment/app-blue -n webapps
kubectl logs -f deployment/app-green -n webapps

# Get service details
kubectl get svc app -n webapps

# Describe service
kubectl describe svc app -n webapps
```

### Access LoadBalancer

```bash
# Get LoadBalancer URL
LB_URL=$(kubectl get svc app -n webapps -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Application URL: http://$LB_URL"

# Test the application
curl http://$LB_URL
```

## 🔐 Security

- **RBAC**: Kubernetes Role-Based Access Control
- **Security Scanning**: Trivy for vulnerabilities
- **Code Quality**: SonarQube analysis
- **Network Policies**: Restricted pod communication
- **Secrets Management**: Kubernetes secrets

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## 👥 Author

**Nandeesh DK**
- GitHub: [@NandeeshDK](https://github.com/NandeeshDK)

## 🙏 Acknowledgments

- AWS EKS Documentation
- Kubernetes Best Practices
- Jenkins Pipeline Examples
- DevOps Community

---

⭐ **Star this repository if you find it helpful!**


