# CI/CD Pipeline Configuration

Jenkins-based CI/CD pipeline for automated build, test, and deployment with Blue-Green strategy.

## 📋 Overview

This directory contains the Jenkinsfile that defines a complete CI/CD pipeline with the following stages:

1. **Git Checkout** - Clone source code
2. **SonarQube Analysis** - Code quality and security analysis
3. **Trivy FS Scan** - Filesystem vulnerability scanning
4. **Docker Build** - Build container image
5. **Trivy Image Scan** - Container image security scanning
6. **Docker Push** - Push image to Docker Hub
7. **Deploy SVC-APP** - Deploy Kubernetes service
8. **Deploy to Kubernetes** - Deploy application (Blue/Green)
9. **Switch Traffic** - Route traffic between environments
10. **Verify Deployment** - Health checks and validation

## 🎯 Pipeline Parameters

| Parameter | Type | Options | Description |
|-----------|------|---------|-------------|
| `DEPLOY_ENV` | choice | blue, green | Target deployment environment |
| `DOCKER_TAG` | choice | blue, green | Docker image tag to use |
| `SWITCH_TRAFFIC` | boolean | true, false | Enable traffic switching |

## 🔧 Prerequisites

### Jenkins Server Setup

1. **Required Plugins:**
   ```
   - Docker Pipeline
   - Kubernetes CLI Plugin
   - SonarQube Scanner
   - Git Plugin
   - Pipeline Plugin
   ```

2. **Install Plugins:**
   ```
   Manage Jenkins → Plugins → Available Plugins
   Search and install the above plugins
   ```

### Tool Configuration

1. **SonarQube Scanner**
   ```
   Manage Jenkins → Tools → SonarQube Scanner installations
   Name: sonar-scanner
   Install automatically: Yes
   ```

2. **Docker**
   ```
   Ensure Docker is installed on Jenkins agent
   sudo usermod -aG docker jenkins
   sudo systemctl restart jenkins
   ```

3. **kubectl**
   ```bash
   # Install kubectl on Jenkins server
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   chmod +x kubectl
   sudo mv kubectl /usr/local/bin/
   ```

4. **Trivy**
   ```bash
   # Install Trivy for security scanning
   sudo apt-get install wget apt-transport-https gnupg lsb-release
   wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
   echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
   sudo apt-get update
   sudo apt-get install trivy
   ```

5. **Node.js** (for SonarQube JavaScript analysis)
   ```bash
   curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
   sudo apt-get install -y nodejs
   ```

## 🔑 Credentials Configuration

### 1. SonarQube Token (`sonar-token`)

**Steps:**
1. Go to SonarQube → My Account → Security → Generate Token
2. In Jenkins: Manage Jenkins → Credentials → Add Credentials
   - Kind: Secret text
   - Secret: [Your SonarQube token]
   - ID: `sonar-token`

**Configure SonarQube Server:**
```
Manage Jenkins → Configure System → SonarQube servers
Name: sonar
Server URL: http://your-sonarqube-server:9000
Server authentication token: sonar-token
```

### 2. Docker Hub Credentials (`docker-cred`)

**Steps:**
1. Create Docker Hub Access Token:
   - Go to Docker Hub → Account Settings → Security
   - Create New Access Token
2. In Jenkins: Add Credentials
   - Kind: Username with password
   - Username: nandeeshdk (your Docker Hub username in lowercase)
   - Password: [Your Docker Hub access token]
   - ID: `docker-cred`

### 3. Kubernetes Token (`k8-token`)

**Steps:**
1. Create Service Account Token:
   ```bash
   # Apply RBAC configuration
   kubectl apply -f ../RBAC/rbac.yml
   
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
   ```

2. In Jenkins: Add Credentials
   - Kind: Secret text
   - Secret: [Your Kubernetes token from above]
   - ID: `k8-token`

## 🚀 Pipeline Usage

### First-Time Setup

1. **Create Pipeline Job:**
   ```
   Jenkins → New Item → Pipeline
   Name: Blue-Green-Deployment
   ```

2. **Configure Pipeline:**
   ```
   Pipeline → Definition: Pipeline script from SCM
   SCM: Git
   Repository URL: https://github.com/NandeeshDK/3-Tier-NodeJS-MySql-Docker-main.git
   Branch: main
   Script Path: ci-cd/Jenkinsfile
   ```

3. **Enable Parameters:**
   ```
   Check "This project is parameterized"
   Parameters are defined in Jenkinsfile
   ```

### Running the Pipeline

#### Deploy Blue Environment

```
Build with Parameters:
- DEPLOY_ENV: blue
- DOCKER_TAG: blue
- SWITCH_TRAFFIC: false
```

#### Deploy Green Environment

