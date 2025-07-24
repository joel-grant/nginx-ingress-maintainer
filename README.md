# nginx-ingress-maintainer

Welcome! This is a demo project showcasing a Helm chart that automates the deployment of NGINX Ingress Controller with automatic TLS certificate management for Kubernetes applications. 

While this started as a demo, I actively use this chart to manage ingress and TLS for my own deployed projects. If you'd like to try it out for your own applications, keep reading - there are detailed instructions below!

## Overview

This chart simplifies the process of exposing services in Kubernetes by providing:
- **NGINX Ingress Controller** - Load balancing and HTTP/HTTPS routing
- **Automatic TLS certificates** - Let's Encrypt integration via cert-manager
- **DNS-based validation** - Using DNS01 challenge with DigitalOcean
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