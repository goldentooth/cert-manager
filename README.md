# cert-manager with step-ca Integration

This Helm chart deploys cert-manager configured to work with the Goldentooth cluster's step-ca certificate authority.

## Overview

cert-manager automates certificate management in Kubernetes using ACME protocol. This configuration integrates with our private step-ca instance running on the `jast` node.

## Architecture

- **cert-manager**: Kubernetes-native certificate management
- **step-ca**: Private certificate authority (running on jast:9443)
- **ACME protocol**: Automated certificate issuance and renewal
- **ClusterIssuer**: Configured for step-ca ACME endpoint

## Prerequisites

- step-ca running with ACME provisioner enabled
- ArgoCD deployed in the cluster
- Kubernetes cluster with CustomResourceDefinition support

## Configuration

The chart includes:

1. **cert-manager Application**: ArgoCD application for cert-manager deployment
2. **step-ca Root Certificate**: Secret containing the step-ca root CA certificate
3. **ClusterIssuer**: ACME issuer pointing to step-ca ACME endpoint

## Usage

### Request a Certificate

Create a Certificate resource:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-tls
  namespace: default
spec:
  secretName: example-tls-secret
  issuerRef:
    name: step-ca-acme-issuer
    kind: ClusterIssuer
  dnsNames:
  - example.goldentooth.net
```

### Configure Ingress with TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    cert-manager.io/cluster-issuer: step-ca-acme-issuer
spec:
  tls:
  - hosts:
    - example.goldentooth.net
    secretName: example-tls-secret
  rules:
  - host: example.goldentooth.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: example-service
            port:
              number: 80
```

## step-ca Configuration

The step-ca instance should have an ACME provisioner configured:

```bash
step ca provisioner add acme --type ACME
```

## Deployment

Deploy via ArgoCD by adding this chart to the gitops repository:

```bash
# Add to ArgoCD applications
helm template cert-manager . --namespace argocd | kubectl apply -f -
```

## Monitoring

cert-manager provides metrics and can be monitored via Prometheus. Check certificate status:

```bash
kubectl get certificates -A
kubectl get certificaterequests -A
kubectl describe clusterissuer step-ca-acme-issuer
```

## Troubleshooting

1. **ACME Challenge Failures**: Ensure DNS or HTTP01 challenges can reach your domains
2. **Root CA Issues**: Verify the step-ca root certificate is correctly configured
3. **Network Connectivity**: Ensure cert-manager can reach step-ca at 10.4.0.19:9443

## Security

- Root CA certificate is stored as a Kubernetes Secret
- ACME private keys are automatically managed by cert-manager
- All communication with step-ca uses TLS