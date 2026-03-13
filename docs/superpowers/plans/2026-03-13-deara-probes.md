# Deara Probes Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the broken HTTP health probes for `deara` with TCP probes on port `3000` in both dev and prod values.

**Architecture:** Keep the chart inputs unchanged except for the probe handler type in the two environment values files. Validate the resulting YAML shape directly and, if practical, render the chart using the existing chart/version wiring.

**Tech Stack:** YAML, Helm-based app-template chart, git

---

## Chunk 1: Probe configuration update

### Task 1: Switch dev and prod probes to TCP

**Files:**
- Modify: `clusters/dev/apps/deara/values.yaml`
- Modify: `clusters/prod/apps/deara/values.yaml`
- Test: `clusters/dev/apps/deara/values.yaml`
- Test: `clusters/prod/apps/deara/values.yaml`

- [x] **Step 1: Update the dev probe handlers**

Change `probes.readiness` and `probes.liveness` in `clusters/dev/apps/deara/values.yaml` from `httpGet` with `/api/health` to `tcpSocket` on port `3000`.

- [x] **Step 2: Update the prod probe handlers**

Change `probes.readiness` and `probes.liveness` in `clusters/prod/apps/deara/values.yaml` from `httpGet` with `/api/health` to `tcpSocket` on port `3000`.

- [x] **Step 3: Verify the YAML edits directly**

Run: `git diff -- clusters/dev/apps/deara/values.yaml clusters/prod/apps/deara/values.yaml`
Expected: only the readiness/liveness handler blocks change from `httpGet` to `tcpSocket` and keep port `3000`.

## Chunk 2: Practical validation and commit

### Task 2: Validate and commit the probe fix

**Files:**
- Modify: `docs/superpowers/plans/2026-03-13-deara-probes.md`
- Test: `clusters/dev/apps/deara/values.yaml`
- Test: `clusters/prod/apps/deara/values.yaml`

- [x] **Step 1: Run practical validation**

Run a lightweight validation that is available in the repo/tooling, preferring Helm/template rendering if the chart can be resolved with the current environment. If not, capture the limitation and rely on YAML diff inspection.

- [x] **Step 2: Commit the change**

Run: `git add clusters/dev/apps/deara/values.yaml clusters/prod/apps/deara/values.yaml docs/superpowers/plans/2026-03-13-deara-probes.md && git commit -m "fix(gitops): switch deara probes to tcp"`
Expected: commit succeeds on the current branch.
