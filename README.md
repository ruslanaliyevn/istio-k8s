# Istio Service Mesh on Kubernetes

<div align="center">
  
[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Istio](https://img.shields.io/badge/Istio-466BB0?style=for-the-badge&logo=istio&logoColor=white)](https://istio.io/)
[![MetalLB](https://img.shields.io/badge/MetalLB-0078D4?style=for-the-badge&logo=loadbalancer&logoColor=white)](https://metallb.universe.tf/)
[![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)](LICENSE)

**Production-ready Istio service mesh setup with MetalLB integration**

</div>

---

## 📖 Overview

Istio provides traffic management, security, and observability for microservices on Kubernetes. This guide covers complete installation with MetalLB LoadBalancer integration.

## 📋 Prerequisites

- Kubernetes cluster (v1.27+)
- kubectl configured
- Basic Kubernetes knowledge

## 🚀 Installation

### 1. Install Istio

```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -

# Add istioctl to PATH
export PATH="$PATH:$HOME/istio-1.28.1/bin"

# Install Istio with demo profile
istioctl install --set profile=demo -y

# Enable sidecar injection
kubectl label namespace default istio-injection=enabled
```

### 2. Verify Installation

```bash
kubectl get pods -n istio-system
kubectl get svc -n istio-system
```

Expected output:
```
NAME                                    READY   STATUS    RESTARTS   AGE
istio-ingressgateway-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
istiod-xxxxxxxxxx-xxxxx                 1/1     Running   0          2m
```

## 🌐 MetalLB Configuration

MetalLB provides LoadBalancer support for bare-metal clusters.

### Install MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml

# Wait for pods to be ready
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s
```

### Configure IP Address Pool

Create `metallb/address-pool.yaml`:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: metallb-pool
  namespace: metallb-system
spec:
  addresses:
  - 203.0.113.10/32  # Replace with your actual public IP (e.g., 45.67.89.123/32)
```

### Configure L2 Advertisement

Create `metallb/l2-advertisement.yaml`:

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - metallb-pool
```

### Apply Configuration

```bash
kubectl apply -f metallb/address-pool.yaml
kubectl apply -f metallb/l2-advertisement.yaml

# Verify external IP is assigned
kubectl get svc istio-ingressgateway -n istio-system
```

Expected output:
```
NAME                   TYPE           EXTERNAL-IP      PORT(S)
istio-ingressgateway   LoadBalancer   203.0.113.10     80:xxxxx/TCP...
```

## 📦 Deploy Sample Application

### NGINX Example

**1. Create Deployment**

`examples/nginx/deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.3
        ports:
        - containerPort: 80
```

**2. Create Service**

`examples/nginx/service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
```

**3. Create Gateway**

`examples/nginx/gateway.yaml`:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: nginx-gateway
  namespace: default
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "nginx.example.com"  # Replace with your domain
```

**4. Create VirtualService**

`examples/nginx/virtualservice.yaml`:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: nginx-vs
  namespace: default
spec:
  hosts:
  - "nginx.example.com"  # Must match Gateway host
  gateways:
  - nginx-gateway
  http:
  - match:
    - uri:
        prefix: "/"
    route:
    - destination:
        host: nginx-service
        port:
          number: 80
```

### Deploy Everything

```bash
kubectl apply -f examples/nginx/deployment.yaml
kubectl apply -f examples/nginx/service.yaml
kubectl apply -f examples/nginx/gateway.yaml
kubectl apply -f examples/nginx/virtualservice.yaml
```

### Verify Deployment

```bash
# Check pods (should show 2/2 with sidecar)
kubectl get pods

# Check all resources
kubectl get gateway
kubectl get virtualservice
kubectl get svc
```

## 🌍 Access Your Application

### Configure DNS

First, point your domain to the external IP:

**DNS Configuration:**
```
Type: A
Name: nginx.example.com
Value: 203.0.113.10  (your MetalLB external IP)
TTL: 3600
```

Or add to your local `/etc/hosts` for testing:
```
203.0.113.10  nginx.example.com
```

### Get External IP

```bash
export INGRESS_IP=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Gateway IP: $INGRESS_IP"
```

### Test Access

```bash
# Access via domain name
curl http://nginx.example.com

# Or in browser
http://nginx.example.com
```

## 🔐 HTTPS/TLS Setup (Optional)

### Create Self-Signed Certificate

```bash
openssl req -x509 -newkey rsa:4096 -keyout tls.key -out tls.crt \
  -days 365 -nodes -subj "/CN=nginx.example.com"
```

### Create Secret

```bash
kubectl create secret tls nginx-tls \
  --cert=tls.crt \
  --key=tls.key \
  -n istio-system
```

### Update Gateway for HTTPS

`examples/nginx/gateway-https.yaml`:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: nginx-gateway
  namespace: default
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "nginx.example.com"
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: nginx-tls
    hosts:
    - "nginx.example.com"
```

Apply the updated gateway:
```bash
kubectl apply -f examples/nginx/gateway-https.yaml
```

Access via HTTPS:
```bash
curl https://nginx.example.com
```

## 🛠️ Useful Commands

### Check Istio Status

```bash
# Check components
kubectl get pods -n istio-system

# Check gateway
kubectl get gateway

# Check virtual services
kubectl get virtualservice

# Analyze configuration
istioctl analyze

# Check proxy status
istioctl proxy-status
```

### Debug Issues

```bash
# Describe gateway
kubectl describe gateway nginx-gateway

# Check ingress gateway logs
kubectl logs -n istio-system -l istio=ingressgateway

# Check application logs
kubectl logs <pod-name> -c nginx
kubectl logs <pod-name> -c istio-proxy
```

## 📊 Observability with Kiali

### Install Kiali Dashboard

```bash
# Install Kiali
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/addons/kiali.yaml

# Wait for Kiali to be ready
kubectl wait --for=condition=available --timeout=300s deployment/kiali -n istio-system

# Access Kiali dashboard
istioctl dashboard kiali
```

Kiali will open at `http://localhost:20001` and show:
- Service topology and dependencies
- Traffic flow visualization
- Health status of services
- Configuration validation

## 🔍 Troubleshooting

### Pods Not Getting Sidecar

**Check namespace label:**
```bash
kubectl get namespace default -L istio-injection
```

**Fix:**
```bash
kubectl label namespace default istio-injection=enabled --overwrite
kubectl rollout restart deployment nginx-deployment
```

### Gateway Not Accessible

**Check if external IP is assigned:**
```bash
kubectl get svc istio-ingressgateway -n istio-system
```

**If pending:**
- Verify MetalLB is running
- Check IP pool configuration
- Ensure IP is not in use

### 404 Not Found

**Check service endpoints:**
```bash
kubectl get endpoints nginx-service
```

**Verify VirtualService:**
```bash
kubectl get virtualservice nginx-vs -o yaml
```

**Test internal connectivity:**
```bash
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- curl http://nginx-service
```

## 🧹 Cleanup

```bash
# Remove application
kubectl delete -f examples/nginx/

# Uninstall Istio
istioctl uninstall --purge -y
kubectl delete namespace istio-system

# Remove MetalLB
kubectl delete -f metallb/

# Remove namespace label
kubectl label namespace default istio-injection-
```

## 📁 Repository Structure

```
istio-k8s/
├── README.md
├── metallb/
│   ├── address-pool.yaml
│   └── l2-advertisement.yaml
└── examples/
    └── nginx/
        ├── deployment.yaml
        ├── service.yaml
        ├── gateway.yaml
        └── virtualservice.yaml
```

## 📚 Resources

- [Istio Documentation](https://istio.io/latest/docs/)
- [MetalLB Documentation](https://metallb.universe.tf/)
- [Traffic Management](https://istio.io/latest/docs/concepts/traffic-management/)
- [Kiali Documentation](https://kiali.io/docs/)

---

<div align="center">
  
**⭐ Star this repo if you find it helpful!**



</div>
