# New Project Fast Path

This is the shortest safe path for onboarding a new project onto the current platform.

Use this when the project is a normal Next.js / Node web app or API and does **not** need a custom platform capability.

If the project needs unusual infra, billing/webhook isolation, or custom network flows, stop and write a dedicated migration plan first.

## 0. Preconditions

Before touching the new project, confirm the shared platform is already healthy:

- `platform-gitops-root` is `Synced Healthy`
- `platform-charts/app-template` latest release is published in GHCR
- Vault and External Secrets Operator are healthy in `platform`
- Traefik is healthy in `kube-system`
- GitHub repo secrets are available for the app repo:
  - `GHCR_USERNAME`
  - `GHCR_TOKEN`
  - `GITOPS_SSH_KEY`
  - `GITOPS_PR_TOKEN`

## 1. Copy the Known-Good App Files First

Start by copying the most recent working shapes from a migrated app (`aiclaw` or `deara`).

For application repo work:

- `deploy/app.yaml`
- `scripts/update_gitops_values.py`
- `tests/test_update_gitops_values.py`
- `.github/workflows/release.yaml`
- `.github/workflows/promote-prod.yaml`
- legacy workflow notice pattern from `.github/workflows/docker-build.yaml`

For GitOps repo work:

- `clusters/dev/apps/<app>/app.yaml`
- `clusters/dev/apps/<app>/values.yaml`
- `clusters/dev/apps/<app>/addons/secretstore.yaml`
- `clusters/dev/apps/<app>/addons/externalsecret.yaml`
- `clusters/dev/apps/<app>/addons/networkpolicy.yaml`
- same set for `prod`

Do **not** author new YAML patterns from scratch if a working file already exists for the same concern.

## 2. Fill Only the App-Specific Contract Points

When adapting the copied files, change only these fields first:

- app name
- repo/image name
- namespace names:
  - `app-<name>-dev`
  - `app-<name>-prod`
- container and service port
- secret name
- Vault secret paths
- ingress hosts
- any real extra network egress requirements

Keep these defaults unless the app proves they are wrong:

- `chartVersion: 0.1.1`
- `imagePullSecrets: [{name: ghcr-pull-secret}]`
- `serviceAccount.name: <app>`
- `traefik.ingress.kubernetes.io/router.priority: "1000"`
- `networkPolicy.ingressControllerNamespace: kube-system`
- TCP probes on the application port

## 3. Do These Preflight Checks Before the First Release

This is where most time was lost on `deara` and `aiclaw`.

### App repo

- confirm `pnpm build` passes
- run `pnpm audit --prod --audit-level high`
- patch direct dependency blockers before the first release run
- use Alpine 3.23 base and `apk upgrade --no-cache`
- if only Next vendored toolchain findings remain, add a **narrow** `.trivyignore.yaml`

### URL strategy

Before standardizing promotion, check whether the app bakes environment-specific URLs at build time.

Search for:

```bash
grep -R "NEXT_PUBLIC_APP_URL\|AUTH_URL\|APP_URL\|APP_BASE_URL" src -n
```

If found, do one of these before proceeding:

- refactor to derive runtime origin where practical
- or explicitly choose one canonical production URL strategy and accept that dev/shadow are smoke environments

Do **not** leave hidden build-time env coupling unresolved and then assume a single immutable image can safely move from dev to prod.

## 4. Seed Secrets Before You Test the App

Do not wait for the first rollout to fail on missing secrets.

### Vault paths

Create these per app:

- `kv/dev/app-<name>/app`
- `kv/prod/app-<name>/app`

### Secret names

Match the live app expectation exactly, for example:

- `aiclaw-secrets`
- `deara-env`

### Seed source

If this is a migration, copy the current live K8s secret into Vault for both dev and prod as an initial seed. Tighten or split later.

### GHCR pull secret

Copy `apps/ghcr-pull-secret` into the new app namespaces before the first rollout.

## 5. Wire Cloudflare/Tunnel Hosts Before Public Validation

If the project uses Cloudflare Tunnel, add the dev/shadow hosts before the first smoke run.

