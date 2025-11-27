# Complete Deployment Guide

Step-by-step guide to deploy the 3-Tier NodeJS MySQL application with Blue-Green deployment strategy.

## 📋 Prerequisites Checklist

Before starting, ensure you have:

- [ ] AWS Account with EKS access
- [ ] AWS CLI installed and configured
- [ ] kubectl installed
- [ ] Terraform installed (1.0+)
- [ ] Docker installed
- [ ] Jenkins server set up
- [ ] Docker Hub account
- [ ] SSH key pair for EC2 instances

## 🚀 Deployment Steps

### Phase 1: Infrastructure Setup

#### Step 1.1: Clone Repository

```bash
git clone https://github.com/NandeeshDK/3-Tier-NodeJS-MySql-Docker-main.git
cd 3-Tier-NodeJS-MySql-Docker-main
```

#### Step 1.2: Configure AWS

```bash
# Configure AWS CLI
aws configure
# Enter your AWS Access Key ID
# Enter your AWS Secret Access Key
# Default region: us-east-1
# Default output format: json

# Verify configuration
aws sts get-caller-identity
```

#### Step 1.3: Create EKS Cluster with Terraform

```bash
cd terraform

# Initialize Terraform
terraform init

# Review the plan
terraform plan

# Apply configuration (takes 10-15 minutes)
terraform apply
# Type 'yes' to confirm

# Note the outputs (cluster_id, vpc_id, etc.)
```

#### Step 1.4: Configure kubectl

```bash
# Update kubeconfig
aws eks update-kubeconfig --region us-east-1 --name devopsshack-cluster

# Verify connection
kubectl get nodes
# Should show 3 nodes in Ready state
```

### Phase 2: Kubernetes Setup

#### Step 2.1: Create Namespace

```bash
# Create webapps namespace
kubectl create namespace webapps

# Verify
kubectl get namespaces
```

#### Step 2.2: Apply RBAC Configuration

```bash
# Navigate to project root
cd ..

# Apply RBAC
kubectl apply -f RBAC/rbac.yml

# Verify service account
kubectl get sa jenkins -n webapps

# Verify roles and bindings
kubectl get role app-role -n webapps
kubectl get rolebinding app-rolebinding -n webapps
kubectl get clusterrole persistent-volume-access
kubectl get clusterrolebinding jenkins-persistent-volume-access
```

#### Step 2.3: Generate Jenkins Token

```bash
# Create token secret
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: jenkins-token
  namespace: webapps
  annotations:
    kubernetes.io/service-account.name: jenkins
type: kubernetes.io/service-account-token
EOF

# Get the token
kubectl get secret jenkins-token -n webapps -o jsonpath='{.data.token}' | base64 --decode
echo

# Save this token - you'll need it for Jenkins
```

#### Step 2.4: Deploy Database

```bash
# Deploy PersistentVolume and PVC
kubectl apply -f kubernetes/pv-pvc.yml -n webapps

# Verify PV and PVC
kubectl get pv
kubectl get pvc -n webapps

# Deploy MySQL
kubectl apply -f kubernetes/mysql-ds.yml -n webapps

# Wait for MySQL to be ready
kubectl wait --for=condition=ready pod -l app=mysql -n webapps --timeout=300s

# Verify MySQL is running
kubectl get pods -l app=mysql -n webapps
kubectl logs deployment/mysql -n webapps
```

#### Step 2.5: Deploy Application Service

```bash
# Deploy LoadBalancer service
kubectl apply -f kubernetes/app-service.yml -n webapps

# Wait for LoadBalancer to be provisioned (5-10 minutes)
kubectl get svc app -n webapps -w
# Press Ctrl+C once EXTERNAL-IP is assigned

# Save LoadBalancer URL
LB_URL=$(kubectl get svc app -n webapps -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "LoadBalancer URL: http://$LB_URL"
```

### Phase 3: Jenkins Setup

#### Step 3.1: Install Jenkins (if not already installed)

