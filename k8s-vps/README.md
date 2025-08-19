# 🚀 Apache HTTP Server with Kubernetes VPA (Vertical Pod Autoscaler)

> **Deploy Apache on Kind cluster with automatic vertical scaling using VPA**

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
    │  │  │ ┌─────────────────────────────────┐ │││
    │  │  │ │           Pod                   │ │││
    │  │  │ │  ┌───────────┐ ┌─────────────┐  │ │││
    │  │  │ │  │ Apache    │ │   Resource  │  │ │││
    │  │  │ │  │Container  │ │  Optimizer  │  │ │││
    │  │  │ │  │           │ │    (VPA)    │  │ │││
    │  │  │ │  └───────────┘ └─────────────┘  │ │││
    │  │  │ └─────────────────────────────────┘ │││
    │  │  │            ↑                      │││
    │  │  │    ┌─────────────────┐            │││
    │  │  │    │ Apache Service  │            │││
    │  │  │    └─────────────────┘            │││
    │  │  │            ↑                      │││
    │  │  │    ┌─────────────────┐            │││
    │  │  │    │      VPA        │            │││
    │  │  │    │ (Resource Opt.) │            │││
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
- **VPA Controller** installed

### Setup Kind, Metrics Server & VPA

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

# Setup VPA
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh

# Verify VPA installation
kubectl get pods -n kube-system | grep vpa
kubectl get crd | grep verticalpodautoscalers
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
              cpu: 50m      # Intentionally low for VPA demo
              memory: 64Mi  # Intentionally low for VPA demo
            limits:
              cpu: 500m
              memory: 512Mi
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

### 4. **VPA Configuration** (`apache-vpa.yaml`)
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: apache-vpa
  namespace: apache
spec:
  targetRef:
    name: apache-deployment
    apiVersion: apps/v1
    kind: Deployment
  updatePolicy:
    updateMode: "Auto"  # Options: Off, Initial, Recreation, Auto
  resourcePolicy:
    containerPolicies:
    - containerName: apache
      minAllowed:
        cpu: 50m
        memory: 64Mi
      maxAllowed:
        cpu: 1000m
        memory: 1Gi
      controlledResources: ["cpu", "memory"]
      controlledValues: RequestsAndLimits
```

---

## 🚀 Deployment Steps

### Step 1: Apply all manifests
```bash
kubectl apply -f namespace.yaml
kubectl apply -f apache-deployment.yaml
kubectl apply -f apache-service.yaml
kubectl apply -f apache-vpa.yaml
```

### Step 2: Verify deployment
```bash
kubectl get pods -n apache
kubectl get svc -n apache
kubectl get vpa -n apache
kubectl describe vpa apache-vpa -n apache
```

### Step 3: Port forward for access
```bash
kubectl port-forward svc/apache-service 82:80 --address 0.0.0.0 -n apache
```

Access at: `http://YOUR_EC2_IP:82`

---

## 🧪 Load Testing & VPA in Action

### Generate resource usage:
```bash
# Create a load generator pod
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh
```

Inside the load generator container:
```bash
# Generate moderate continuous load
while true; do 
  for i in $(seq 1 50); do
    wget -q -O- http://apache-service.apache.svc.cluster.local &
  done
  sleep 2
done
```

### Monitor VPA recommendations:
```bash
# Check VPA status
kubectl get vpa -n apache

# Get detailed VPA recommendations
kubectl describe vpa apache-vpa -n apache

# Watch resource changes in pods
kubectl get pods -n apache -o wide -w
```

### VPA Monitoring Commands:
```bash
# Get VPA recommendations in YAML format
kubectl get vpa apache-vpa -n apache -o yaml

# Check current pod resource usage
kubectl top pods -n apache --containers

# Monitor VPA events
kubectl get events -n apache --sort-by=.metadata.creationTimestamp
```

---

## 📊 VPA Behavior & Expected Results

### VPA Update Modes:

1. **"Off"**: Only provides recommendations, no updates
2. **"Initial"**: Sets resources only at pod creation
3. **"Recreation"**: Deletes and recreates pods with new resources  
4. **"Auto"**: Currently same as "Recreation"

### VPA Recommendation Types:

```bash
$ kubectl describe vpa apache-vpa -n apache

Recommendation:
  Container Recommendations:
    Container Name:  apache
    Lower Bound:     # Minimum safe values
      Cpu:     25m
      Memory:  50Mi
    Target:          # Recommended optimal values
      Cpu:     100m
      Memory:  128Mi  
    Uncapped Target: # Without max limits
      Cpu:     150m
      Memory:  200Mi
    Upper Bound:     # Maximum reasonable values
      Cpu:     200m
      Memory:  256Mi
```

