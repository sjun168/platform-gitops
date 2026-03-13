# Deara Public Cutover

## Public Cutover

1. Prepare and approve a GitOps PR that changes `clusters/prod/apps/deara/values.yaml` hosts from `next.deara.ai` to `deara.ai` and `www.deara.ai`.
2. Keep `docs/runbooks/attachments/deara-legacy-ingress-precutover.yaml` as the exact pre-cutover backup.
3. Patch the legacy ingress in namespace `apps` so it stops owning `deara.ai` and `www.deara.ai`, and instead serves `legacy.deara.ai` and `www-legacy.deara.ai` with a lower explicit Traefik priority than the new prod ingress.
4. Merge the approved GitOps PR immediately after the legacy ingress patch.
5. Verify `https://deara.ai/zh`, `https://www.deara.ai/zh`, and `https://deara.ai/api/llm-models` return success from the new `app-deara-prod` deployment.

## Rollback Window

1. Keep the legacy Helm release running for at least 24 hours after cutover.
2. For app rollback, revert the GitOps prod image tag through a PR.
3. For host-routing rollback, restore `docs/runbooks/attachments/deara-legacy-ingress-precutover.yaml` and revert the cutover PR.

## Verification Checklist

- `deara-prod` Argo app is `Synced Healthy`
- `app-deara-prod` deployment image matches the intended immutable `sha-*` tag
- `https://next.deara.ai/api/llm-models` returns 200 before cutover
- `https://deara.ai/api/llm-models` returns 200 after cutover
- `https://legacy.deara.ai/zh` returns 200 during the rollback window
