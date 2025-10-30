# test-customer Multi-Application Setup

This directory contains a multi-application ArgoCD setup for testing the principal-agent architecture with SOPS-encrypted secrets.

## Structure

```
test-customer/
├── metadata.yaml                      # Customer metadata and repository config
├── helmfile.yaml.gotmpl              # Creates ArgoCD applications
├── argocd-apps.yaml.gotmpl           # Defines 4 applications
├── fsai-os/                          # FSAI OS application
│   ├── helmfile.yaml.gotmpl
│   ├── values/common.yaml
│   └── versions.yaml
├── supabase/                         # Supabase application
│   ├── helmfile.yaml.gotmpl
│   ├── values/common.yaml
│   ├── versions.yaml
│   └── components/
│       ├── Chart.yaml
│       └── templates/usecase-db.yaml
├── minio-tenant/                     # MinIO tenant application
│   ├── helmfile.yaml.gotmpl
│   └── versions.yaml
└── secrets/                          # Encrypted secrets
    ├── helmfile.yaml.gotmpl
    └── staging/
        ├── Chart.yaml
        └── templates/fsai-os.enc.yaml (SOPS encrypted)
```

## Applications Created

When deployed, this creates 4 ArgoCD applications in the principal cluster:

1. **test-customer-secrets** → `helmfile/fsai-os/test-customer/secrets`
2. **test-customer-fsai-os** → `helmfile/fsai-os/test-customer/fsai-os`
3. **test-customer-minio-tenant** → `helmfile/fsai-os/test-customer/minio-tenant`
4. **test-customer-supabase** → `helmfile/fsai-os/test-customer/supabase`

## Prerequisites

### 1. SOPS Operator Deployment

The SOPS operator must be deployed to the **agent cluster** (where applications will run):

```bash
# Create the age key secret in the cluster
kubectl create namespace sre-security
kubectl create secret generic sops-age \
  --from-file=keys.txt=/tmp/age-encrypt-keys/age-key.txt \
  -n sre-security
```

### 2. Deploy SOPS Operator

Deploy the SOPS operator using the existing helmfile configuration at `helmfile/sre-security/sops-operator/`.

**Option A: Via helmfile directly**

```bash
cd helmfile/sre-security/sops-operator
helmfile -e staging apply
```

**Option B: Via ArgoCD**
Create an ArgoCD application pointing to `helmfile/sre-security/sops-operator`.

### 3. Verify SOPS Operator

```bash
kubectl get pods -n sre-security
# Should see: sops-secrets-operator-xxxxx running
```

## Deployment

### Method 1: Create ArgoCD Application (Recommended)

Create an ArgoCD Application in the principal cluster pointing to this directory:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: test-customer
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Future-Secure-AI/infrastructure-saas-customer-platform
    targetRevision: main
    path: helmfile/fsai-os/test-customer
    plugin:
      name: helmfile
  destination:
    server: https://kubernetes.default.svc # or your agent cluster URL
    namespace: argocd
  syncPolicy:
    automated: {}
```

This will create the 4 child applications automatically.

### Method 2: Deploy via helmfile (Testing)

For local testing:

```bash
cd helmfile/fsai-os/test-customer
export ENVIRONMENT=staging
export CLUSTER=aks-staging  # or your cluster name
export BASE_DOMAIN=fsai.cloud

helmfile template
# Review the output

helmfile apply
```

## How It Works

### Principal Cluster (ArgoCD)

1. ArgoCD detects the test-customer application
2. Helmfile plugin renders `argocd-apps.yaml.gotmpl`
3. Creates 4 Application resources in argocd namespace
4. Each application points to a subdirectory path

### Agent Cluster (Application Runtime)

For each application, ArgoCD:

1. Clones the repo
2. Renders helmfile templates
3. Applies manifests to the agent cluster

For secrets specifically:

1. `SopsSecret` resource is created in test-customer namespace
2. SOPS operator detects it
3. Decrypts using age key from `sre-security/sops-age` secret
4. Creates standard Kubernetes `Secret` with decrypted values
5. Applications use the decrypted secrets

## Age Encryption Details

- **Public Key**: `age1a75gcewnqrplsckw23c5pvzsuy7xs5a9uuhqn6kshlr98yvq0sas8kclma`
- **Private Key**: Stored in `/tmp/age-encrypt-keys/age-key.txt` (backup securely!)
- **SOPS Config**: See `.sops.yaml` in repository root

## Modifying Secrets

To update encrypted secrets:

```bash
# Edit the encrypted file (SOPS will decrypt for editing)
sops helmfile/fsai-os/test-customer/secrets/staging/templates/fsai-os.enc.yaml

# Or encrypt a new file
sops -e -i helmfile/fsai-os/test-customer/secrets/staging/templates/fsai-os.enc.yaml
```

## Testing

1. Deploy SOPS operator
2. Deploy test-customer application
3. Verify all 4 applications are created:
   ```bash
   kubectl get applications -n argocd | grep test-customer
   ```
4. Verify secrets are decrypted:
   ```bash
   kubectl get secret fsai-os-global-secrets -n test-customer
   kubectl get secret fsai-os-global-secrets -n test-customer -o yaml
   ```

## Cleanup

```bash
kubectl delete application test-customer -n argocd
# This will delete all child applications and resources
```

## Troubleshooting

### Secrets not decrypting

- Check SOPS operator is running: `kubectl get pods -n sre-security`
- Check age secret exists: `kubectl get secret sops-age -n sre-security`
- Check SopsSecret status: `kubectl get sopssecret -n test-customer`
- Check operator logs: `kubectl logs -n sre-security deployment/sops-secrets-operator`

### Applications not syncing

- Check helmfile plugin is configured in ArgoCD
- Verify environment variables are set (ENVIRONMENT, CLUSTER, BASE_DOMAIN)
- Check ArgoCD application logs

## Notes

- This is a **test setup** with dummy secret values
- For production, generate real secrets and encrypt them
- Keep the age private key secure (backup in password manager/vault)
- The age key can be rotated by generating a new key and re-encrypting secrets