```bash
# On Ubuntu/Debian
sudo apt update
sudo apt install openjdk-11-jdk -y
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins -y

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Get initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

#### Step 3.2: Install Required Tools on Jenkins Server

```bash
# Install Docker
sudo apt install docker.io -y
sudo usermod -aG docker jenkins
sudo systemctl restart docker

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Install Trivy
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y

# Install Node.js (for SonarQube)
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt-get install -y nodejs

# Restart Jenkins
sudo systemctl restart jenkins
```

#### Step 3.3: Install Jenkins Plugins

Access Jenkins at `http://YOUR_JENKINS_IP:8080`

1. Navigate to: **Manage Jenkins** → **Plugins** → **Available Plugins**
2. Search and install:
   - Docker Pipeline
   - Kubernetes CLI
   - SonarQube Scanner
   - Git Plugin
3. Restart Jenkins after installation

#### Step 3.4: Configure Jenkins Tools

**SonarQube Scanner:**
1. Go to: **Manage Jenkins** → **Tools**
2. Find **SonarQube Scanner installations**
3. Click **Add SonarQube Scanner**
   - Name: `sonar-scanner`
   - Install automatically: ✓
   - Version: Latest
4. Save

#### Step 3.5: Configure Jenkins Credentials

**1. SonarQube Token (`sonar-token`)**
- **Manage Jenkins** → **Credentials** → **System** → **Global credentials**
- Click **Add Credentials**
  - Kind: Secret text
  - Secret: [Your SonarQube token]
  - ID: `sonar-token`
  - Description: SonarQube Authentication Token

**2. Docker Hub (`docker-cred`)**
- Click **Add Credentials**
  - Kind: Username with password
  - Username: `nandeeshdk` (your Docker Hub username, lowercase)
  - Password: [Your Docker Hub access token]
  - ID: `docker-cred`
  - Description: Docker Hub Credentials

**3. Kubernetes Token (`k8-token`)**
- Click **Add Credentials**
  - Kind: Secret text
  - Secret: [Jenkins token from Step 2.3]
  - ID: `k8-token`
  - Description: Kubernetes Service Account Token

#### Step 3.6: Create Jenkins Pipeline

1. **New Item**
   - Name: `Blue-Green-Deployment`
   - Type: Pipeline
   - Click OK

2. **Configure Pipeline**
   - **General**: Check "This project is parameterized"
   - **Pipeline**:
     - Definition: Pipeline script from SCM
     - SCM: Git
     - Repository URL: `https://github.com/NandeeshDK/3-Tier-NodeJS-MySql-Docker-main.git`
     - Branch: `*/main`
     - Script Path: `ci-cd/Jenkinsfile`
   - Save

### Phase 4: Deploy Application via Jenkins

#### Step 4.1: Deploy Blue Environment

1. Go to Jenkins pipeline: **Blue-Green-Deployment**
2. Click **Build with Parameters**
3. Set parameters:
   - `DEPLOY_ENV`: **blue**
   - `DOCKER_TAG`: **blue**
   - `SWITCH_TRAFFIC`: **false** (unchecked)
4. Click **Build**
5. Monitor the build progress
6. Wait for "SUCCESS" status

#### Step 4.2: Verify Blue Deployment

```bash
# Check pods
kubectl get pods -n webapps

# Should see:
# app-blue-xxx  1/1  Running
# mysql-xxx     1/1  Running

# Check service
kubectl get svc app -n webapps

# Test application
curl http://$LB_URL
# Should return the application
```

#### Step 4.3: Deploy Green Environment

1. Click **Build with Parameters**
2. Set parameters:
   - `DEPLOY_ENV`: **green**
   - `DOCKER_TAG`: **green**
   - `SWITCH_TRAFFIC`: **false**
3. Click **Build**
4. Wait for completion

#### Step 4.4: Verify Both Environments Running

