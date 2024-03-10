# README

Similar to vault-poc v1, but with a few key differences:

* each secret should be inserted into the vault at `secret/<secretPath>` (should match the values file).
* create one policy for all secrets a pod should read. For example, with the commented values:

```BASH
# notice how the path to read a secret is secret/data/<secretPath>, NOT secret/<secretPath>
vault policy write internal-app - <<EOF
path "secret/data/key1" {
  capabilities = ["read"]
}
path "secret/data/key2" {
  capabilities = ["read"]
}
EOF
```

* create an authentication role as previously and insert role name in .Values.secretProviderClass.roleName. Example:

```BASH
# Here the role is named "reader"
vault write auth/kubernetes/role/reader \
    bound_service_account_names=webapp-sa \
    bound_service_account_namespaces=default \
    policies=internal-app \
    ttl=20m
```

```YAML
secretProviderClass:
  # must match name used when creating role in Vault
  roleName: reader
```

* insert a list of secret keys and their paths in the Vault under .Values.objects (see comments there for example).
* Kubernetes Secrets will be created and everything should work.
