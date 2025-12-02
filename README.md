# nginx-ingress-maintainer

This is a demo/practice project showcasing a Helm chart that automates the deployment of NGINX Ingress Controller with automatic TLS certificate management, rate limiting, and flexible host configuration for Kubernetes applications. 

While this started as a demo/practice project, I have continued to use it for projects as I grow in the field of infrastructure automation and software engineering.

With that said... **Use at Your Own Risk** :D

## Here's what I've set up so far...

This chart simplifies the process of exposing services in Kubernetes by providing:
- **NGINX Ingress Controller** - Load balancing and HTTP/HTTPS routing
- **Automatic TLS certificates** - Let's Encrypt integration via cert-manager
- **DNS-based validation** - Using DNS01 challenge with DigitalOcean
- **Rate limiting** - Configurable per-ingress traffic controls with custom responses
- **Multiple host support** - Flexible subdomain and multi-service routing
- **High availability** - Configurable replica count and monitoring

## Prerequisites

- **Kubernetes cluster** (v1.19+)
- **Helm** (v3.0+)
- **cert-manager** installed in the cluster
- **DigitalOcean DNS** (for DNS01 challenge)
- **DigitalOcean API token** with DNS write permissions

## Installation

### Install this chart

```bash
helm repo add nginx-ingress-maintainer https://your-repo-url
helm install my-ingress nginx-ingress-maintainer/nginx-ingress-maintainer \
  --namespace ingress-system \
  --create-namespace \ # Remove this if the namespace already exists
  --values values.yaml
```

## Configuration

### Required Values

Create a `values.yaml` file with the following required configuration:

```yaml
doAccessToken: "your-digitalocean-api-token"

ingress:
  name: "my-app"
  namespace: "default"
  domain:
    name: "example.com"
    targetService: "my-app-service"

issuer:
  name: "letsencrypt-issuer"
  namespace: "default"
  email: "your-email@example.com"

ingress-nginx:
  controller:
    ingressClassResource:
      name: "my-app-nginx"
```

### Subdomain Support

This chart supports multiple ways to configure subdomains:

#### Option 1: Simple Subdomains (Same Service)
Point multiple subdomains to the same service as your main domain:

```yaml
ingress:
  domain:
    name: "example.com"
    targetService: "my-app-service"
    subdomains:
      - "api"      # api.example.com -> my-app-service
      - "admin"    # admin.example.com -> my-app-service
      - "blog"     # blog.example.com -> my-app-service
```

#### Option 2: Advanced Subdomains (Different Services)
Configure subdomains to point to different services with custom ports and paths:

```yaml
ingress:
  domain:
    name: "example.com"
    targetService: "main-service"
  hosts:
    - host: "api.example.com"
      service: "api-service"
      port: 8080
      paths:
        - path: "/"
          pathType: "Prefix"
    - host: "admin.example.com"
      service: "admin-dashboard"
      port: 3000
      paths:
        - path: "/"
          pathType: "Prefix"
        - path: "/api"
          pathType: "Prefix"
```

**Note**: All configured domains and subdomains will automatically be included in the TLS certificate.

### Rate Limiting

Protect your services from abuse with configurable rate limiting:

```yaml
ingress:
  rateLimit:
    enabled: true
    rps: 20                    # 20 requests per second per IP
    connections: 10            # Max 10 concurrent connections per IP
    burstMultiplier: 5         # Allow bursts up to rps * 5
    whitelist: "10.0.0.0/8"    # Exclude internal IPs

# Global response customization (affects all ingresses)
ingress-nginx:
  controller:
    config:
      limit-req-status-code: "429"  # HTTP 429 instead of 503
      limit-conn-status-code: "429"
```

**Features:**
- Per-IP rate limiting (requests per second or per minute)
- Connection limits with burst handling
- IP whitelisting for trusted sources
- Customizable HTTP response codes (429, 503, etc.)
- Applied per ingress with global response settings

### Optional Configuration

```yaml
ingress:
  whitelist:
    enabled: true
    whitelistSourceRange: "192.168.1.0/24,10.0.0.0/16"

ingress-nginx:
  controller:
    replicaCount: 2  # For high availability
    metrics:
      enabled: true  # Enable Prometheus metrics
```

## Using as a Dependency

To use this chart as a dependency in another Helm chart:

### 1. Add to your Chart.yaml

```yaml
dependencies:
- name: nginx-ingress-maintainer
  version: "1.5.0"
  repository: "https://your-repo-url"
```

### 2. Configure in your values.yaml

```yaml
nginx-ingress-maintainer:
  doAccessToken: "your-digitalocean-api-token"
  
  ingress:
    name: "my-app"
    namespace: "default"
    domain:
      name: "my-app.example.com"
      targetService: "my-app-service"
  
  issuer:
    name: "my-app-issuer"
    namespace: "default"
    email: "admin@example.com"
  
  ingress-nginx:
    controller:
      ingressClassResource:
        name: "my-app-nginx"
```

### 3. Update dependencies

```bash
helm dependency update
```

## How It Works

1. **NGINX Ingress Controller** is deployed using the official ingress-nginx Helm chart
2. **cert-manager Issuer** is created to handle Let's Encrypt certificate requests
3. **Ingress resource** is configured with TLS and routes traffic to your service
4. **DNS01 challenge** automatically validates domain ownership using DigitalOcean DNS
5. **TLS certificates** are automatically issued, renewed, and applied

## Architecture

```
Internet → DigitalOcean LoadBalancer → NGINX Ingress → Your Service
                                    ↓
                              cert-manager → Let's Encrypt
```

## Troubleshooting

### Certificate Issues
```bash
kubectl describe issuer <issuer-name> -n <namespace>
kubectl describe certificate <certificate-name> -n <namespace>
kubectl logs -n cert-manager deployment/cert-manager
```

### Ingress Issues
```bash
kubectl describe ingress <ingress-name> -n <namespace>
kubectl logs -n ingress-system deployment/ingress-nginx-controller
```

## References

- [NGINX Ingress Controller Documentation](https://kubernetes.github.io/ingress-nginx/)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [Multiple Ingress Controllers](https://kubernetes.github.io/ingress-nginx/user-guide/multiple-ingress/)
- [Let's Encrypt DNS Validation](https://letsencrypt.org/docs/challenge-types/#dns-01-challenge)