```
Build with Parameters:
- DEPLOY_ENV: green
- DOCKER_TAG: green
- SWITCH_TRAFFIC: false
```

#### Switch Traffic to Green

```
Build with Parameters:
- DEPLOY_ENV: green
- DOCKER_TAG: green
- SWITCH_TRAFFIC: true
```

## 📊 Pipeline Flow Diagram

```
┌─────────────────┐
│  Git Checkout   │
└────────┬────────┘
         ▼
┌─────────────────┐
│  SonarQube Scan │
└────────┬────────┘
         ▼
┌─────────────────┐
│ Trivy FS Scan   │
└────────┬────────┘
         ▼
┌─────────────────┐
│  Docker Build   │
└────────┬────────┘
         ▼
┌─────────────────┐
│ Trivy Image Scan│
└────────┬────────┘
         ▼
┌─────────────────┐
│  Docker Push    │
└────────┬────────┘
         ▼
┌─────────────────┐
│ Deploy Service  │
└────────┬────────┘
         ▼
┌─────────────────┐
│Deploy Blue/Green│
└────────┬────────┘
         ▼
┌─────────────────┐
│ Switch Traffic? │ ◄── (Optional)
└────────┬────────┘
         ▼
┌─────────────────┐
│     Verify      │
└─────────────────┘
```

## 🔍 Troubleshooting

### Common Issues

#### 1. Docker Permission Denied

```bash
# Add Jenkins user to docker group
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

#### 2. kubectl Unauthorized

```bash
# Verify token
TOKEN=$(kubectl get secret jenkins-token -n webapps -o jsonpath='{.data.token}' | base64 --decode)
kubectl get pods -n webapps --token=$TOKEN --server=https://YOUR_EKS_ENDPOINT

# Update token in Jenkins credentials
```

#### 3. SonarQube Node.js Error

```bash
# Install Node.js
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt-get install -y nodejs

# Restart Jenkins
sudo systemctl restart jenkins
```

#### 4. Docker Login Failed

```
# Use lowercase username
# Use Docker Hub Access Token instead of password
# Verify credentials in Jenkins
```

### View Pipeline Logs

```
Jenkins → Pipeline Job → Build History → Console Output
```

### Debug Pipeline

```groovy
// Add debug statements in Jenkinsfile
sh "kubectl get pods -n webapps"
sh "docker images"
sh "echo 'Debug info: ${env.VARIABLE_NAME}'"
```

## 📈 Pipeline Metrics

### SonarQube Quality Gates

- Code Coverage > 80%
- No Critical/Blocker issues
- Security hotspots reviewed
- Duplications < 3%

### Trivy Security Thresholds

- CRITICAL vulnerabilities: 0
- HIGH vulnerabilities: < 5
- MEDIUM vulnerabilities: < 10

## 🔐 Security Best Practices

1. **Never commit credentials** to the repository
2. **Use Jenkins Credentials Store** for all secrets
3. **Rotate tokens regularly** (every 90 days)
4. **Enable audit logging** in Jenkins
5. **Use RBAC** for Jenkins users
6. **Scan images** before deployment
7. **Use private Docker registry** for production

## 🎯 Blue-Green Deployment Strategy

### Workflow

1. **Deploy Blue** (v1.0)
   - Deploy to blue environment
   - Traffic goes to blue
   - Green is idle

2. **Deploy Green** (v2.0)
   - Deploy new version to green
   - Test green environment
   - Blue still serves traffic

3. **Switch Traffic**
   - Validate green is healthy
   - Switch traffic to green
   - Blue becomes idle

4. **Rollback** (if needed)
   - Switch traffic back to blue
   - No downtime
   - Investigate green issues

### Benefits

- **Zero Downtime** - No service interruption
- **Quick Rollback** - Instant switch back
- **Testing in Production** - Test before switching
- **Risk Mitigation** - New version validated before traffic switch

## 📝 Customization

### Update EKS Cluster URL

Edit `Jenkinsfile` and update the server URL:

```groovy
serverUrl: 'https://YOUR_EKS_CLUSTER_ENDPOINT'
clusterName: 'YOUR_CLUSTER_NAME'
```

### Change Docker Image Name

```groovy
environment {
    IMAGE_NAME = "your-dockerhub-username/your-app-name"
}
```

### Modify SonarQube Project

```groovy
sh "${SCANNER_HOME}/bin/sonar-scanner \
    -Dsonar.projectKey=your-project-key \
    -Dsonar.projectName=your-project-name"
```

## 🔗 Related Documentation

- [Kubernetes Manifests](../kubernetes/README.md)
- [Terraform Infrastructure](../terraform/README.md)
- [RBAC Configuration](../RBAC/README.md)
- [Main README](../README.md)

---

For questions or issues, please create an issue in the repository.
