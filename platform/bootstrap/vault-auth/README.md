# Vault Kubernetes Auth Bootstrap

This directory captures the exact pilot Vault auth inputs used for Task 4.

Files:

- `auth-config.yaml` - Vault Kubernetes auth and KV mount parameters
- `vault-tokenreview-binding.yaml` - Kubernetes RBAC required for token review
- `policies/app-deara-dev.hcl`
- `policies/app-deara-prod.hcl`
- `policies/app-aiclaw-dev.hcl`
- `policies/app-aiclaw-prod.hcl`
- `roles/app-deara-dev.yaml`
- `roles/app-deara-prod.yaml`
- `roles/app-aiclaw-dev.yaml`
- `roles/app-aiclaw-prod.yaml`

Apply the Kubernetes RBAC declaratively:

`kubectl apply -k platform/bootstrap/vault-auth`

Then configure Vault from inside `vault-0` using the checked-in definitions:

1. Enable `kv-v2` at `kv` if it is not already enabled.
2. Enable the `kubernetes` auth mount if it is not already enabled.
3. Write `auth/kubernetes/config` with:
   - `kubernetes_host=https://kubernetes.default.svc:443`
   - `kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`
   - `token_reviewer_jwt=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)`
4. Write the policies from `policies/`.
5. Write the roles from `roles/`.

Exact role payloads:

- `app-deara-dev` -> service account `deara` in namespace `app-deara-dev`, policy `app-deara-dev`, TTL `1h`
- `app-deara-prod` -> service account `deara` in namespace `app-deara-prod`, policy `app-deara-prod`, TTL `1h`
- `app-aiclaw-dev` -> service account `aiclaw` in namespace `app-aiclaw-dev`, policy `app-aiclaw-dev`, TTL `1h`
- `app-aiclaw-prod` -> service account `aiclaw` in namespace `app-aiclaw-prod`, policy `app-aiclaw-prod`, TTL `1h`

Exact `aiclaw` bootstrap commands:

`vault policy write app-aiclaw-dev platform/bootstrap/vault-auth/policies/app-aiclaw-dev.hcl`

`vault policy write app-aiclaw-prod platform/bootstrap/vault-auth/policies/app-aiclaw-prod.hcl`

`vault write auth/kubernetes/role/app-aiclaw-dev bound_service_account_names=aiclaw bound_service_account_namespaces=app-aiclaw-dev policies=app-aiclaw-dev ttl=1h`

`vault write auth/kubernetes/role/app-aiclaw-prod bound_service_account_names=aiclaw bound_service_account_namespaces=app-aiclaw-prod policies=app-aiclaw-prod ttl=1h`
