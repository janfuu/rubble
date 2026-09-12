# Distribution: what the installer ships, what you bring

Status: design, 2026-09-12. Rubble is a product with a domain (`rubble.cloud`), so the
recipe has to install itself. This document decides the shape of that installer and
which ingredients are bundled addons versus bring-your-own. It is written for
contributors and for operators evaluating what a Rubble instance needs.

## The shape

One principle from the rest of the design carries over unchanged: **an instance is a
git repository.** The installer's job is to *write that repository* and bootstrap the
cluster from it, not to hold state of its own.

```
curl -sSL https://get.rubble.cloud | sh          # installs the rubble CLI
rubble cluster check                              # preflight: k8s version, storage class, LB, GPU, runtimes
rubble cluster init --profile quickstart          # writes ./rubble-instance/: values, addon toggles, app-of-apps
rubble cluster up                                 # bootstraps the applier and syncs the repo
rubble cluster status                             # every component: Synced/Healthy, URLs, next steps
```

Under the repository there is exactly one artifact type: **a Helm umbrella chart per
component**, the same pattern the reference instance uses today (`Chart.yaml` naming the
upstream chart + `values.yaml`). The umbrella chart `charts/rubble` lists them as
dependencies with a `<component>.enabled` condition each. So there are three ways to use
the same charts, and they never diverge:

| way | who | applier |
|---|---|---|
| `helm install rubble oci://ghcr.io/janfuu/rubble/charts/rubble -f values.yaml` | evaluators, CI | Helm, no git |
| `rubble cluster init/up` with `applier: local` | quickstart, single-node dev | the CLI commits to the instance repo, then applies what it committed |
| `rubble cluster init/up` with `applier: argocd` | homelab, production | ArgoCD app-of-apps over the instance repo |

The audit-trail invariant ("agent definitions come only from git") holds in all three for
*agents*: `rubble deploy` always commits. Only the platform's own applier differs.

## Profiles

| profile | target | what it assumes | footprint (estimate, to be measured) |
|---|---|---|---|
| **quickstart** | one node: kind, k3d, k3s, a 16 GB VM | nothing external except **one OpenAI-compatible endpoint you already run** (a llama.cpp on the host, Ollama, or a provider key). Every addon bundled single-replica, self-signed TLS on `*.rubble.localhost` or an `sslip.io` name, secrets generated locally, `applier: local` | ~6 GB RAM without observability, ~9 GB with |
| **homelab** | a few nodes, a GPU or two | bundled addons where cheap, BYO where you already have it (NAS S3, existing Keycloak), `applier: argocd`, real TLS via cert-manager | as quickstart plus HA for the DB and identity |
| **production** | many nodes, multi-tenant | BYO object store, IdP federation, external secrets, HA everything, GPU nodes labelled, sandbox runtime installed | sized per tenant |

A profile is a values preset plus a set of addon toggles. Nothing else.

## Component matrix

Legend: **core** = always installed · **addon** = bundled, `enabled: true` by default in
the profile shown, replaceable by BYO · **BYO** = never bundled, a value pointing at yours.

