# Troubleshooting Guide

Common issues and solutions for the 3-Tier NodeJS MySQL Docker application.

## 🔧 Application Issues

### Database Connection Errors

**Symptom**: Application logs show "Cannot connect to MySQL"

**Solutions**:
```bash
# 1. Check MySQL pod status
kubectl get pods -l app=mysql -n webapps

# 2. Check MySQL logs
kubectl logs deployment/mysql -n webapps

# 3. Verify MySQL service
kubectl get svc mysql -n webapps

# 4. Test connection from app pod
kubectl exec -it deployment/app-blue -n webapps -- sh
nc -zv mysql 3306

# 5. Check environment variables
kubectl exec deployment/app-blue -n webapps -- env | grep DB_
```

### Application Pod CrashLoopBackOff

**Symptom**: Pods keep restarting

**Solutions**:
```bash
# 1. Check pod logs
kubectl logs pod-name -n webapps
kubectl logs pod-name -n webapps --previous

# 2. Describe pod for events
kubectl describe pod pod-name -n webapps

# 3. Common causes:
# - Missing environment variables
# - Database not ready
# - Port conflicts
# - Resource limits too low
```

### ImagePullBackOff Error

**Symptom**: Cannot pull Docker image

**Solutions**:
```bash
# 1. Verify image exists
docker pull nandeeshdk/app:blue

# 2. Check image name in deployment
kubectl get deployment app-blue -n webapps -o yaml | grep image

# 3. If using private registry, add imagePullSecrets
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=nandeeshdk \
  --docker-password=YOUR_TOKEN \
  -n webapps
```

## 🌐 Networking Issues

### LoadBalancer External-IP Pending

**Symptom**: Service shows `<pending>` for EXTERNAL-IP

**Solutions**:
```bash
# 1. Check service events
kubectl describe svc app -n webapps

# 2. Verify cloud provider integration
kubectl get nodes -o wide

# 3. Wait 5-10 minutes for AWS to provision LoadBalancer

# 4. Check AWS Console for LoadBalancer creation
aws elb describe-load-balancers
```

### Cannot Access Application via LoadBalancer

**Symptom**: LoadBalancer URL returns timeout/connection refused

**Solutions**:
```bash
# 1. Check if pods are ready
kubectl get pods -n webapps

# 2. Verify service selector matches pod labels
kubectl describe svc app -n webapps
kubectl get pods -n webapps --show-labels

# 3. Check security groups (AWS)
# Ensure LoadBalancer security group allows port 80

# 4. Test from within cluster
kubectl run test-pod --image=busybox -it --rm -- wget -O- http://app.webapps
```

### Blue-Green Traffic Not Switching

**Symptom**: Traffic still goes to old version after switch

**Solutions**:
```bash
# 1. Verify service selector
kubectl get svc app -n webapps -o yaml | grep -A5 selector

# 2. Check pod labels
kubectl get pods -n webapps --show-labels

# 3. Manually patch service
kubectl patch svc app -n webapps -p '{"spec":{"selector":{"version":"green"}}}'

# 4. Verify endpoints
kubectl get endpoints app -n webapps
```

## 🐳 Docker Issues

### Docker Build Fails

**Symptom**: Jenkins pipeline fails at Docker Build stage

**Solutions**:
```bash
# 1. Check Dockerfile syntax
docker build -t test .

# 2. Verify base image exists
docker pull node:16-alpine

# 3. Check Docker daemon
sudo systemctl status docker

# 4. Add Jenkins to docker group
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### Docker Push Authentication Failed

**Symptom**: "unauthorized: incorrect username or password"

**Solutions**:
```bash
# 1. Use lowercase username
# nandeeshdk (not Nandeeshdk)

# 2. Create Docker Hub Access Token
# Use token instead of password

# 3. Test login manually
echo "YOUR_TOKEN" | docker login -u nandeeshdk --password-stdin

# 4. Update Jenkins credential
# Manage Jenkins → Credentials → Update docker-cred
```

## ☸️ Kubernetes Issues

### kubectl Unauthorized Error

**Symptom**: "error: You must be logged in to the server (Unauthorized)"

**Solutions**:
```bash
# 1. Update kubeconfig
aws eks update-kubeconfig --region us-east-1 --name devopsshack-cluster

# 2. Verify AWS credentials
aws sts get-caller-identity

# 3. Check RBAC permissions
kubectl auth can-i get pods -n webapps

# 4. For Jenkins: Regenerate service account token
kubectl get secret jenkins-token -n webapps -o jsonpath='{.data.token}' | base64 --decode
```

### PersistentVolume Permission Denied

**Symptom**: "persistentvolumes is forbidden"

**Solutions**:
```bash
# 1. Apply RBAC configuration
kubectl apply -f RBAC/rbac.yml

# 2. Verify ClusterRole
kubectl get clusterrole persistent-volume-access

# 3. Verify ClusterRoleBinding
kubectl get clusterrolebinding jenkins-persistent-volume-access

# 4. Test permissions
kubectl auth can-i create persistentvolumes --as=system:serviceaccount:webapps:jenkins
```

### Pod Stuck in Pending State

**Symptom**: Pod shows Pending status for extended time

**Solutions**:
```bash
# 1. Describe pod to see reason
kubectl describe pod pod-name -n webapps

