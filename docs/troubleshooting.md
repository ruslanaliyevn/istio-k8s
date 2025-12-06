# Istio Troubleshooting Guide

Common issues and solutions for Istio on Kubernetes.

## Installation Issues

### Pods Not Starting

Check pod status:
```bash
kubectl get pods -n istio-system
kubectl describe pod <pod-name> -n istio-system
kubectl logs <pod-name> -n istio-system
```

**Solution:**
- Ensure sufficient resources (4GB RAM minimum)
- Check node status: `kubectl get nodes`
- Restart deployment: `kubectl rollout restart deployment -n istio-system`

## Gateway Issues

### External IP Pending

```bash
kubectl get svc istio-ingressgateway -n istio-system
```

**Solution:**
- Verify MetalLB is installed: `kubectl get pods -n metallb`
- Check MetalLB logs: `kubectl logs -n metallb deployment/metallb-controller`
- Verify IP pool configuration

### Gateway Not Working

```bash
kubectl get gateway -n istio-system
kubectl describe gateway http-gateway -n istio-system
```

**Solution:**
- Ensure gateway is in `istio-system` namespace
- Check selector matches ingress gateway
- Verify hosts configuration

## Service Issues

### Service Not Accessible

```bash
kubectl get virtualservice
kubectl describe virtualservice <name>
```

**Common causes:**

1. **Wrong gateway reference:**
   ```yaml
   gateways:
   - istio-system/http-gateway  # Must include namespace
   ```

2. **DNS not configured:**
   - Verify DNS A record points to External IP
   - Test: `nslookup yourapp.example.com`

3. **Wrong host in VirtualService:**
   ```yaml
   hosts:
   - "yourapp.example.com"  # Must match DNS
   ```

### Sidecar Not Injected

Check if pods have 2 containers:
```bash
kubectl get pods
```

Should show `2/2` (app + istio-proxy).

**Solution:**
```bash
# Check namespace label
kubectl get namespace -L istio-injection

# Label namespace
kubectl label namespace default istio-injection=enabled --overwrite

# Restart deployment
kubectl rollout restart deployment/<deployment-name>
```

### 404 Not Found

```bash
# Check VirtualService routing
kubectl get virtualservice <name> -o yaml

# Check service exists
kubectl get svc

# Check endpoints
kubectl get endpoints <service-name>
```

**Solution:**
- Verify VirtualService host matches request
- Check service selector matches pod labels
- Verify service port matches destination port

## TLS/SSL Issues

### Certificate Not Working

```bash
# Check secret exists
kubectl get secret -n istio-system

# Describe secret
kubectl describe secret example-tls -n istio-system
```

**Solution:**
- Ensure secret is in `istio-system` namespace
- Verify secret type is `kubernetes.io/tls`
- Check certificate validity: `openssl x509 -in tls.crt -noout -dates`

### HTTPS Not Working

```bash
# Test HTTPS
curl -k https://yourapp.example.com

# Check gateway configuration
kubectl get gateway http-gateway -n istio-system -o yaml
```

**Solution:**
- Verify `credentialName` in gateway matches secret name
- Check TLS mode is `SIMPLE`
- Ensure port 443 is configured

## Network Issues

### Cannot Access from Outside Cluster

```bash
# From outside cluster
curl -v http://yourapp.example.com

# Check ingress gateway
kubectl get svc istio-ingressgateway -n istio-system
```

**Solution:**
- Verify External IP is assigned
- Check firewall rules allow traffic
- Test DNS resolution
- Verify LoadBalancer service is working

### Service to Service Communication Issues

```bash
# Check service mesh
istioctl proxy-status

# Analyze configuration
istioctl analyze
```

**Solution:**
- Ensure all namespaces have sidecar injection enabled
- Check network policies don't block traffic
- Verify services are properly configured

## Debug Commands

```bash
# Check Istio version
istioctl version

# Verify installation
istioctl verify-install

# Check configuration
istioctl analyze

# Proxy status
istioctl proxy-status

# Gateway logs
kubectl logs -n istio-system -l app=istio-ingressgateway --tail=100

# Istiod logs
kubectl logs -n istio-system -l app=istiod --tail=100

# Check all Istio resources
kubectl get gateway,virtualservice,destinationrule,serviceentry -A
```

## Common Error Messages

### "upstream connect error or disconnect/reset before headers"

**Cause:** Service not reachable or port mismatch

**Solution:**
- Verify service exists and is running
- Check service port matches VirtualService destination port
- Ensure pods are ready

### "no healthy upstream"

**Cause:** No pods available for service

**Solution:**
- Check pod status: `kubectl get pods`
- Verify pod health checks
- Check service selector matches pod labels

### "404 Not Found"

**Cause:** No matching route in VirtualService

**Solution:**
- Check VirtualService host and path
- Verify gateway is attached
- Check URI match rules

## Getting Help

```bash
# Collect debug information
istioctl bug-report

# Save to file
istioctl bug-report > istio-debug.tar.gz
```

**Resources:**
- [Istio Troubleshooting](https://istio.io/latest/docs/ops/diagnostic-tools/)
- [Istio FAQ](https://istio.io/latest/about/faq/)
- [Istio GitHub Issues](https://github.com/istio/istio/issues)

---

**Back to:** [Main README](../README.md)
