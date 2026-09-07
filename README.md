# orion-fed-helm

A Helm chart for deploying a stack of core **FIWARE** components on Kubernetes: **Orion Context Broker**, **MongoDB**, **CrateDB**, and **QuantumLeap**. It also supports **federation** subscriptions between edge (Jetson) and cloud Orion instances over Ziti.


## Architecture

```
                 ┌─────────────┐        ┌──────────────┐
   entities  --->│  Orion CB   │<------>│   MongoDB    │
                 │ (context    │        │ (persistence)│
                 │  broker)    │        └──────────────┘
                 └──────┬──────┘
                        │ subscription
                        v
                 ┌─────────────┐        ┌──────────────┐
                 │ QuantumLeap │------->│   CrateDB    │
                 │ (time series│        │ (time-series │
                 │  translator)│        │   storage)   │
                 └─────────────┘        └──────────────┘
```

- **Orion** is the context broker: it accepts and stores entities (state) in **MongoDB**.
- A **subscription** on Orion forwards changes to **QuantumLeap**, which translates the data into time series and writes it to **CrateDB**.
- Optionally, a second Orion running at the edge (e.g. a Jetson) can **federate** entities into this cluster over Ziti (see [Federation](#federation)).

## Components

| Component     | What it does                                          | Version/Image              |
|---------------|--------------------------------------------------------|------------------------------|
| `mongo`       | Storage backend for Orion                               | `mongo:5.0`                   |
| `orion`       | FIWARE Context Broker (NGSIv2)                           | `fiware/orion:latest`         |
| `crate`       | Time-series database backing QuantumLeap                 | `crate:4.6.7`                 |
| `quantumLeap` | Translates NGSI notifications into time series in Crate  | `orchestracities/quantumleap:latest` |
| `iotAgent`    | (optional, **disabled** by default)                       | -                              |

Each component can be enabled/disabled independently via its `enabled` flag in [`values.yaml`](orion-fed-helm/values.yaml).

## Federation

The chart supports **two independent federation setups**, each with its own ConfigMap + post-install Job that registers the subscriptions automatically:

### 1. Energy / Jetson-1 edge
- Entities: `ACMeasurement`
- Fiware service: `energy` (path `/`)
- The cloud registers a subscription **on the edge Orion** (`orion.federation.targetHost`), so the edge pushes entities to the cloud Orion.
- Templates: [`orion-subscription-configmap.yaml`](orion-fed-helm/templates/orion-subscription-configmap.yaml), [`orion-subscription-job.yaml`](orion-fed-helm/templates/orion-subscription-job.yaml)

### 2. BPO / Biogas
- Entities: `Digester`, `CHPUnit`
- Fiware service: `bpo` (path `/v1`)
- Kept as a separate instance so it doesn't get mixed up with the energy federation.
- Templates: [`orion-subscription-configmap-biogas.yaml`](orion-fed-helm/templates/orion-subscription-configmap-biogas.yaml), [`orion-subscription-job-biogas.yaml`](orion-fed-helm/templates/orion-subscription-job-biogas.yaml)

Both Jobs:
1. Wait for the target (edge) Orion to become ready.
2. Register the federation subscription on the target Orion (push entities to the cloud over Ziti).
3. If `quantumLeap.enabled` is `true`, also register a local subscription on the cloud Orion so federated entities get persisted to CrateDB.
4. Are idempotent — they check whether a subscription already exists (by `description`) before creating it again.

Enable/disable each independently via `orion.federation.enabled` and `orion.biogasFederation.enabled`.

## Installation

```bash
helm install orion-fed ./orion-fed-helm -n <namespace> --create-namespace
```

Or, leaving `namespace: ""` in `values.yaml`, the chart inherits the namespace from `-n`:

```bash
helm upgrade --install orion-fed ./orion-fed-helm -n orion-fed --create-namespace
```

For custom settings, create your own `my-values.yaml` and pass it with `-f`:

```bash
helm upgrade --install orion-fed ./orion-fed-helm -n orion-fed -f my-values.yaml
```

## Configuration

The main parameters live in [`orion-fed-helm/values.yaml`](orion-fed-helm/values.yaml):

```yaml
namespace: ""        # Empty = inherited from helm install -n <ns>

mongo:
  enabled: true
  storage: 10Gi

orion:
  enabled: true
  nodePort: 30897
  federation:
    enabled: true
    targetHost: orion-cb-jetson-1.ziti
    entityTypes: [ACMeasurement]
    ...
  biogasFederation:
    enabled: true
    targetHost: orion-cb-jetson.ziti
    entityTypes: [Digester, CHPUnit]
    ...

crate:
  enabled: true
  nodePort: 30420
  storage: 10Gi

quantumLeap:
  enabled: true

iotAgent:
  enabled: false       # placeholder — values are kept around but the matching
                        # template needs to be restored to re-enable it
```

Note: `storageClassName` on the PVC/volumeClaimTemplates is hardcoded to `openebs-hostpath` — adjust the templates if your cluster uses a different storage class.

## Chart layout

```
orion-fed-helm/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── mongo.yaml                          # PVC + Deployment + Service
    ├── orion.yaml                          # Deployment + Service (NodePort)
    ├── crate.yaml                          # Service + StatefulSet
    ├── quantumleap.yaml                    # Deployment + Service
    ├── orion-subscription-configmap.yaml       # federation subscriptions (energy)
    ├── orion-subscription-job.yaml             # registration job (energy)
    ├── orion-subscription-configmap-biogas.yaml # federation subscriptions (bpo)
    └── orion-subscription-job-biogas.yaml       # registration job (bpo)
```
