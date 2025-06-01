# 🌐 Public Access Setup Guide - VaultLan Kubernetes Cluster

This guide explains how to expose your Kubernetes homelab cluster to the internet securely using Cloudflare DNS and TLS certificates. We'll configure domain management, TLS certificates, and ingress rules for your cluster's services.

## 📋 Prerequisites

- A registered domain (e.g., `vaultlan.com`)
- A Cloudflare account
- Your cluster's public IP address
- Kubernetes cluster with:
  - NGINX Ingress Controller (already installed via Helmfile)
  - cert-manager (already installed via Helmfile)
  - Working LoadBalancer service (Cilium L2 announcements configured for IPs 192.168.20.130-230)

## 1. 🔗 Domain & DNS Configuration (Cloudflare)

### Add Domain to Cloudflare

1. Log in to your Cloudflare account
2. Click "Add Site" and enter `vaultlan.com`
3. Select the Free plan (or other plan based on your needs)
4. Update your domain's nameservers to use Cloudflare's nameservers

### Configure DNS Records

Add the following DNS records in Cloudflare:

```plaintext
Type  Name     Content          Proxy Status  TTL
A     @        <Your Public IP>    Proxied     Auto
A     *        <Your Public IP>    Proxied     Auto
```

### Enable Security Features

1. **SSL/TLS Settings**:
   - Go to SSL/TLS tab
   - Set encryption mode to "Full (strict)"
   - Enable "Always Use HTTPS"

2. **Enable DNSSEC**:
   - Go to DNS tab
   - Click "Enable DNSSEC"
   - Copy the DS record to your domain registrar

## 2. 🔒 TLS & Certificates with cert-manager

### Create Cloudflare API Token

1. Go to Cloudflare dashboard → API Tokens
2. Create new token with these permissions:
   - Zone - DNS - Edit
   - Zone - Zone - Read
3. Select your domain as the zone resource

### Create Kubernetes Secret for Cloudflare API

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token
  namespace: cert-manager
type: Opaque
stringData:
  api-token: "your-cloudflare-api-token"
```

### Configure ClusterIssuer for Cloudflare DNS01 Challenge

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: your-email@vaultlan.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - dns01:
        cloudflare:
          apiTokenSecretRef:
            name: cloudflare-api-token
            key: api-token
```

### Example Certificate Request

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-vaultlan-com
  namespace: cert-manager
spec:
  secretName: wildcard-vaultlan-com-tls
  dnsNames:
  - vaultlan.com
  - "*.vaultlan.com"
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  duration: 2160h # 90 days
  renewBefore: 360h # 15 days
```

## 3. 🚪 Ingress Configuration

### Example Ingress with TLS and Cloudflare Annotations

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: monitoring
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - grafana.vaultlan.com
    secretName: grafana-tls
  rules:
  - host: grafana.vaultlan.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kube-prometheus-stack-grafana
            port:
              number: 80
```

## 4. 🚇 Cloudflare Tunnel Setup (Optional)

### Install cloudflared

```cmd
REM Download cloudflared for Windows
curl -L --output cloudflared.exe https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-windows-amd64.exe

REM Log in to Cloudflare
cloudflared.exe tunnel login

REM Create a tunnel
cloudflared.exe tunnel create vaultlan-cluster

REM Configure the tunnel
cloudflared.exe tunnel route dns vaultlan-cluster *.vaultlan.com
```

### Create Kubernetes ConfigMap for cloudflared config

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudflared-config
  namespace: cloudflared
data:
  config.yaml: |
    tunnel: your-tunnel-id
    credentials-file: /etc/cloudflared/creds/credentials.json
    ingress:
      - hostname: "*.vaultlan.com"
        service: http://ingress-nginx-controller.ingress-nginx:80
      - service: http_status:404
```

### Deploy cloudflared to Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
  namespace: cloudflared
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cloudflared
  template:
    metadata:
      labels:
        app: cloudflared
    spec:
      containers:
      - name: cloudflared
        image: cloudflare/cloudflared:latest
        args:
        - tunnel
        - --config
        - /etc/cloudflared/config/config.yaml
        - run
        volumeMounts:
        - name: config
          mountPath: /etc/cloudflared/config
          readOnly: true
        - name: creds
          mountPath: /etc/cloudflared/creds
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: cloudflared-config
      - name: creds
        secret:
          secretName: cloudflared-credentials
```

## 5. 🔧 Networking Tips

### LoadBalancer IP Management

Your cluster is configured to use Cilium for LoadBalancer services with IPs in the range 192.168.20.130-230. When creating LoadBalancer services, they will automatically receive IPs from this pool.

### External-DNS Integration (Optional)

To automatically manage DNS records based on your Ingress resources:

1. Create Cloudflare API token with Zone DNS permissions
2. Install external-dns:

```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: external-dns
  namespace: kube-system
spec:
  chart: external-dns
  repo: https://kubernetes-sigs.github.io/external-dns
  targetNamespace: kube-system
  set:
    - name: provider
      value: cloudflare
    - name: cloudflare.apiToken
      valueFrom:
        secretKeyRef:
          name: cloudflare-api-token
          key: api-token
    - name: cloudflare.proxied
      value: "true"
    - name: policy
      value: sync
    - name: interval
      value: 1m
```

### Best Practices

1. **Security**:
   - Always use TLS for all ingress endpoints
   - Enable Cloudflare's security features (WAF, Rate Limiting)
   - Use Cloudflare's "Full (strict)" SSL mode

2. **Performance**:
   - Enable Cloudflare caching for static content
   - Use Cloudflare's Auto Minify feature
   - Configure appropriate browser cache TTLs

3. **Monitoring**:
   - Set up Cloudflare notifications for security events
   - Monitor certificate expiration dates
   - Use Grafana dashboards to monitor ingress controller metrics

## 📚 Additional Resources

- [Cloudflare DNS Documentation](https://developers.cloudflare.com/dns/)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [NGINX Ingress Controller Documentation](https://kubernetes.github.io/ingress-nginx/)
- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)

## 🔍 Troubleshooting

1. **Certificate Issues**:
   ```cmd
   REM Check certificate status
   kubectl get certificate -A
   kubectl get certificaterequest -A
   kubectl get order.acme.cert-manager.io -A
   ```

2. **DNS Issues**:
   ```cmd
   REM Test DNS resolution
   nslookup grafana.vaultlan.com
   nslookup -type=TXT _acme-challenge.grafana.vaultlan.com
   ```

3. **Ingress Issues**:
   ```cmd
   REM Check ingress status
   kubectl get ingress -A
   kubectl describe ingress -n monitoring grafana
   ```

4. **SSL Verification**:
   ```cmd
   REM Test SSL configuration
   curl -vI https://grafana.vaultlan.com
   ```