| component | role in the recipe (Bedrock name) | bundled as | BYO alternative | default in quickstart / homelab / prod | upstream chart | license of the upstream |
|---|---|---|---|---|---|---|
| agentgateway + CRDs, Gateway API CRDs | the gateway: models, MCP, A2A, policies (Runtime front, Gateway) | **core** | — | on / on / on | `cr.agentgateway.dev/charts` | Apache-2.0 |
| Rubble services: console, memory, kb-ingest, guardrail webhook, policy (Cedar), sandbox broker | the product | **core** | — | on / on / on | `charts/rubble` | MIT |
| Keycloak, pre-configured realm + CNPG database | identity broker (Identity) | **addon** | your Keycloak (26.5.6+, realm import applied) — other IdPs federate *into* the bundled one | on / on / on-or-BYO | keycloak-operator or the official image + our realm JSON | Apache-2.0 |
| SPIRE server + agent + CSI + OIDC discovery | workload identity for agents (Identity) | **addon** | your SPIRE (trust domain + JWKS URL as values) | on / on / on | `spiffe/helm-charts-hardened` | Apache-2.0 |
| CloudNativePG operator + `rubble-pg` cluster | Postgres for memory events, KB ledger, Keycloak, registry | **addon** | a connection string per database | on / on / BYO-or-on | `cloudnative-pg/charts` | Apache-2.0 |
| Garage | S3 for sessions, KBs, models, traces (the S3 in "S3 and so on") | **addon** | any S3: endpoint + key per tenant bucket | on / on-or-BYO / BYO | Garage's `script/helm/garage` (vendored) | **AGPL-3.0** — shipped as an unmodified container, which is fine; documented |
| Qdrant | vectors for KBs and memory (Knowledge Bases, Memory) | **addon** | your Qdrant URL + key | on / on / on-or-BYO | `qdrant/qdrant-helm` | Apache-2.0 |
| models | the models behind the endpoint (Bedrock Runtime backends) | **BYO, always.** Rubble ships no model-serving layer: any OpenAI-compatible server — llama.cpp, vLLM, Ollama, or a cloud provider, since agentgateway fronts OpenAI, Anthropic, Vertex and Bedrock natively. Each endpoint is one `AgentgatewayModel` entry in values (name, URL, optional key, capabilities: vision, tools, embeddings). | — | quickstart: point at a llama.cpp on the host (`host.docker.internal` from kind) or a provider key / your GPU servers / same | — | model weights carry their own licenses |
| embedding model | KB and memory vectors | **BYO**: any `/v1/embeddings` endpoint, dimension read at KB creation | — | same as models | — | |
| agentregistry | catalog (Registry) | **addon** | — | off / on / on | `ghcr.io/agentregistry-dev/agentregistry/charts` (0.4+) | Apache-2.0 |
| kube-prometheus-stack, Loki, Tempo, Alloy or otel-collector | metrics, logs, traces, dashboards (Observability) | **addon** | an OTLP endpoint + Prometheus remote-write URL | off / on / on-or-BYO | prometheus-community, grafana | Apache-2.0 / **AGPL-3.0** (Grafana, Loki, Tempo) |
| cert-manager | real TLS | **addon** | your certs as a Secret | off (self-signed) / on / on | `charts.jetstack.io` | Apache-2.0 |
| external-secrets operator | secrets from a store | **addon** | plain Secrets (quickstart generates them) | off / on / on | `charts.external-secrets.io` | Apache-2.0 |
| ArgoCD | the GitOps applier | **addon** | your ArgoCD (we register an `AppProject` + app-of-apps) or `applier: local` | off / on / on | `argoproj/argo-helm` | Apache-2.0 |
| agent-sandbox controller + `SandboxTemplate`s | Code Interpreter, session isolation | **addon** | — | on (runc, **no isolation**, clearly labelled) / on + gVisor / on + gVisor or Kata | upstream release manifests, vendored | Apache-2.0 |
| gVisor runtime (`runsc`) + `RuntimeClass` | isolation for sandboxes | **node addon** (DaemonSet installer, opt-in) | you install it | off / on / on | Rubble's own, modelled on kata-deploy | Apache-2.0 |
| Kata Containers | microVM isolation | **node addon**, KVM required | you install it | off / off / opt-in | kata-deploy | Apache-2.0 |
| KEDA | scale-to-zero metrics (needs k8s 1.37) | **addon** | — | off / opt-in / on | `kedacore/charts` | Apache-2.0 |
| ingress in front of agentgateway | TLS termination, hostnames | **none needed**: agentgateway is a Gateway API implementation and is the only public entry (models, MCP, agents, console) | your existing ingress can front it | Gateway listener with TLS / same / same | — | |
| storage class | PVCs for Postgres, Qdrant, Garage | **BYO** | — | preflight checks a default class exists | | |
| GPU drivers / device plugins / DRA | GPU scheduling | **BYO** | — | preflight detects and reports | | |
| LoadBalancer | a reachable IP | **BYO** (k3s ServiceLB, MetalLB, cloud) | — | preflight; quickstart falls back to NodePort | | |