### VPA Resource Calculation:
- **Target**: Optimal resource recommendation
- **Lower Bound**: Minimum to avoid resource starvation  
- **Upper Bound**: Maximum reasonable allocation
- **Uncapped Target**: Recommendation without limits

---

## 🔧 Advanced VPA Configurations

### 1. **VPA with specific containers** (`apache-vpa-advanced.yaml`)
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: apache-vpa-advanced
  namespace: apache
spec:
  targetRef:
    name: apache-deployment
    apiVersion: apps/v1
    kind: Deployment
  updatePolicy:
    updateMode: "Recreation"
  resourcePolicy:
    containerPolicies:
    - containerName: apache
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2000m
        memory: 2Gi
      controlledResources: ["cpu", "memory"]
      controlledValues: RequestsAndLimits
      mode: Auto
```

### 2. **VPA Recommendation Only Mode**
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: apache-vpa-recommend
  namespace: apache
spec:
  targetRef:
    name: apache-deployment
    apiVersion: apps/v1
    kind: Deployment
  updatePolicy:
    updateMode: "Off"  # Only recommendations, no automatic updates
```

### 3. **VPA with Memory Only**
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: apache-vpa-memory
  namespace: apache
spec:
  targetRef:
    name: apache-deployment
    apiVersion: apps/v1
    kind: Deployment
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: apache
      controlledResources: ["memory"]  # Only manage memory
      controlledValues: RequestsAndLimits
```

---

## 📈 Monitoring & Troubleshooting

### VPA Status Monitoring:
```bash
# Check VPA controller pods
kubectl get pods -n kube-system | grep vpa

# Get VPA recommendations history
kubectl get vpa apache-vpa -n apache -o yaml | grep -A 20 recommendation

# Monitor pod restarts (VPA recreates pods)
kubectl get pods -n apache -o wide
```

### Common Issues & Solutions:

1. **VPA not providing recommendations**:
   ```bash
   # Check if metrics-server is working
   kubectl top pods -n apache
   
   # Check VPA controller logs
   kubectl logs -n kube-system -l app=vpa-recommender
   ```

2. **Pods not getting updated**:
   ```bash
   # Verify updateMode is not "Off"
   kubectl get vpa apache-vpa -n apache -o yaml | grep updateMode
   
   # Check VPA admission controller
   kubectl get pods -n kube-system | grep vpa-admission-controller
   ```

3. **Resource limits too restrictive**:
   ```bash
   # Check resource policies
   kubectl describe vpa apache-vpa -n apache | grep -A 10 "Container Policies"
   
   # Verify min/max allowed values
   ```

### VPA Metrics & Analysis:
```bash
# Compare before/after resource allocation
kubectl describe pod <old-pod-name> -n apache | grep -A 10 Resources
kubectl describe pod <new-pod-name> -n apache | grep -A 10 Resources

# Check VPA events for scaling decisions
kubectl describe vpa apache-vpa -n apache | grep Events -A 10

# Monitor resource efficiency
kubectl top pods -n apache --containers --use-protocol-buffers
```

---

## ⚖️ VPA vs Manual Resource Management

### Before VPA (Manual):
```yaml
resources:
  requests:
    cpu: 100m      # Fixed - might be too high/low
    memory: 128Mi  # Fixed - might be too high/low
```

### After VPA (Automatic):
```yaml
resources:
  requests:
    cpu: 156m      # VPA calculated optimal
    memory: 201Mi  # VPA calculated optimal
```

### Benefits of VPA:
- **Resource Optimization**: Right-sizing based on actual usage
- **Cost Efficiency**: No over-provisioning of resources
- **Performance**: Prevents resource starvation
- **Automation**: No manual tuning required

---

## 🧹 Cleanup

```bash
# Delete VPA first
kubectl delete vpa apache-vpa -n apache

# Delete entire namespace  
kubectl delete namespace apache

# Optional: Remove VPA from cluster
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-down.sh
```

---

## 📚 VPA Best Practices

1. **Start with "Off" mode** to see recommendations before auto-scaling
2. **Set reasonable min/max limits** to prevent extreme resource allocation
3. **Monitor pod restarts** as VPA recreates pods for updates
4. **Use with stateless applications** - VPA restarts can disrupt stateful apps
5. **Combine with HPA carefully** - can conflict if both manage the same deployment
6. **Test in staging first** before production deployment

---

**🎯 Your Apache server now automatically optimizes resource allocation! VPA will adjust CPU and memory requests/limits based on actual usage patterns, ensuring optimal performance and cost efficiency.**