```bash
# Check all pods
kubectl get pods -n webapps

# Should see:
# app-blue-xxx   1/1  Running
# app-green-xxx  1/1  Running
# mysql-xxx      1/1  Running

# Check which version is serving traffic
kubectl describe svc app -n webapps | grep Selector
# Should show version: blue
```

#### Step 4.5: Switch Traffic to Green

1. Click **Build with Parameters**
2. Set parameters:
   - `DEPLOY_ENV`: **green**
   - `DOCKER_TAG`: **green**
   - `SWITCH_TRAFFIC`: **true** (checked)
3. Click **Build**

#### Step 4.6: Verify Traffic Switch

```bash
# Check service selector
kubectl describe svc app -n webapps | grep Selector
# Should now show version: green

# Test application (should now show green version)
curl http://$LB_URL
```

### Phase 5: Testing

#### Test 5.1: Application Functionality

```bash
# Get LoadBalancer URL
echo $LB_URL

# Open in browser
# http://$LB_URL

# Test CRUD operations:
# - Create a user
# - View users list
# - Edit a user
# - Delete a user
```

#### Test 5.2: Blue-Green Switch

```bash
# Switch back to Blue
kubectl patch svc app -n webapps -p '{"spec":{"selector":{"version":"blue"}}}'

# Verify
kubectl describe svc app -n webapps | grep Selector

# Switch to Green
kubectl patch svc app -n webapps -p '{"spec":{"selector":{"version":"green"}}}'
```

#### Test 5.3: Rollback Scenario

```bash
# If green has issues, immediately switch back
kubectl patch svc app -n webapps -p '{"spec":{"selector":{"version":"blue"}}}'

# Zero downtime rollback!
```

### Phase 6: Monitoring

#### Monitor 6.1: Check Application Health

```bash
# Pod status
kubectl get pods -n webapps -w

# Logs
kubectl logs -f deployment/app-green -n webapps
kubectl logs -f deployment/mysql -n webapps

# Resource usage
kubectl top pods -n webapps
kubectl top nodes
```

#### Monitor 6.2: View Jenkins Builds

- Check Console Output for each build
- Review SonarQube results
- Check Trivy scan reports

## 🎯 Post-Deployment

### Security Hardening

```bash
# Update security groups to restrict SSH
# In terraform/main.tf, change:
cidr_blocks = ["YOUR_IP/32"]  # Instead of 0.0.0.0/0

# Apply changes
cd terraform
terraform apply
```

### Backup

```bash
# Backup MySQL data
kubectl exec deployment/mysql -n webapps -- mysqldump -u root -prootpassword devops > backup.sql

# Backup Kubernetes configs
kubectl get all -n webapps -o yaml > backup-k8s.yaml
```

### Scaling

```bash
# Scale application
kubectl scale deployment app-blue --replicas=5 -n webapps
kubectl scale deployment app-green --replicas=5 -n webapps

# Enable autoscaling
kubectl autoscale deployment app-blue --min=2 --max=10 --cpu-percent=70 -n webapps
```

## 🗑️ Cleanup

### Remove Application

```bash
# Delete all resources in namespace
kubectl delete namespace webapps
```

### Destroy Infrastructure

```bash
# Delete EKS cluster
cd terraform
terraform destroy
# Type 'yes' to confirm
```

## 📚 Next Steps

1. Configure monitoring (Prometheus, Grafana)
2. Set up logging (EFK stack)
3. Implement GitOps with ArgoCD
4. Add Ingress controller
5. Set up SSL/TLS certificates
6. Implement network policies
7. Add health checks and readiness probes

## 🆘 Troubleshooting

If you encounter issues, refer to:
- [Troubleshooting Guide](troubleshooting.md)
- [Kubernetes README](../kubernetes/README.md)
- [CI/CD README](../ci-cd/README.md)
- [Terraform README](../terraform/README.md)

---

**Congratulations!** 🎉 You've successfully deployed a production-ready 3-tier application with Blue-Green deployment strategy on AWS EKS!