# 2. Common causes and fixes:

# Insufficient resources:
kubectl describe nodes | grep -A5 "Allocated resources"

# PVC not bound:
kubectl get pvc -n webapps

# Node selector mismatch:
kubectl get nodes --show-labels
```

## 🔨 Jenkins Pipeline Issues

### SonarQube Analysis Fails

**Symptom**: "Error when running: 'node -v'"

**Solutions**:
```bash
# 1. Install Node.js on Jenkins server
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt-get install -y nodejs

# 2. Verify Node.js
node -v
npm -v

# 3. Restart Jenkins
sudo systemctl restart jenkins
```

### Trivy Scan Timeout

**Symptom**: Trivy scanner hangs or times out

**Solutions**:
```bash
# 1. Update Trivy database
trivy image --download-db-only

# 2. Increase timeout in Jenkinsfile
sh "trivy image --timeout 15m ..."

# 3. Skip vulnerability database update
sh "trivy image --skip-update ..."
```

### withKubeConfig Authentication Failed

**Symptom**: "unable to find credentials with id 'k8-token'"

**Solutions**:
```bash
# 1. Verify credential exists in Jenkins
# Manage Jenkins → Credentials → Check for k8-token

# 2. Recreate token
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

kubectl get secret jenkins-token -n webapps -o jsonpath='{.data.token}' | base64 --decode

# 3. Update in Jenkins (Secret text type)
```

## 🔍 Terraform Issues

### Terraform Init Fails

**Symptom**: "Error configuring the backend"

**Solutions**:
```bash
# 1. Clear Terraform cache
rm -rf .terraform
rm .terraform.lock.hcl

# 2. Re-initialize
terraform init

# 3. If using remote backend, verify S3 bucket exists
aws s3 ls s3://your-terraform-bucket
```

### Cluster Creation Timeout

**Symptom**: "timeout while waiting for cluster to become ready"

**Solutions**:
```bash
# 1. Check AWS CloudTrail for errors
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateCluster

# 2. Verify IAM permissions
aws iam get-user

# 3. Check service quotas
aws service-quotas list-service-quotas --service-code eks

# 4. Increase timeout in main.tf
```

### Security Group Cycle Error

**Symptom**: "Cycle: aws_security_group.X, aws_security_group.Y"

**Solution**: Already fixed in the code - uses separate security group rules

## 📊 Monitoring and Debugging

### View Cluster Events

```bash
# All events in namespace
kubectl get events -n webapps --sort-by='.lastTimestamp'

# Watch events in real-time
kubectl get events -n webapps -w
```

### Check Resource Usage

```bash
# Pod resource usage
kubectl top pods -n webapps

# Node resource usage
kubectl top nodes

# Detailed node info
kubectl describe node node-name
```

### Debug Network Connectivity

```bash
# Run debug pod
kubectl run debug-pod --image=nicolaka/netshoot -it --rm -- /bin/bash

# Inside pod:
nslookup mysql.webapps.svc.cluster.local
curl http://app.webapps.svc.cluster.local
ping mysql
```

### Collect Logs for Support

```bash
# Collect all logs
kubectl logs --all-containers=true -l app=app -n webapps > app-logs.txt
kubectl logs --all-containers=true -l app=mysql -n webapps > mysql-logs.txt

# Get cluster info
kubectl cluster-info dump > cluster-dump.txt

# Get events
kubectl get events --all-namespaces > events.txt
```

## 🆘 Emergency Recovery

### Complete Application Reset

```bash
# 1. Delete all deployments
kubectl delete deployment --all -n webapps

# 2. Delete services (careful with LoadBalancer)
kubectl delete svc app -n webapps

# 3. Redeploy from scratch
kubectl apply -f kubernetes/pv-pvc.yml
kubectl apply -f kubernetes/mysql-ds.yml
kubectl apply -f kubernetes/app-service.yml
kubectl apply -f kubernetes/app-deployment-blue.yml
```

### Rollback Deployment

```bash
# View rollout history
kubectl rollout history deployment/app-blue -n webapps

# Rollback to previous version
kubectl rollout undo deployment/app-blue -n webapps

# Rollback to specific revision
kubectl rollout undo deployment/app-blue --to-revision=2 -n webapps
```

### Force Delete Stuck Resources

```bash
# Force delete pod
kubectl delete pod pod-name -n webapps --grace-period=0 --force

# Remove finalizers from stuck namespace
kubectl get namespace webapps -o json | jq '.spec.finalizers = []' | kubectl replace --raw "/api/v1/namespaces/webapps/finalize" -f -
```

## 📞 Getting Help

If you're still stuck:

1. **Check Documentation**
   - [Main README](../README.md)
   - [Kubernetes README](../kubernetes/README.md)
   - [CI/CD README](../ci-cd/README.md)
   - [Terraform README](../terraform/README.md)

2. **Search Issues**
   - GitHub Issues for similar problems
   - Stack Overflow
   - Kubernetes Slack

3. **Create an Issue**
   - Provide error messages
   - Include relevant logs
   - Describe steps to reproduce
   - Share configuration (sanitized)

---

Remember: Most issues are configuration-related. Double-check credentials, endpoints, and resource names!
