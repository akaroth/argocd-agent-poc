# ArgoCD Agent mTLS Authentication Setup Guide

## Overview

This document describes the complete configuration for switching ArgoCD agent authentication from **userpass** to **mTLS (mutual TLS)**, where both the principal and agent verify each other's certificates using X.509 certificates.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Principal Configuration](#principal-configuration)
3. [Agent Configuration](#agent-configuration)
4. [Ingress Configuration](#ingress-configuration)
5. [Certificate Management](#certificate-management)
6. [Authentication Flow](#authentication-flow)
7. [Setup Commands](#setup-commands)
8. [Verification](#verification)
9. [Troubleshooting](#troubleshooting)

---

## Architecture Overview

```
┌─────────────────┐           ┌─────────────────┐           ┌─────────────────┐
│  Agent Cluster  │           │  Nginx Ingress  │           │ Principal (AKS) │
│   (minikube)    │           │  (SSL Passthru) │           │  fsaiostest2    │
│                 │           │                 │           │                 │
│  fsai-poc       │◄─────────►│  argocd.        │◄─────────►│  argocd-agent-  │
│  agent          │  TLS+Cert │  tikalabs.dev   │  No Termin│  principal      │
│                 │           │  :443           │           │  :8443          │
└─────────────────┘           └─────────────────┘           └─────────────────┘
```

**Key Components:**

- **Principal**: Central ArgoCD control plane running on Azure AKS
- **Agent**: Remote ArgoCD agent running on minikube
- **Ingress**: nginx-ingress with SSL passthrough enabled
- **Authentication**: mTLS using X.509 certificates

---

## Principal Configuration

### Cluster: `fsaiostest2-admin` (Azure AKS)

### 1. ConfigMap: `argocd-agent-params`

**Namespace:** `argocd`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-agent-params
  namespace: argocd
data:
  # Authentication configuration
  principal.auth: "mtls:CN=([^,]+)"
  # ↑ Extracts agent name from client certificate's Common Name (CN)
  # Regex pattern matches any CN and uses it as the agent identifier

  principal.tls.client-cert.require: "true"
  # ↑ REQUIRES client certificates for authentication
  # Connections without valid client certs are rejected

  principal.tls.server.root-ca-secret-name: "argocd-agent-ca"
  # ↑ CA certificate used to verify client certificates

  principal.allowed-namespaces: "*"
  # ↑ Allow principal to watch all namespaces for application propagation
```

**Key Changes:**

- `principal.auth`: Changed from `userpass:/app/config/userpass/passwd` to `mtls:CN=([^,]+)`
- `principal.tls.client-cert.require`: Set to `"true"` (was `"false"`)

### 2. Secrets

#### `argocd-agent-ca`

- **Type:** `Opaque`
- **Purpose:** CA certificate that signed all agent client certificates
- **Usage:** Principal uses this to verify incoming client certificates

#### `argocd-agent-principal-tls`

- **Type:** `kubernetes.io/tls`
- **Purpose:** Principal's server certificate for TLS
- **Signed by:** Same CA as agent certificates

---

## Agent Configuration

### Cluster: `minikube`

### 1. ConfigMap: `argocd-agent-params`

**Namespace:** `argocd`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-agent-params
  namespace: argocd
data:
  # Authentication configuration
  agent.creds: "mtls:any"
  # ↑ Uses mTLS authentication instead of userpass
  # "any" means accept any certificate signed by the CA

  # Connection details
  agent.server.address: "argocd.tikalabs.dev"
  agent.server.port: "443"
  # ↑ Connects to principal via Ingress with SSL passthrough

  # TLS configuration
  agent.tls.client.cert-path: "/app/config/tls/client/tls.crt"
  agent.tls.client.key-path: "/app/config/tls/client/tls.key"
  agent.tls.root-ca-path: "/app/config/tls/ca/ca.crt"
  # ↑ Paths to mounted certificate files

  agent.tls.client.insecure: "false"
  # ↑ Verify principal's server certificate
```

**Key Changes:**

- `agent.creds`: Changed from `userpass:/app/config/creds/userpass.creds` to `mtls:any`
- `agent.tls.*`: Added paths to client certificate, key, and CA certificate

### 2. Deployment: `argocd-agent-agent`

**Volume Mounts:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-agent-agent
  namespace: argocd
spec:
  template:
    spec:
      volumes:
        - name: argocd-agent-client-tls
          secret:
            secretName: argocd-agent-client-tls
            defaultMode: 420 # 0644 permissions
        - name: argocd-agent-ca
          secret:
            secretName: argocd-agent-ca
            items:
              - key: tls.crt
                path: ca.crt
            defaultMode: 420

      containers:
        - name: agent
          volumeMounts:
            - name: argocd-agent-client-tls
              mountPath: /app/config/tls/client
              readOnly: true
            - name: argocd-agent-ca
              mountPath: /app/config/tls/ca
              readOnly: true
```

**Note:** `defaultMode: 420` (octal 0644) ensures proper file permissions for certificate files.

### 3. Secrets

#### `argocd-agent-client-tls`

- **Type:** `kubernetes.io/tls`
- **Generated by:** `argocd-agentctl pki issue`
- **Contains:**
  - `tls.crt`: Client certificate with `CN=fsai-poc`
  - `tls.key`: Private key for client certificate

#### `argocd-agent-ca`

- **Type:** `kubernetes.io/tls`
- **Purpose:** Copy of CA certificate from principal
- **Usage:** Agent uses this to verify principal's server certificate

---

## Ingress Configuration

### Cluster: `fsaiostest2-admin` (Azure AKS)

### 1. Ingress Resource: `argocd-principal-grpc`

**Namespace:** `argocd`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-principal-grpc
  namespace: argocd
  annotations:
    # CRITICAL: Enable SSL passthrough
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    # ↑ Nginx passes TLS connection directly to principal
    # Client certificate reaches the principal without termination

    # Removed: backend-protocol: "GRPCS"
    # ↑ This was terminating TLS at nginx and re-encrypting
    # Would strip away the client certificate

    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
spec:
  ingressClassName: nginx
  rules:
    - host: argocd.tikalabs.dev
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-agent-principal
                port:
                  number: 8443 # Principal's gRPC port
  tls:
    - hosts:
        - argocd.tikalabs.dev
      secretName: argocd-agent-principal-tls
```

**Key Changes:**

- **Added:** `nginx.ingress.kubernetes.io/ssl-passthrough: "true"`
- **Removed:** `nginx.ingress.kubernetes.io/backend-protocol: "GRPCS"`
- **Backend port:** Changed from `443` to `8443` (gRPC port)

### 2. Ingress Controller: `ingress-nginx-controller`

**Namespace:** `ingress-nginx`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  template:
    spec:
      containers:
        - name: controller
          args:
            # ... other args ...
            - --enable-ssl-passthrough # ← ADDED
            # ↑ Global flag required for SSL passthrough to work
            # Without this, the Ingress annotation is ignored
```

**Command to add:**

```bash
kubectl patch deployment ingress-nginx-controller \
  -n ingress-nginx \
  --context fsaiostest2-admin \
  --type='json' \
  -p='[{
    "op": "add",
    "path": "/spec/template/spec/containers/0/args/-",
    "value": "--enable-ssl-passthrough"
  }]'
```

---

## Certificate Management

### Certificate Authority (CA)

The CA certificate is stored in the `argocd-agent-ca` secret on the principal cluster:

```bash
kubectl get secret argocd-agent-ca \
  -n argocd \
  --context fsaiostest2-admin \
  -o jsonpath='{.data.tls\.crt}' | base64 -d
```

### Agent Client Certificate

Generated using `argocd-agentctl`:

```bash
argocd-agentctl pki issue agent fsai-poc \
  --principal-context fsaiostest2-admin \
  --agent-context minikube \
  --agent-namespace argocd \
  --upsert
```

**Certificate Details:**

- **Subject CN:** `fsai-poc` (agent name)
- **Issuer:** ArgoCD Agent CA
- **Key Usage:** Digital Signature, Key Encipherment
- **Extended Key Usage:** TLS Web Client Authentication

**Verify certificate:**

```bash
kubectl get secret argocd-agent-client-tls \
  -n argocd \
  --context minikube \
  -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | \
  openssl x509 -noout -text | \
  grep -E "Subject:|Issuer:|CN="
```

---

## Authentication Flow

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         mTLS Authentication Flow                             │
└──────────────────────────────────────────────────────────────────────────────┘

Step 1: Agent → Ingress (argocd.tikalabs.dev:443)
┌─────────────┐
│    Agent    │  Initiates TLS handshake
│  (minikube) │  Presents client certificate (CN=fsai-poc)
└──────┬──────┘
       │  TLS ClientHello + Client Certificate
       ▼
┌─────────────┐
│   Ingress   │  SSL Passthrough enabled
│   (nginx)   │  Does NOT terminate TLS
└──────┬──────┘  Forwards packets as-is
       │
       │  Raw TLS packets (certificate intact)
       ▼
┌─────────────┐
│  Principal  │  Receives TLS handshake
│    (AKS)    │
└─────────────┘

Step 2: Principal Verifies Client Certificate
┌─────────────┐
│  Principal  │  1. Checks certificate signature using argocd-agent-ca
│    (AKS)    │  2. Extracts CN from certificate (e.g., "fsai-poc")
└──────┬──────┘  3. Uses CN as agent identifier
       │
       │  Verification Result
       ▼
   ┌────────┐
   │ Accept │ if signed by trusted CA
   └────────┘

Step 3: Agent Verifies Principal Certificate
┌─────────────┐
│    Agent    │  Verifies principal's server certificate
│  (minikube) │  Uses CA from /app/config/tls/ca/ca.crt
└──────┬──────┘  Ensures talking to real principal
       │
       │  Verification Result
       ▼
   ┌────────┐
   │ Accept │ if signed by trusted CA
   └────────┘

Step 4: Secure Bidirectional Channel Established
┌─────────────┐                              ┌─────────────┐
│    Agent    │ ◄────────────────────────────► │  Principal  │
│  (minikube) │  Encrypted gRPC communication │    (AKS)    │
└─────────────┘  Both sides authenticated     └─────────────┘
```

---

## Setup Commands

### Prerequisites

1. **Install argocd-agentctl:**

   ```bash
   # Download from GitHub releases
   curl -sL https://github.com/argoproj-labs/argocd-agent/releases/download/v0.5.0/argocd-agentctl-darwin-arm64 \
     -o /usr/local/bin/argocd-agentctl
   chmod +x /usr/local/bin/argocd-agentctl
   ```

2. **Ensure kubectl contexts are configured:**
   - `fsaiostest2-admin` → Principal cluster (Azure AKS)
   - `minikube` → Agent cluster

### Step-by-Step Setup

#### 1. Configure Principal for mTLS

```bash
# Update principal ConfigMap
kubectl patch configmap argocd-agent-params \
  -n argocd \
  --context fsaiostest2-admin \
  --patch '{
    "data": {
      "principal.auth": "mtls:CN=([^,]+)",
      "principal.tls.client-cert.require": "true"
    }
  }'

# Restart principal
kubectl rollout restart deployment argocd-agent-principal \
  -n argocd \
  --context fsaiostest2-admin
```

#### 2. Enable SSL Passthrough on Ingress

```bash
# Update Ingress resource
kubectl patch ingress argocd-principal-grpc \
  -n argocd \
  --context fsaiostest2-admin \
  --type=json \
  -p='[
    {
      "op": "add",
      "path": "/metadata/annotations/nginx.ingress.kubernetes.io~1ssl-passthrough",
      "value": "true"
    },
    {
      "op": "remove",
      "path": "/metadata/annotations/nginx.ingress.kubernetes.io~1backend-protocol"
    }
  ]'

# Enable SSL passthrough in ingress controller
kubectl patch deployment ingress-nginx-controller \
  -n ingress-nginx \
  --context fsaiostest2-admin \
  --type='json' \
  -p='[{
    "op": "add",
    "path": "/spec/template/spec/containers/0/args/-",
    "value": "--enable-ssl-passthrough"
  }]'

# Wait for rollout
kubectl rollout status deployment ingress-nginx-controller \
  -n ingress-nginx \
  --timeout=180s \
  --context fsaiostest2-admin
```

#### 3. Configure Agent with argocd-agentctl

```bash
# Reconfigure agent on principal
argocd-agentctl agent reconfigure fsai-poc \
  --principal-context fsaiostest2-admin \
  --principal-namespace argocd \
  --resource-proxy-server 4.254.62.81:9090 \
  --resource-proxy-username fsai-poc \
  --resource-proxy-password "fsai-123"

# Issue client certificates
argocd-agentctl pki issue agent fsai-poc \
  --principal-context fsaiostest2-admin \
  --agent-context minikube \
  --agent-namespace argocd \
  --upsert
```

#### 4. Update Agent ConfigMap

```bash
# Update agent ConfigMap
kubectl patch configmap argocd-agent-params \
  -n argocd \
  --context minikube \
  --patch '{
    "data": {
      "agent.creds": "mtls:any",
      "agent.server.address": "argocd.tikalabs.dev",
      "agent.server.port": "443",
      "agent.tls.client.cert-path": "/app/config/tls/client/tls.crt",
      "agent.tls.client.key-path": "/app/config/tls/client/tls.key",
      "agent.tls.root-ca-path": "/app/config/tls/ca/ca.crt"
    }
  }'

# Restart agent
kubectl rollout restart deployment argocd-agent-agent \
  -n argocd \
  --context minikube
```

---

## Verification

### 1. Check Agent Connection

```bash
# Agent logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-agent-agent \
  --tail=50 \
  --context minikube | \
  grep -E "Starting event writer|Updated application|Connected"
```

**Expected output:**

```
time="..." level=info msg="Starting event writer" clientAddr="4.254.62.81:443"
time="..." level=info msg="Updated application" application=argocd/test-nginx
```

### 2. Check Principal Receiving Events

```bash
# Principal logs
kubectl logs -n argocd deployment/argocd-agent-principal \
  --tail=50 \
  --context fsaiostest2-admin | \
  grep "client=fsai-poc"
```

**Expected output:**

```
time="..." level=info msg="Processing clusterCacheInfoUpdate event" client=fsai-poc
time="..." level=info msg="Updated cluster cache stats in cluster." agent=fsai-poc
```

### 3. Verify SSL Passthrough Configuration

```bash
# Check Ingress annotation
kubectl get ingress argocd-principal-grpc \
  -n argocd \
  --context fsaiostest2-admin \
  -o jsonpath='{.metadata.annotations.nginx\.ingress\.kubernetes\.io/ssl-passthrough}'
# Expected: true

# Check ingress controller flag
kubectl get deployment ingress-nginx-controller \
  -n ingress-nginx \
  --context fsaiostest2-admin \
  -o jsonpath='{.spec.template.spec.containers[0].args}' | \
  jq -r '.[]' | \
  grep "ssl-passthrough"
# Expected: --enable-ssl-passthrough
```

### 4. Verify Cluster Secret

```bash
# Check cluster secret generated by agent
kubectl get secret cluster-fsai-poc \
  -n argocd \
  --context fsaiostest2-admin \
  -o jsonpath='{.data.config}' | \
  base64 -d | \
  jq '.tlsClientConfig'
```

**Expected structure:**

```json
{
  "insecure": false,
  "certData": "<base64-encoded-client-cert>",
  "keyData": "<base64-encoded-client-key>",
  "caData": "<base64-encoded-ca-cert>"
}
```

### 5. Test Application Sync

```bash
# Create a test application
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: test-app
  namespace: fsai-poc
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps
    path: guestbook
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: default
EOF

# Check sync status
kubectl get application test-app -n fsai-poc \
  --context fsaiostest2-admin \
  -o jsonpath='{.status.sync.status}'
# Expected: Synced
```

---

## Troubleshooting

### Issue: 502 Bad Gateway

**Symptom:**

```
Auth failure: rpc error: code = Unavailable desc = unexpected HTTP status code received from server: 502 (Bad Gateway)
```

**Cause:** SSL passthrough not enabled

**Solution:**

1. Verify Ingress annotation:
   ```bash
   kubectl get ingress argocd-principal-grpc -n argocd --context fsaiostest2-admin \
     -o jsonpath='{.metadata.annotations}' | jq '."nginx.ingress.kubernetes.io/ssl-passthrough"'
   ```
2. Verify ingress controller flag:
   ```bash
   kubectl get deployment ingress-nginx-controller -n ingress-nginx --context fsaiostest2-admin \
     -o jsonpath='{.spec.template.spec.containers[0].args}' | jq -r '.[]' | grep ssl-passthrough
   ```

### Issue: TLS Handshake Timeout

**Symptom:**

```
Auth failure: rpc error: code = Unavailable desc = connection timeout
```

**Cause:** Agent cannot reach principal

**Solution:**

1. Check agent server address:
   ```bash
   kubectl get configmap argocd-agent-params -n argocd --context minikube \
     -o jsonpath='{.data.agent\.server\.address}'
   ```
2. Test DNS resolution:
   ```bash
   kubectl exec -n argocd deployment/argocd-agent-agent --context minikube -- \
     nslookup argocd.tikalabs.dev
   ```
3. Test connectivity:
   ```bash
   kubectl exec -n argocd deployment/argocd-agent-agent --context minikube -- \
     timeout 5 openssl s_client -connect argocd.tikalabs.dev:443
   ```

### Issue: Certificate Verification Failed

**Symptom:**

```
Auth failure: rpc error: code = Unavailable desc = x509: certificate signed by unknown authority
```

**Cause:** Agent's CA certificate doesn't match principal's server certificate

**Solution:**

1. Verify agent has correct CA certificate:
   ```bash
   kubectl get secret argocd-agent-ca -n argocd --context minikube \
     -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text
   ```
2. Compare with principal's CA:
   ```bash
   kubectl get secret argocd-agent-ca -n argocd --context fsaiostest2-admin \
     -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text
   ```
3. Re-copy CA if needed:
   ```bash
   kubectl get secret argocd-agent-ca -n argocd --context fsaiostest2-admin -o yaml | \
     kubectl apply -f - --context minikube
   ```

### Issue: Permission Denied Reading Certificate Files

**Symptom:**

```
FATAL: Error creating remote: open /app/config/tls/ca/ca.crt: permission denied
```

**Cause:** Incorrect file permissions on mounted secrets

**Solution:**

1. Check deployment volume mount configuration:
   ```bash
   kubectl get deployment argocd-agent-agent -n argocd --context minikube \
     -o jsonpath='{.spec.template.spec.volumes}' | jq '.[] | select(.name | contains("tls"))'
   ```
2. Ensure `defaultMode: 420` (0644) is set:
   ```bash
   kubectl patch deployment argocd-agent-agent -n argocd --context minikube --type='json' -p='[
     {
       "op": "add",
       "path": "/spec/template/spec/volumes/0/secret/defaultMode",
       "value": 420
     }
   ]'
   ```

---

## Comparison: Userpass vs mTLS

| Aspect                     | Userpass                     | mTLS                       |
| -------------------------- | ---------------------------- | -------------------------- |
| **Authentication Method**  | Username + password          | X.509 certificates         |
| **Security Level**         | Medium                       | High                       |
| **Secret Management**      | Store bcrypt hash            | Store private key          |
| **Rotation**               | Update password hash         | Reissue certificates       |
| **Ingress Requirement**    | Can terminate TLS            | MUST passthrough TLS       |
| **Certificate Validation** | N/A                          | Cryptographic signature    |
| **Compromise Impact**      | Password can be brute-forced | Private key must be stolen |
| **Best For**               | Development/testing          | Production environments    |

---

## Additional Resources

- [ArgoCD Agent Documentation](https://argocd-agent.readthedocs.io/)
- [nginx-ingress SSL Passthrough](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#ssl-passthrough)
- [X.509 Certificates](https://en.wikipedia.org/wiki/X.509)
- [mTLS Explained](https://www.cloudflare.com/learning/access-management/what-is-mutual-tls/)

---

## Summary

✅ **Principal configured** for mTLS authentication
✅ **Ingress configured** with SSL passthrough enabled
✅ **Agent configured** with client certificates
✅ **Certificates managed** via argocd-agentctl
✅ **Authentication working** via `argocd.tikalabs.dev:443`

**Status: OPERATIONAL** 🚀

---

**Document Version:** 1.0
**Last Updated:** November 3, 2025
**Agent Name:** `fsai-poc`
**Principal Cluster:** `fsaiostest2-admin` (Azure AKS)
**Agent Cluster:** `minikube`
