# Decrypt for inspection

SOPS_AGE_KEY_FILE=$HOME/src/personal/security/age-keys/age-key.txt \
sops -d personal/POC/argocd-agent-poc/test-customer/secrets/staging/secrets.enc.yaml

# Edit/reencrypt interactively

SOPS_AGE_KEY_FILE=$HOME/src/personal/security/age-keys/age-key.txt \
sops personal/POC/argocd-agent-poc/test-customer/secrets/staging/secrets.enc.yaml

# Set a specific key without opening an editor

SOPS_AGE_KEY_FILE=$HOME/src/personal/security/age-keys/age-key.txt \
sops --set '["secrets","KEY_NAME"] "new-value"' personal/POC/argocd-agent-poc/test-customer/secrets/staging/secrets.enc.yaml
