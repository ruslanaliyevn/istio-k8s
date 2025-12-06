<div align="center">
  <img src="https://istio.io/latest/img/istio-bluelogo-whitebackground-unframed.svg" alt="Istio Logo" width="300"/>
  <h1>Istio Service Mesh on Kubernetes</h1>
  
  <p>
    <img src="https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white" alt="Kubernetes"/>
    <img src="https://img.shields.io/badge/Istio-466BB0?style=for-the-badge&logo=istio&logoColor=white" alt="Istio"/>
    <img src="https://img.shields.io/badge/License-MIT-green?style=for-the-badge" alt="License"/>
  </p>
  
  <p><strong>Production-ready Istio service mesh setup with MetalLB integration</strong></p>
</div>

---

## 📖 Overview

Istio provides traffic management, security, and observability for microservices on Kubernetes. This guide covers installation and basic configuration with MetalLB LoadBalancer.

## 📋 Prerequisites

- Kubernetes cluster (v1.19+)
- kubectl configured
- MetalLB installed

## 🚀 Installation

### 1. Download and Install Istio

```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# Install Istio
istioctl install --set profile=default -y

# Enable sidecar injection
kubectl label namespace default istio-injection=enabled
```

### 2. Verify Installation

```bash
kubectl get pods -n istio-system
kubectl get svc -n istio-system
```

### 3. Get External IP

```bash
kubectl get svc istio-ingressgateway -n istio-system
```

Note the EXTERNAL-IP (e.g., `203.0.113.10`) from MetalLB.

### 4. Configure DNS

Point your domain to the External IP:

```
Type: A
Name: *.example.com
Value: 203.0.113.10
TTL: 3600
```

## 🌐 Gateway Setup

### Deploy HTTP Gateway

```bash
kubectl apply -f gateway.yaml
```

This creates a gateway accepting traffic on:
- Port 80 (HTTP)
- Port 443 (HTTPS with TLS)

Hosts: `*.example.com`

## 📦 Deploy Applications

### Option 1: Nginx Demo

```bash
kubectl apply -f examples/nginx-demo.yaml
```

Access: `http://nginx.example.com`

### Option 2: Bookinfo Sample

```bash
# Download Istio samples
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/bookinfo/platform/kube/bookinfo.yaml

# Apply VirtualService
kubectl apply -f examples/bookinfo-app.yaml
```

Access: `http://bookinfo.example.com/productpage`

## 🔐 HTTPS/TLS Setup

### Create Self-Signed Certificate (for testing)

```bash
openssl req -x509 -newkey rsa:4096 -keyout tls.key -out tls.crt \
  -days 365 -nodes -subj "/CN=*.example.com"
```

### Create Kubernetes Secret

```bash
kubectl create secret tls example-tls \
  --cert=tls.crt \
  --key=tls.key \
  -n istio-system
```

### Update Gateway

The gateway.yaml already includes HTTPS configuration. Just update the `credentialName` if needed.

## 🔧 Custom Application Example

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  ports:
  - port: 8080
  selector:
    app: myapp
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - "myapp.example.com"
  gateways:
  - istio-system/http-gateway
  http:
  - route:
    - destination:
        host: myapp
        port:
          number: 8080
```

Save as `myapp.yaml` and apply:

```bash
kubectl apply -f myapp.yaml
```

## 🔍 Verification

```bash
# Check Istio version
istioctl version

# Check gateway
kubectl get gateway -n istio-system

# Check VirtualServices
kubectl get virtualservice

# Check proxy status
istioctl proxy-status

# Analyze configuration
istioctl analyze
```

## 📊 Observability (Optional)

### Install Kiali Dashboard

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/kiali.yaml

# Access Kiali
kubectl port-forward -n istio-system svc/kiali 20001:20001
```

Open: `http://localhost:20001`

## 🔍 Troubleshooting

### Service Not Accessible

```bash
# Check VirtualService
kubectl describe virtualservice <name>

# Check Gateway
kubectl describe gateway http-gateway -n istio-system

# Check pods have sidecar (should show 2/2)
kubectl get pods
```

### External IP Pending

```bash
# Check MetalLB
kubectl get svc -n istio-system
kubectl logs -n metallb deployment/metallb-controller
```

### Sidecar Not Injected

```bash
# Verify namespace label
kubectl get namespace -L istio-injection

# Re-label if needed
kubectl label namespace default istio-injection=enabled --overwrite

# Restart pods
kubectl rollout restart deployment/<deployment-name>
```

For more details, see [Troubleshooting Guide](docs/troubleshooting.md).

## 🧹 Cleanup

```bash
# Remove applications
kubectl delete -f examples/

# Uninstall Istio
istioctl uninstall --purge -y
kubectl delete namespace istio-system

# Remove namespace label
kubectl label namespace default istio-injection-
```

## 📚 Resources

- [Istio Documentation](https://istio.io/latest/docs/)
- [Traffic Management](https://istio.io/latest/docs/concepts/traffic-management/)
- [Security](https://istio.io/latest/docs/concepts/security/)

---

<div align="center">
  <p>Made with ❤️ for Kubernetes</p>
</div>
