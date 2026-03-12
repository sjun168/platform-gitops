# Cluster layout

`clusters/dev` and `clusters/prod` hold the live desired state for each environment.

## Structure

- `namespaces/` contains namespace bootstrap manifests that must exist before app sync.
- `apps/<name>/app.yaml` is the ApplicationSet metadata contract for one app instance.
- `apps/<name>/values.yaml` provides the Helm values consumed through the shared OCI chart.
- `apps/<name>/addons/` contains GitOps-owned environment overlays such as `SecretStore`, `ExternalSecret`, and `NetworkPolicy`.

## Task 5 baseline state

- Task 5 creates the `AppProject`, `ApplicationSet`, root app, and namespace/bootstrap wiring before any app instance metadata exists under `clusters/*/apps/*/app.yaml`.
- An empty `clusters/*/apps/*/app.yaml` result is intentional at this revision; generated child Applications begin in Task 6 when the first app metadata files are added.
- If the `ApplicationSet` exists without child Applications during this stage, treat that as expected sequencing rather than drift.

## Promotion rules

- Dev updates are expected to flow automatically through GitOps after CI updates the environment values.
- Prod updates must arrive through reviewed GitOps changes and remain pinned to immutable image and chart versions.
- Changes under `applicationsets/`, `projects/`, `platform/root/`, and `clusters/prod/` are treated as prod-impacting platform changes.

## Pilot-only exceptions

- `deara` keeps a temporary compatibility path to shared services outside the final `shared-dev` and `shared-prod` split.
- The app-scoped Vault `SecretStore` currently talks to in-cluster Vault over HTTP and is annotated as a temporary pilot exception.
- Do not reuse either exception for other applications; remove them in the follow-up shared-services isolation plan.
