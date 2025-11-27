# Kubernetes Manifests

This directory contains all Kubernetes manifest files for deploying the 3-tier application with Blue-Green deployment strategy.

## 📁 Files Overview

### Application Deployments

- **`app-deployment-blue.yml`** - Blue environment deployment
- **`app-deployment-green.yml`** - Green environment deployment
- **`app-service.yml`** - LoadBalancer service for traffic routing

### Database

- **`mysql-ds.yml`** - MySQL deployment and headless service
- **`pv-pvc.yml`** - PersistentVolume and PersistentVolumeClaim for MySQL data

## 🚀 Deployment Order

### 1. Prerequisites

```bash
# Create namespace
kubectl create namespace webapps

# Apply RBAC (from project root)
kubectl apply -f ../RBAC/rbac.yml
```

### 2. Deploy Database

```bash
# Create persistent storage
kubectl apply -f pv-pvc.yml -n webapps

# Deploy MySQL
kubectl apply -f mysql-ds.yml -n webapps

# Verify MySQL is running
kubectl get pods -l app=mysql -n webapps
kubectl logs -f deployment/mysql -n webapps
```

### 3. Deploy Application Service

```bash
# Create LoadBalancer service
kubectl apply -f app-service.yml -n webapps

# Get LoadBalancer URL (wait a few minutes for provisioning)
kubectl get svc app -n webapps
```

### 4. Deploy Blue Environment

```bash
# Deploy Blue version
kubectl apply -f app-deployment-blue.yml -n webapps

# Verify deployment
kubectl get pods -l version=blue -n webapps
kubectl logs -f deployment/app-blue -n webapps
```

### 5. Deploy Green Environment (for Blue-Green strategy)

```bash
# Deploy Green version
kubectl apply -f app-deployment-green.yml -n webapps

# Verify deployment
kubectl get pods -l version=green -n webapps
kubectl logs -f deployment/app-green -n webapps
```

## 🔄 Blue-Green Traffic Switching

### Check Current Traffic Routing

```bash
kubectl describe svc app -n webapps | grep -i selector
```

### Switch Traffic to Blue

```bash
kubectl patch service app -n webapps -p '{"spec":{"selector":{"version":"blue"}}}'
```

### Switch Traffic to Green

```bash
kubectl patch service app -n webapps -p '{"spec":{"selector":{"version":"green"}}}'
```

## 🔍 Monitoring and Troubleshooting

### Check Pod Status

```bash
# All pods in webapps namespace
kubectl get pods -n webapps

# Watch pod status in real-time
kubectl get pods -n webapps -w

# Get detailed pod information
kubectl describe pod <pod-name> -n webapps
```

### View Logs

```bash
# Application logs (Blue)
kubectl logs -f deployment/app-blue -n webapps

# Application logs (Green)
kubectl logs -f deployment/app-green -n webapps

# MySQL logs
kubectl logs -f deployment/mysql -n webapps

# Previous container logs (if crashed)
kubectl logs deployment/app-blue -n webapps --previous
```

### Debug Connection Issues

```bash
# Test MySQL connection from a pod
kubectl run -it --rm mysql-test --image=mysql:8.0 --restart=Never -n webapps -- \
  mysql -h mysql -u root -prootpassword -e "SELECT 1"

# Port forward for local testing
kubectl port-forward svc/app 8080:80 -n webapps
kubectl port-forward svc/mysql 3306:3306 -n webapps
```

### Resource Usage

```bash
# Check resource consumption
kubectl top pods -n webapps
kubectl top nodes

# Get events
kubectl get events -n webapps --sort-by='.lastTimestamp'
```

## 📊 Service Information

### Get LoadBalancer Details

```bash
# Get external IP/DNS
kubectl get svc app -n webapps -o wide

# Get full service configuration
kubectl get svc app -n webapps -o yaml
```

### Test the Application

```bash
# Get LoadBalancer URL
LB_URL=$(kubectl get svc app -n webapps -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Test connectivity
curl http://$LB_URL

# Test API endpoint
curl http://$LB_URL/api/users
```

## 🔐 Security Considerations

1. **Secrets Management**: Update MySQL credentials in `mysql-ds.yml`
2. **Network Policies**: Consider adding network policies to restrict traffic
3. **Resource Limits**: Adjust CPU/Memory limits based on your workload
4. **RBAC**: Ensure proper RBAC is applied (see `../RBAC/`)

## 🎯 Scaling

### Scale Deployments

```bash
# Scale Blue deployment
kubectl scale deployment app-blue --replicas=3 -n webapps

# Scale Green deployment
kubectl scale deployment app-green --replicas=3 -n webapps

# Scale MySQL (not recommended for production)
kubectl scale deployment mysql --replicas=1 -n webapps
```

### Horizontal Pod Autoscaling

```bash
# Create HPA for Blue deployment
kubectl autoscale deployment app-blue \
  --min=2 --max=10 \
  --cpu-percent=70 \
  -n webapps

# Check HPA status
kubectl get hpa -n webapps
```

## 🗑️ Cleanup

### Delete Specific Resources

```bash
# Delete Blue deployment
kubectl delete -f app-deployment-blue.yml -n webapps

# Delete Green deployment
kubectl delete -f app-deployment-green.yml -n webapps

# Delete service
kubectl delete -f app-service.yml -n webapps

# Delete MySQL
kubectl delete -f mysql-ds.yml -n webapps
kubectl delete -f pv-pvc.yml -n webapps
```

### Delete Everything

```bash
# Delete entire namespace (careful!)
kubectl delete namespace webapps
```

## 📝 Configuration Notes

### Environment Variables

The application expects these environment variables (configured in deployments):

- `DB_HOST`: MySQL service hostname
- `DB_USER`: Database username
- `DB_PASSWORD`: Database password
- `DB_NAME`: Database name
- `PORT`: Application port (default: 3000)

### PersistentVolume Storage

- **Storage Class**: `gp2` (AWS EBS)
- **Capacity**: 5Gi
- **Access Mode**: ReadWriteOnce
- **Path**: `/mnt/data` (hostPath for local testing)

### LoadBalancer

- **Type**: AWS Classic Load Balancer (automatically provisioned by EKS)
- **Port**: 80 (HTTP)
- **Target Port**: 3000 (Node.js application)

## 🔗 Related Documentation

- [CI/CD Pipeline](../ci-cd/README.md)
- [Terraform Infrastructure](../terraform/README.md)
- [RBAC Configuration](../RBAC/README.md)
- [Troubleshooting Guide](../docs/troubleshooting.md)

---

For questions or issues, please refer to the main [README](../README.md) or create an issue.