Everything in the reference instance's `platform/`, `spire/` and `agentgateway/`
directories is one of these rows already; the port is mechanical.

## What the installer cannot do, and says so

Node-level things stay outside Kubernetes' reach and outside the installer's: GPU drivers,
a KVM-capable node, a container runtime for gVisor or Kata, a default storage class, a
LoadBalancer. `rubble cluster check` detects each, prints what is missing and what it
costs you (no GPU → CPU model only; no gVisor → sandboxes run without isolation and the
console shows a warning; no LB → NodePort URLs). It never guesses.

## Configuration and secrets in quickstart

`rubble cluster init` generates every secret the bundled addons need (Keycloak admin,
database owners, Garage keys, the first API key, the console's OIDC client secret) into the
instance repo's `secrets/` **as SOPS-encrypted files with an age key kept in
`~/.rubble/`**, so even the quickstart repo is safe to push. External Secrets is the
production path; SOPS is the zero-dependency path. Both feed the same Secret names.

## Open decisions

| # | decision | options | recommendation |
|---|---|---|---|
| D1 | applier for quickstart | `local` (CLI applies what it committed) · always ArgoCD · a tiny in-cluster git server + ArgoCD | `local`. ArgoCD needs a reachable git remote, which a laptop quickstart does not have; the audit trail is the local repo's history, which is real. |
| D2 | Keycloak packaging | official image + realm import JSON · keycloak-operator | official image + `--import-realm`: fewer moving parts, the operator adds a CRD layer for no gain at this size. Blocker inherited: production mode breaks the SPIFFE exchange in the reference instance; must be root-caused before the addon ships (it cannot ship `start-dev`). |
| D3 | object store default | Garage · SeaweedFS · MinIO | Garage: small, S3-faithful enough, proven in the reference instance. AGPL is fine for an unmodified container; say so in the docs. MinIO's community edition is no longer a safe default. |
| D4 | models | ~~ship model packs~~ · **BYO endpoints only** | Settled 2026-09-12 (Jan): no model-serving layer. Any running llama.cpp will do. The docs carry a short "serving models for Rubble" page with the reference instance's measured llama.cpp flags as *examples*, not as a chart. |
| D5 | how the CLI is delivered | `curl \| sh` from `get.rubble.cloud` + GitHub releases · pip only | both, and `pipx install rubble-agents` as the Python-native path. The script verifies a checksum. |
| D6 | what lives under `rubble.cloud` | docs, `get.` script, chart index; later: a hosted "try it" instance | docs + get + chart index now. A hosted instance is a separate product decision. |
| D7 | vendored vs referenced upstream charts | pin versions in `Chart.yaml` and let Helm fetch · vendor tarballs into the repo | pin and fetch, with a lockfile; vendor only what has no chart (agent-sandbox manifests, Garage's in-repo chart). |
| D8 | quickstart isolation honesty | ship sandboxes on runc with a warning · refuse to enable Code Interpreter without gVisor | ship with a warning, off by default in `production` unless a runtime is detected. |

## Order of work

1. `charts/rubble` umbrella with the core services and the addon dependency list, ported
   from the reference instance's umbrella charts. Profiles as values files.
2. `rubble cluster check` and `init` (repo scaffolding, SOPS secrets, profile values).
3. `rubble cluster up` with `applier: local`; then `applier: argocd`.
4. gVisor node addon.
5. `get.rubble.cloud`, chart index, docs site (including the "serving models for Rubble" page).

Exit criterion for the quickstart, observed not assumed: a fresh 16 GB VM with k3s, no GPU,
runs `check → init → up` against a llama.cpp already running on the host and reaches a
working console, a chat in the playground, and a deployed first agent, in under 30 minutes wall-clock including image
pulls, with no step outside the four commands.
