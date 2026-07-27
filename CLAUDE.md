# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single Helm chart (`Fiware-Helm/`) that deploys a FIWARE stack on Kubernetes: Orion Context Broker, MongoDB, CrateDB, and QuantumLeap, plus optional cross-cluster federation between an edge (Jetson) Orion and a cloud Orion over Ziti. There is no application source code — this repo *is* the chart.

## Commands

```bash
# Lint the chart
helm lint ./Fiware-Helm

# Render templates locally without installing
helm template fiware ./Fiware-Helm

# Install / upgrade (namespace inherited from -n since values.yaml namespace is "")
helm upgrade --install fiware ./Fiware-Helm -n fiware --create-namespace

# With overrides
helm upgrade --install fiware ./Fiware-Helm -n fiware -f my-values.yaml
```

There are no unit tests or CI config in this repo — `helm lint` / `helm template` are the only correctness checks available before a real deploy.

Note: go-template comments (`{{/* ... */}}`) placed immediately after a `---` document separator must **not** use `-` trim markers (i.e. not `{{- /* ... */ -}}`). Trimming there eats the newline between `---` and the next resource's `apiVersion` line, which `helm lint` rejects as an invalid document separator (`helm template` masks it because Helm injects its own `# Source: ...` comment line in between). This bit every multi-resource template in the chart until it was fixed — keep new multi-doc templates consistent with this.

## Architecture

See README.md for the full component diagram and table. Points worth knowing that aren't obvious from a single template:

- Every component is gated by its own `enabled` flag in `values.yaml` and its template's top-level `{{- if .Values.X.enabled }}`.
- The Orion → QuantumLeap subscription (writing entities into CrateDB) is **not** standing infrastructure — it only gets created as a side effect of a federation Job (see below), guarded by `quantumLeap.enabled`. With both `orion.federation.enabled` and `orion.biogasFederation.enabled` false, nothing subscribes QuantumLeap to Orion and entities written directly to Orion never reach CrateDB.
- `iotAgent` is a disabled placeholder: values are kept in `values.yaml` but the matching template was removed. Re-enabling it means restoring a template, not just flipping the flag.

### Federation

Two **independent** federation setups exist (entity types, fiware-service/path, and target host differ — see README.md), each registering subscriptions on a remote (edge) Orion so it pushes entities to this cloud cluster over Ziti:

| | Energy / Jetson-1 | BPO / Biogas |
|---|---|---|
| values block | `orion.federation` | `orion.biogasFederation` |
| ConfigMap | `orion-subscription-configmap.yaml` | `orion-subscription-configmap-biogas.yaml` |
| Job | `orion-subscription-job.yaml` | `orion-subscription-job-biogas.yaml` |

They are kept as separate template pairs (not a shared helper/loop) deliberately, so the two don't get mixed up — when changing federation logic, **mirror the change in both files** rather than trying to unify them.

Each Job is a Helm post-install/post-upgrade hook that:
1. Waits for the target (edge) Orion's `/version` endpoint to respond.
2. Registers a subscription on the target Orion so it pushes entities to the cloud (`orion.federation.host:port`).
3. If `quantumLeap.enabled`, also registers a local subscription on the cloud Orion so federated entities land in CrateDB via QuantumLeap.
4. Is idempotent: it lists existing subscriptions and greps for a matching `description` string before POSTing — **the description string is the dedup key**, so changing entity types or the description text will cause a duplicate subscription to be registered rather than updating the old one.

The existence check fetches and greps in separate steps (not piped) on purpose — under `set -e`, a failed `curl` in a pipeline would otherwise be swallowed and read as "no existing subscription," registering a duplicate. Preserve that structure if touching `register_subscription()`.

## Conventions to follow when editing templates

- Every resource wraps its namespace block as `{{- if .Values.namespace }}namespace: {{ .Values.namespace }}{{- end }}` — leave `namespace: ""` in `values.yaml` so the chart stays installable into any namespace via `-n`.
- PVC/StatefulSet `storageClassName` is hardcoded to `openebs-hostpath` across `mongo.yaml` and `crate.yaml` — there's no values knob for it; edit both templates if the target cluster uses a different storage class.
- Memory requests/limits are values-driven per component (`<component>.memoryRequest` / `<component>.memoryLimit`); don't hardcode resource numbers into templates.
- Bump `version` (and `appVersion` if the deployed app itself changed) in `Chart.yaml` before packaging/pushing a new chart version, per the global Helm release process.
