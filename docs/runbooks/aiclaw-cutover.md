# AIClaw Cutover

## Preflight Bootstrap

1. Create the target namespaces and service accounts before applying the app `SecretStore` and `ExternalSecret` resources:

   ```bash
   ssh -p 2233 root@136.243.3.57 'kubectl apply -f - <<"EOF"
   apiVersion: v1
   kind: Namespace
   metadata:
     name: app-aiclaw-dev
     labels:
       app.kubernetes.io/name: aiclaw
       app.kubernetes.io/part-of: platform-pilot
       platform.deara.ai/environment: dev
   ---
   apiVersion: v1
   kind: Namespace
   metadata:
     name: app-aiclaw-prod
     labels:
       app.kubernetes.io/name: aiclaw
       app.kubernetes.io/part-of: platform-pilot
       platform.deara.ai/environment: prod
   ---
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: aiclaw
     namespace: app-aiclaw-dev
   ---
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: aiclaw
     namespace: app-aiclaw-prod
   EOF'
   ```

2. Seed Vault from the existing live secret in `apps/aiclaw-secrets` without writing secret values into Git:

   ```bash
   ssh -p 2233 root@136.243.3.57 "set -e; kubectl port-forward -n platform pod/vault-0 18200:8200 >/tmp/aiclaw-vault-pf.log 2>&1 & pf=\$!; trap 'kill \$pf' EXIT; python3 - <<'PY'
   import base64, json, subprocess, time, urllib.request
   root_token = json.load(open('/root/vault-pilot-init.json'))['root_token']
   for _ in range(30):
       try:
           urllib.request.urlopen('http://127.0.0.1:18200/v1/sys/health', timeout=2).read()
           break
       except Exception:
           time.sleep(1)
   else:
       raise SystemExit('vault port-forward did not become ready')
   secret = json.loads(subprocess.check_output(['kubectl', 'get', 'secret', '-n', 'apps', 'aiclaw-secrets', '-o', 'json']))
   data = {k: base64.b64decode(v).decode() for k, v in secret['data'].items()}
   headers = {'X-Vault-Token': root_token, 'Content-Type': 'application/json'}
   for path in ['/v1/kv/data/dev/app-aiclaw/app', '/v1/kv/data/prod/app-aiclaw/app']:
       req = urllib.request.Request('http://127.0.0.1:18200' + path, data=json.dumps({'data': data}).encode(), headers=headers, method='POST')
       with urllib.request.urlopen(req, timeout=10) as resp:
           assert resp.status == 200, path
   PY"
   ```

3. Write the Vault policies and Kubernetes auth roles for `app-aiclaw-dev` and `app-aiclaw-prod` from inside `vault-0`:

   ```bash
   ssh -p 2233 root@136.243.3.57 'token=$(python3 - <<"PY"
   import json
   print(json.load(open("/root/vault-pilot-init.json"))["root_token"])
   PY
   ) && kubectl exec -n platform vault-0 -- sh -lc "cat >/tmp/app-aiclaw-dev.hcl <<'\''EOF1'\''
   path \"kv/data/dev/app-aiclaw/*\" {
     capabilities = [\"read\"]
   }
   EOF1
   cat >/tmp/app-aiclaw-prod.hcl <<'\''EOF2'\''
   path \"kv/data/prod/app-aiclaw/*\" {
     capabilities = [\"read\"]
   }
   EOF2
   export VAULT_ADDR=http://127.0.0.1:8200 VAULT_TOKEN=$token
   vault policy write app-aiclaw-dev /tmp/app-aiclaw-dev.hcl
   vault policy write app-aiclaw-prod /tmp/app-aiclaw-prod.hcl
   vault write auth/kubernetes/role/app-aiclaw-dev bound_service_account_names=aiclaw bound_service_account_namespaces=app-aiclaw-dev policies=app-aiclaw-dev ttl=1h
   vault write auth/kubernetes/role/app-aiclaw-prod bound_service_account_names=aiclaw bound_service_account_namespaces=app-aiclaw-prod policies=app-aiclaw-prod ttl=1h"'
   ```

4. Copy `apps/ghcr-pull-secret` into both app namespaces and apply the checked-in `SecretStore` / `ExternalSecret` manifests:

   ```bash
   ssh -p 2233 root@136.243.3.57 "python3 - <<'PY'
   import json, subprocess
   source = json.loads(subprocess.check_output(['kubectl', 'get', 'secret', '-n', 'apps', 'ghcr-pull-secret', '-o', 'json']))
   for ns in ['app-aiclaw-dev', 'app-aiclaw-prod']:
       obj = {
           'apiVersion': 'v1',
           'kind': 'Secret',
           'metadata': {'name': 'ghcr-pull-secret', 'namespace': ns},
           'type': source['type'],
           'data': source['data'],
       }
       subprocess.run(['kubectl', 'apply', '-f', '-'], input=json.dumps(obj).encode(), check=True)
   PY"

   ssh -p 2233 root@136.243.3.57 "kubectl apply -f /path/to/clusters/dev/apps/aiclaw/addons/secretstore.yaml && kubectl apply -f /path/to/clusters/dev/apps/aiclaw/addons/externalsecret.yaml && kubectl apply -f /path/to/clusters/prod/apps/aiclaw/addons/secretstore.yaml && kubectl apply -f /path/to/clusters/prod/apps/aiclaw/addons/externalsecret.yaml"
   ```

5. Verify ESO readiness and secret materialization:

   ```bash
   ssh -p 2233 root@136.243.3.57 "kubectl get externalsecret -n app-aiclaw-dev aiclaw-secrets -o yaml && kubectl get secret -n app-aiclaw-dev aiclaw-secrets -o name"
   ssh -p 2233 root@136.243.3.57 "kubectl get externalsecret -n app-aiclaw-prod aiclaw-secrets -o yaml && kubectl get secret -n app-aiclaw-prod aiclaw-secrets -o name"
   ```

## Tunnel Caveat

- `/etc/cloudflared/config.yml` on the server now includes `dev.aiclaw.ai`, `next.aiclaw.ai`, and `legacy.aiclaw.ai` routing to `http://127.0.0.1:30080`.
- Direct origin checks on the server return `404` for those hosts, which confirms Traefik sees the hostnames once traffic reaches port `30080`.
- Public requests still return `503`, and `cloudflared` logs report an active ingress set that does not include the new hosts, so the public hostname routing appears to be controlled outside the local file. Resolve that control-plane mismatch before any public validation step that depends on the tunnel.
