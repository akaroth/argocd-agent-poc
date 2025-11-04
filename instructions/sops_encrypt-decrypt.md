# SOPS quick reference

All commands assume the age private key lives at
`$HOME/src/personal/security/age-keys/age-key.txt`. Adjust the path if you
store it somewhere else.

## Decrypt for read-only inspection

```bash
SOPS_AGE_KEY_FILE=$HOME/src/personal/security/age-keys/age-key.txt \
sops -d personal/POC/argocd-agent-poc/test-customer/secrets/staging/secrets.enc.yaml
```

## Open secret for editing (auto re‑encrypt on save)

```bash
SOPS_AGE_KEY_FILE=$HOME/src/personal/security/age-keys/age-key.txt \
sops personal/POC/argocd-agent-poc/test-customer/secrets/staging/secrets.enc.yaml
```

## Update a single key inline (no editor)

```bash
SOPS_AGE_KEY_FILE=$HOME/src/personal/security/age-keys/age-key.txt \
sops --set '["secrets","KEY_NAME"] "new-value"' \
  personal/POC/argocd-agent-poc/test-customer/secrets/staging/secrets.enc.yaml
```

Replace `KEY_NAME` and `new-value` with the field you need to modify. Repeat
the `--set …` flag to change multiple keys in a single call.

## Sanity checks

```bash
# verify decrypted content before committing
SOPS_AGE_KEY_FILE=$HOME/src/personal/security/age-keys/age-key.txt \
sops -d personal/POC/argocd-agent-poc/test-customer/secrets/staging/secrets.enc.yaml | less

# confirm git diff only shows encrypted blob changes
git status -sb
git diff personal/POC/argocd-agent-poc/test-customer/secrets/staging/secrets.enc.yaml
```

> Tip: keep the decrypted output strictly in memory (e.g., view with `less`)
> so that sensitive values never land in temporary files.
