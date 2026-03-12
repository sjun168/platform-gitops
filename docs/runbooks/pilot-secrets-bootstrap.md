# Pilot Secrets Bootstrap

This pilot uses two Argo CD Applications in `platform/bootstrap/` to declare the
cluster-local secrets control plane:

- `platform/bootstrap/external-secrets.yaml`
- `platform/bootstrap/vault.yaml`

The manifests intentionally match the live pilot installation:

- External Secrets Operator chart `2.1.0`
- Vault chart `0.32.0`
- `platform` namespace destination
- single-node Vault, no HA, no CSI, no injector

Temporary exception: `pilot-vault-http`

- Scope: only in-cluster traffic from ESO to `http://vault.platform.svc.cluster.local:8200`
- Reason: the pilot needs a working Vault backend before a full TLS rollout is designed
- Limit: do not expose Vault over ingress or external networking while this exception exists
- Exit criteria: replace all `http://vault.platform.svc.cluster.local:8200` references with
  an internal TLS endpoint and remove the `platform.deara.ai/temporary-exception` annotations

Manual pilot steps still required after Vault chart sync:

1. Initialize Vault once:

   `kubectl exec -n platform vault-0 -- vault operator init -key-shares=1 -key-threshold=1 -format=json`

2. Store the output securely for operators. The pilot used root-only node storage at
   `/root/vault-pilot-init.json`; move this into the team's real secret-handling flow before
   expanding beyond the pilot.

3. Unseal Vault with the stored key:

   `kubectl exec -n platform vault-0 -- vault operator unseal <unseal-key>`

4. Allow Vault to review Kubernetes service account tokens:

   `kubectl create clusterrolebinding vault-tokenreview --clusterrole=system:auth-delegator --serviceaccount=platform:vault`

5. Configure Vault Kubernetes auth inside `vault-0`:

   - enable `kv-v2` at `kv`
   - enable the `kubernetes` auth mount
   - configure `auth/kubernetes/config` with the pod service account token and cluster CA

6. Create least-privilege pilot policies and roles:

   - `app-deara-dev` -> `kv/data/dev/app-deara/*`
   - `app-deara-prod` -> `kv/data/prod/app-deara/*`

The app `SecretStore` manifests intentionally reference the future Helm-managed `deara`
service account in each app namespace. They validate against the cluster API now, but they
should be applied live only after that service account exists in Task 6.