Recommended host set:

- `dev.<domain>`
- `next.<domain>`
- `legacy.<domain>`

Important lesson:

- local `/etc/cloudflared/config.yml` may **not** be the active source of truth if the tunnel is remotely managed
- if public hosts still return `503` after local config changes, verify the live remote-managed tunnel config before debugging the app

When tunnel routing cannot be fixed immediately, use the proven temporary validation path:

- direct HTTPS to `136.243.3.57:30443`
- `--resolve <host>:30443:136.243.3.57`

This is acceptable for dev/shadow smoke validation, but do not rely on it for long-term public cutover checks.

## 6. Use This Order Every Time

Do not improvise the sequence.

1. App repo blockers fixed
2. GitOps manifests committed
3. Vault auth files committed
4. secrets seeded into Vault
5. `ghcr-pull-secret` copied
6. tunnel routes prepared
7. dev release run
8. verify `app-<name>-dev` healthy
9. prod shadow promotion
10. rollback drill on shadow prod
11. public cutover
12. retire legacy path

## 7. Smoke Check Rules

### Probes

Default to TCP probes first.

Reason:

- fake HTTP health endpoints caused repeated rollout failures during `deara`
- TCP probes matched the working legacy deployments for both `deara` and `aiclaw`

Only switch to HTTP probes if the app has a real, verified health endpoint.

### Release smoke URL

Do not guess the smoke URL.

Validate one real endpoint first:

- public localized page, e.g. `/zh`
- or a real lightweight API endpoint already proven to return `200`

Do **not** use `/api/health` unless the app actually implements it.

## 8. Network Policy Rules

Start with the smallest policy that works.

Baseline:

- ingress from Traefik in `kube-system`
- DNS egress to `kube-dns`

Add extra egress only from real evidence, for example:

- `infra/postgres:5432`
- `clawbot/litellm-svc:4000`

Do not cargo-cult another app's egress list.

## 9. Public Cutover Rules

Do not cut directly from old prod to new prod without shadow validation.

### Before merge

- old ingress backup saved to `docs/runbooks/attachments/`
- cutover PR prepared in `platform-gitops`
- shadow prod already healthy
- rollback drill already proven

### During cutover

- patch old ingress off the primary hostname first
- then merge the GitOps PR immediately
- verify public host returns `200`

### After cutover

- keep legacy path for a short rollback window if you still need it
- otherwise remove the old Argo app and old runtime resources promptly

## 10. Cleanup Rules

After the new path is stable:

- delete the old Argo application
- delete the old runtime resources
- delete old secrets no longer referenced
- mark the old repo/path inactive in docs
- archive the old GitOps repo if it is no longer needed
- remove local worktrees and temporary branches

## 11. Minimal New-Project Checklist

If you just want the shortest checklist, use this:

- [ ] copy `deara`/`aiclaw` release workflow and updater files
- [ ] patch app dependencies and Docker base before first release
- [ ] confirm runtime URL strategy
- [ ] add app manifests to `platform-gitops`
- [ ] add Vault roles/policies + seed secrets
- [ ] copy `ghcr-pull-secret`
- [ ] add tunnel hosts or confirm remote-managed config
- [ ] run dev release
- [ ] verify dev app healthy
- [ ] promote to shadow prod
- [ ] run rollback drill
- [ ] cut over public host
- [ ] delete legacy app/path

## 12. If Something Fails, Check These First

In order:

1. wrong GHCR credential path
2. SSH key not persisted across workflow steps
3. Trivy action wrapper instead of direct container execution
4. app-side high/critical dependency CVEs
5. Alpine `zlib` / base image CVEs
6. missing `imagePullSecrets`
7. Vault path exists but has no secret data
8. fake `/api/health` probe path
9. `networkPolicy.ingressControllerNamespace` wrong (should usually be `kube-system`)
10. missing DB/LLM egress in `NetworkPolicy`
11. wildcard ingress or tunnel route taking precedence over exact host
12. Cloudflare remote-managed tunnel ignoring local config
