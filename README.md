# 🚀 Apache HTTP Server with Kubernetes HPA & VPA

> **Deploy Apache on Kind cluster with automatic horizontal & vertical scaling using HPA and VPA**

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

# Verify installations
kubectl get pods -n kube-system
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

### 5. **VPA** (`apache-vpa.yaml`)
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
    updateMode: "Auto"
```
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
kubectl apply -f apache-vpa.yaml
```

![Kubectl Apply](https://i.imgur.com/VjKGb8K.png)

### Step 2: Verify deployment
```bash
kubectl get pods -n apache
kubectl get svc -n apache
kubectl get hpa -n apache
kubectl get vpa -n apache
```

![Kubectl Get Resources](https://i.imgur.com/7vWJZcH.png)

### Step 3: Port forward for access
```bash
kubectl port-forward svc/apache-service 82:80 --address 0.0.0.0 -n apache
```

Access at: `http://YOUR_EC2_IP:82`

![Apache Welcome Page](https://i.imgur.com/ZjBj5cF.png)

---

## 🧪 Load Testing

### Generate load using BusyBox:
```bash
kubectl run -i --tty load-generator --image=busybox /bin/sh
```

![Load Generator](https://i.imgur.com/Xj3KQsP.png)

Inside container:
```bash
while true; do wget -q -O- http://apache-service.apache.svc.cluster.local; done
```

### Monitor HPA & VPA scaling:
```bash
# Watch HPA scaling
kubectl get hpa -n apache -w

# Check VPA recommendations
kubectl get vpa -n apache
kubectl describe vpa apache-vpa -n apache
```

![HPA Scaling](https://i.imgur.com/M8kRtNx.png)

## 📊 Expected Results

### HPA (Horizontal Pod Autoscaler):
- **CPU < 50%**: Scales down to 1 pod
- **CPU > 50%**: Scales up (max 10 pods)  
- **Scaling time**: ~30 seconds up, ~5 minutes down

### VPA (Vertical Pod Autoscaler):
- **Automatically adjusts** CPU and memory requests
- **Restarts pods** with new resource limits
- **Optimizes** resource utilization

### HPA & VPA in Action:
![HPA Dashboard](https://i.imgur.com/4BcLp2Y.png)
![VPA Dashboard](https://i.imgur.com/V9kPq3L.png)

---

## 🧹 Cleanup

```bash
kubectl delete namespace apache
```

---

**🎯 That's it! Your Apache server will now automatically scale both horizontally (more pods) and vertically (more resources per pod) based on usage.**