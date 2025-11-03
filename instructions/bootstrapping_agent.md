Perfect! Here are the step-by-step commands to create the `fsai-poc` agent:

## Step 1: Generate Password and Update Principal

```bash
# Set agent details
NEW_AGENT_NAME="fsai-poc"
NEW_AGENT_PASSWORD="$(openssl rand -base64 32)"

# Save password for later use
echo "$NEW_AGENT_PASSWORD" > /tmp/fsai-poc-password.txt
echo "✓ Password saved to /tmp/fsai-poc-password.txt"
echo "Password: $NEW_AGENT_PASSWORD"

# Get current passwd file from principal
kubectl get secret argocd-agent-principal-userpass -n argocd \
  --context fsaiostest2-admin \
  -o jsonpath='{.data.passwd}' | base64 -d > /tmp/current-passwd.txt

echo "Current agents:"
cat /tmp/current-passwd.txt

# Generate bcrypt hash for new agent
NEW_HASH=$(htpasswd -nbBC 10 "" "$NEW_AGENT_PASSWORD" | tr -d ':\n' | sed 's/^//')

# Append new agent to passwd file
echo "${NEW_AGENT_NAME}:${NEW_HASH}" >> /tmp/current-passwd.txt

echo ""
echo "Updated agents:"
cat /tmp/current-passwd.txt

# Update the secret on principal
kubectl create secret generic argocd-agent-principal-userpass \
  -n argocd \
  --from-file=passwd=/tmp/current-passwd.txt \
  --context fsaiostest2-admin \
  --dry-run=client -o yaml | kubectl apply -f - --context fsaiostest2-admin

echo "✓ Principal secret updated"

# Restart principal to reload credentials
kubectl rollout restart deployment argocd-agent-principal -n argocd --context fsaiostest2-admin

echo "✓ Principal restarted"
```

## Step 2: Create Agent Namespace on Principal

```bash
# Create namespace for fsai-poc on principal
kubectl create namespace fsai-poc --context fsaiostest2-admin

echo "✓ Namespace fsai-poc created on principal"
```

## Step 3: Install ArgoCD Components on Minikube

```bash
# Create argocd namespace
kubectl create namespace argocd --context minikube

# Install agent-managed ArgoCD components
kubectl apply -n argocd \
  -k 'https://github.com/argoproj-labs/argocd-agent/install/kubernetes/argo-cd/agent-managed?ref=v0.5.0' \
  --context minikube

# Add server.secretkey
kubectl patch secret argocd-secret -n argocd --context minikube \
  --patch="{\"data\":{\"server.secretkey\":\"$(openssl rand -base64 32 | base64 | tr -d '\n')\"}}"

echo "✓ ArgoCD components installed"
```

## Step 4: Deploy Agent

```bash
# Deploy agent
kubectl apply -n argocd \
  -k 'https://github.com/argoproj-labs/argocd-agent/install/kubernetes/agent?ref=v0.5.0' \
  --context minikube

echo "✓ Agent deployed"
```

## Step 5: Configure Certificates

```bash
# Copy CA certificate from principal
kubectl get secret argocd-agent-ca -n argocd --context fsaiostest2-admin \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > /tmp/ca.crt

kubectl create secret generic argocd-agent-ca \
  --from-file=tls.crt=/tmp/ca.crt \
  --from-file=ca.crt=/tmp/ca.crt \
  -n argocd \
  --context minikube

rm /tmp/ca.crt

# Create dummy client TLS (for userpass mode)
openssl req -x509 -newkey rsa:2048 -keyout /tmp/tls.key -out /tmp/tls.crt \
  -days 365 -nodes -subj "/CN=dummy" 2>&1 | grep -v "^+"

kubectl create secret tls argocd-agent-client-tls \
  -n argocd \
  --cert=/tmp/tls.crt \
  --key=/tmp/tls.key \
  --context minikube

rm /tmp/tls.key /tmp/tls.crt

echo "✓ Certificates configured"
```

## Step 6: Configure Network Policies

```bash
# Patch Redis network policy to allow agent
kubectl patch networkpolicy argocd-redis-network-policy -n argocd --context minikube --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/ingress/0/from/-",
    "value": {
      "podSelector": {
        "matchLabels": {
          "app.kubernetes.io/name": "argocd-agent-agent"
        }
      }
    }
  }
]'

# Replace restrictive egress policy with permissive one
kubectl delete networkpolicy agent-allow-redis-egress -n argocd --context minikube

kubectl apply -f - --context minikube <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: agent-allow-egress
  namespace: argocd
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: argocd-agent-agent
  policyTypes:
  - Egress
  egress:
  - {}
EOF

echo "✓ Network policies configured"
```

## Step 7: Configure Agent Connection

```bash
# Get the password we saved earlier
AGENT_PASSWORD=$(cat /tmp/fsai-poc-password.txt)

# Configure agent parameters
kubectl patch configmap argocd-agent-params -n argocd --context minikube \
  --patch '{"data":{
    "agent.server.address":"argocd.tikalabs.dev",
    "agent.server.port":"443",
    "agent.mode":"managed",
    "agent.creds":"userpass:/app/config/creds/userpass.creds",
    "agent.tls.root-ca-secret-name":"argocd-agent-ca"
  }}'

# Create userpass credentials secret
kubectl create secret generic argocd-agent-agent-userpass \
  -n argocd \
  --from-literal=credentials="fsai-poc:${AGENT_PASSWORD}" \
  --context minikube

echo "✓ Agent connection configured"
```

## Step 8: Restart Agent and Verify

```bash
# Restart agent
kubectl rollout restart deployment argocd-agent-agent -n argocd --context minikube

# Wait for agent to be ready
kubectl wait --for=condition=Ready pod -n argocd \
  -l app.kubernetes.io/name=argocd-agent-agent \
  --timeout=60s --context minikube

echo ""
echo "✅ Agent fsai-poc deployed!"
echo ""
echo "=== Agent Logs ==="
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-agent-agent \
  --tail=30 --context minikube
```

## Step 9: Verify on Principal

```bash
# Check principal logs
echo ""
echo "=== Principal Logs ==="
kubectl logs -n argocd deployment/argocd-agent-principal \
  --tail=20 --context fsaiostest2-admin | grep -i "fsai-poc\|connected"
```

---

**Copy all these commands to a script or run them one section at a time!**

The agent should show:

- ✅ `Authentication successful`
- ✅ `Connected to argocd-agent-0.5.0`

The principal should show:

- ✅ `An agent connected to the subscription stream client=fsai-poc`
- ✅ `Updated connection status to 'Successful' in Cluster: 'fsai-poc'`
