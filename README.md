# Instructions to Configure Vault Secrets

## Install Prerequisites

Install and set up Docker, Helm, Kubectl and MiniKube. See `init.sh` for instructions.

Install Vault Helm chart:

```BASH
# Add HashiCorp Helm repo
helm repo add hashicorp https://helm.releases.hashicorp.com

# Add Secret Store CSI Helm repo
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm repo update

# Install vault in dev mode with CSI enabled but injector disabled
helm install vault hashicorp/vault \
    --set "server.dev.enabled=true" \
    --set "injector.enabled=false" \
    --set "csi.enabled=true"

# Install Secret Store CSI driver
helm install csi secrets-store-csi-driver/secrets-store-csi-driver \
    --set syncSecret.enabled=true
```

## Set Up Vault Environment

Wait for all previous pods to be ready. Run the following commands inside the vault (`k exec -it vault-0 -- sh`):

```BASH
# Insert a secret at secret/db-pass with the key "password" and the value "db-secret-password"
vault kv put secret/db-pass password="db-secret-password"

# Enable Kubernetes service account authentication:
vault auth enable kubernetes

# Configure Vault to authenticate with Kubernetes API
vault write auth/kubernetes/config kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"

# Create policy named internal-app allowing bound service accounts to read the secret
vault policy write internal-app - <<EOF
path "secret/data/db-pass" {
  capabilities = ["read"]
}
EOF

# Create authentication role named "database" that binds internal-app to a service account named "webapp-sa"
vault write auth/kubernetes/role/database \
    bound_service_account_names=webapp-sa \
    bound_service_account_namespaces=default \
    policies=internal-app \
    ttl=20m

exit
```

## Set Up Pod and Read Secret

Run the following commands:

```BASH
# Package chart:
helm package vault-poc

# install chart:
helm install webapp vault-poc-0.1.0.tgz

# Verify the secret is readable:
k exec $(k get po | grep webapp | cut -d' ' -f1) -- env | grep DB
```
