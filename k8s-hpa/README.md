# 🚀 Apache HTTP Server with Kubernetes HPA (Horizontal Pod Autoscaler)

> **Deploy Apache on Kind cluster with automatic horizontal scaling using HPA**

![Apache](https://httpd.apache.org/images/httpd_logo_wide_new.png)

## 📐 Architecture Overview

```
    ┌─────────────────────────────────────────────┐
    │            AWS EC2 Instance                 │
    │             (t2.medium)                     │
    │  ┌─────────────────────────────────────────┐│
    │  │           Kind Cluster                  ││
    │  │                                         ││
    │  │  ┌─────────────────────────────────────┐││
    │  │  │        apache namespace            │││
    │  │  │                                     │││
    │  │  │ ┌───┐ ┌───┐ ┌───┐ ... ┌───┐      │││
    │  │  │ │Pod│ │Pod│ │Pod│     │Pod│      │││
    │  │  │ └───┘ └───┘ └───┘     └───┘      │││
    │  │  │            ↑                      │││
    │  │  │    ┌─────────────────┐            │││
    │  │  │    │ Apache Service  │            │││
    │  │  │    └─────────────────┘            │││
    │  │  │            ↑                      │││
    │  │  │    ┌─────────────────┐            │││
    │  │  │    │      HPA        │            │││
    │  │  │    │  (Auto Scaler)  │            │││
    │  │  │    └─────────────────┘            │││
    │  │  └─────────────────────────────────────┘││
    │  └─────────────────────────────────────────┘│
    └─────────────────────────────────────────────┘
```

---

## 📋 Prerequisites

- **AWS EC2 Instance**: t2.medium (2 CPU, 4GB RAM)
- **Kind** cluster setup
- **Metrics Server** enabled

### Setup Kind & Metrics Server

```bash
# Install Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Edit the Metrics Server Deployment
kubectl -n kube-system edit deployment metrics-server

# Add these args under container.args:
# - --kubelet-insecure-tls
# - --kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP

# Restart the deployment
kubectl -n kube-system rollout restart deployment metrics-server

# Verify installation
kubectl get pods -n kube-system | grep metrics-server
kubectl top nodes
```

---

## 📁 Project Files

### 1. **Namespace** (`namespace.yaml`)
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: apache
```

### 2. **Apache Deployment** (`apache-deployment.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apache-deployment
  namespace: apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: apache
  template:
    metadata:
      name: apache
      labels:
        app: apache
    spec:
      containers:
        - name: apache
          image: httpd:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
```

### 3. **Apache Service** (`apache-service.yaml`)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: apache-service
  namespace: apache
spec:
  selector:
    app: apache
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

### 4. **HPA Configuration** (`apache-hpa.yaml`)
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: apache-hpa
  namespace: apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: apache-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

---

## 🚀 Deployment Steps

### Step 1: Apply all manifests
```bash
kubectl apply -f namespace.yaml
kubectl apply -f apache-deployment.yaml
kubectl apply -f apache-service.yaml
kubectl apply -f apache-hpa.yaml
```

### Step 2: Verify deployment
```bash
kubectl get pods -n apache
kubectl get svc -n apache
kubectl get hpa -n apache
kubectl describe hpa apache-hpa -n apache
```

### Step 3: Port forward for access
```bash
kubectl port-forward svc/apache-service 82:80 --address 0.0.0.0 -n apache
```

Access at: `http://YOUR_EC2_IP:82`

---

## 🧪 Load Testing & HPA in Action

### Generate CPU load using BusyBox:
```bash
# Create a load generator pod
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh
```

Inside the load generator container:
```bash
# Generate continuous traffic to apache service
while true; do 
  for i in $(seq 1 100); do
    wget -q -O- http://apache-service.apache.svc.cluster.local &
  done
  sleep 1
done
```

### Monitor HPA scaling in real-time:
```bash
# Watch HPA scaling events
kubectl get hpa -n apache -w

# Monitor pod scaling
kubectl get pods -n apache -w

# Check resource usage
kubectl top pods -n apache
```

### HPA Commands for Monitoring:
```bash
# Get detailed HPA status
kubectl describe hpa apache-hpa -n apache

# Check HPA events
kubectl get events -n apache --sort-by=.metadata.creationTimestamp

# Monitor CPU utilization
kubectl top pods -n apache --containers
```

## 📊 HPA Behavior & Expected Results

### HPA Scaling Logic:
- **Target CPU**: 50% utilization
- **Scale Up**: When average CPU > 50% for 3 minutes
- **Scale Down**: When average CPU < 50% for 5 minutes
- **Min Replicas**: 1 pod
- **Max Replicas**: 10 pods

### HPA Scaling Timeline:
```
Load Increase → CPU > 50% → Wait 3 mins → Scale Up
Load Decrease → CPU < 50% → Wait 5 mins → Scale Down
```

### Sample HPA Output:
```bash
$ kubectl get hpa -n apache
NAME         REFERENCE                     TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
apache-hpa   Deployment/apache-deployment   75%/50%   1         10        3          5m
```

### HPA Status Messages:
- `ScalingActive`: HPA is actively monitoring and scaling
- `AbleToScale`: Ready to scale up/down
- `ScalingLimited`: Hit min/max replica limits

---

## 🔧 Advanced HPA Configuration

### Memory-based scaling:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: apache-hpa-advanced
  namespace: apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: apache-deployment
  minReplicas: 2
  maxReplicas: 15
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

---

## 📈 Monitoring & Troubleshooting

### Common Issues:
1. **HPA shows "unknown" metrics**:
   ```bash
   kubectl describe hpa apache-hpa -n apache
   # Check if metrics-server is running
   kubectl get pods -n kube-system | grep metrics
   ```

2. **Pods not scaling**:
   ```bash
   # Check resource requests are defined
   kubectl describe pod <pod-name> -n apache
   
   # Verify HPA conditions
   kubectl get hpa apache-hpa -n apache -o yaml
   ```

3. **Scaling too slow**:
   ```bash
   # Check HPA behavior settings
   kubectl describe hpa apache-hpa -n apache
   ```

### Useful Monitoring Commands:
```bash
# Real-time resource monitoring
watch kubectl top pods -n apache

# HPA decision logs
kubectl describe hpa apache-hpa -n apache | grep Events -A 10

# Check deployment scaling events
kubectl describe deployment apache-deployment -n apache
```

---

## 🧹 Cleanup

```bash
# Delete HPA first
kubectl delete hpa apache-hpa -n apache

# Delete entire namespace
kubectl delete namespace apache
```

---

**🎯 Your Apache server now automatically scales horizontally based on CPU utilization! HPA will add more pods when traffic increases and remove them when traffic decreases.**