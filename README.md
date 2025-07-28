# cert-manager with step-ca Integration

This Helm chart deploys cert-manager configured to work with the Goldentooth cluster's step-ca certificate authority.

## Overview

cert-manager automates certificate management in Kubernetes using ACME protocol. This configuration integrates with our private step-ca instance running on the `jast` node.

## Architecture

- **cert-manager**: Kubernetes-native certificate management
- **step-ca**: Private certificate authority (running on jast:9443)
- **ACME protocol**: Automated certificate issuance and renewal
- **ClusterIssuer**: Configured for step-ca ACME endpoint with DNS-01 validation
- **Route53**: AWS Route53 DNS-01 challenges for domain validation

## Prerequisites

- step-ca running with ACME provisioner enabled
- ArgoCD deployed in the cluster
- Kubernetes cluster with CustomResourceDefinition support

## Configuration

The chart includes:

1. **cert-manager Application**: ArgoCD application for cert-manager deployment
2. **step-ca Root Certificate**: Secret containing the step-ca root CA certificate
3. **ClusterIssuer**: ACME issuer pointing to step-ca ACME endpoint with DNS-01 validation
4. **AWS Credentials**: Secret containing Route53 credentials for DNS challenges

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

### 1. Create AWS Credentials Secret

Create the AWS credentials secret manually in the cert-manager namespace:

```bash
# Get existing credentials from external-dns if available
kubectl get secret external-dns -n external-dns -o jsonpath='{.data.credentials}' | base64 -d

# Create the secret manually (replace with your actual credentials)
kubectl create secret generic cert-manager-route53-credentials \
  --from-literal=secret-access-key=YOUR_AWS_SECRET_ACCESS_KEY \
  -n cert-manager
```

### 2. Deploy via ArgoCD

Deploy the chart with your AWS access key ID:

```bash
# Deploy with access key ID
helm template cert-manager . \
  --namespace argocd \
  --set spec.aws.accessKeyID="YOUR_ACCESS_KEY_ID" | \
  kubectl apply -f -
```

**Important**: The secret access key is provided via the manually created Secret, while the access key ID is provided via Helm values. This approach keeps secrets out of Git while allowing ArgoCD deployment.

## Monitoring

cert-manager provides metrics and can be monitored via Prometheus. Check certificate status:

```bash
kubectl get certificates -A
kubectl get certificaterequests -A
kubectl describe clusterissuer step-ca-acme-issuer
```

## DNS-01 Validation

This configuration uses DNS-01 challenges via AWS Route53, which provides several advantages:

- **No ingress controller required**: Works without nginx or other ingress controllers
- **Wildcard certificate support**: Can issue `*.goldentooth.net` certificates
- **Private services**: Can validate domains for services not exposed to the internet
- **Firewall friendly**: No need to expose challenge endpoints

DNS-01 challenges work by creating TXT records in Route53 that step-ca validates.

## Troubleshooting

1. **DNS-01 Challenge Failures**: 
   - Check AWS credentials have Route53 permissions
   - Verify DNS zone configuration in Route53
   - Ensure step-ca can resolve DNS queries
2. **Root CA Issues**: Verify the step-ca root certificate is correctly configured
3. **Network Connectivity**: Ensure cert-manager can reach step-ca at 10.4.0.19:9443
4. **AWS Permissions**: Ensure AWS credentials have `route53:ChangeResourceRecordSets` permission

## Security

- Root CA certificate is stored as a Kubernetes Secret
- ACME private keys are automatically managed by cert-manager
- All communication with step-ca uses TLS