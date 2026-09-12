# Rubble

**We have Bedrock at home.**

Rubble is a recipe, plus the glue code, for running the Amazon Bedrock / AgentCore
experience on your own Kubernetes cluster with open-source parts and your own models.
Same shape as the original: one model endpoint, an agent runtime, a tool gateway, memory,
knowledge bases, guardrails, workload identity, observability, a registry, a console, a
CLI, and a zero-config SDK built on [Strands Agents](https://strandsagents.com). No AWS
account, no per-token bill, your data stays on your disks.

> **Status (September 2026): planning.** Nothing to install yet. The service map below is
> settled; the recipe and the first code (SDK + CLI) are being written. Follow the repo if
> you want to run it, open an issue if you want to shape it.

## Who it is for

Small teams and businesses that want what Bedrock gives them — one model endpoint, agents
with tools, knowledge bases, memory, guardrails, tenants, an audit trail — on hardware they
own, without an AWS account. And homelabs, which is where it was built: the reference
instance is a multi-node cluster with several GPUs. It also runs on a single-node kind
cluster, so you can try it on a laptop before you commit a rack.

## What "Bedrock at home" means

Bedrock is not one thing. It is a set of managed services that happen to compose well.
Almost every one of them has a mature open-source counterpart; what is missing is the
composition, the defaults, and the convenience layer on top. Rubble is that layer.

| Bedrock / AgentCore | what it does | the Rubble ingredient | provided by |
|---|---|---|---|
| Bedrock Runtime (`Converse`, `InvokeModel`) | one endpoint, many models, streaming | one OpenAI-compatible endpoint in front of your model servers, routed by the `model` field | [agentgateway](https://agentgateway.dev) |
| Model catalog, aliases, failover | discoverable model IDs, weighted routing | `AgentgatewayModel` resources, virtual models | agentgateway |
| Custom model import | bring a model file, get an endpoint | model files in S3 → a Deployment per model (llama.cpp reference) | recipe |
| Guardrails | filter prompts and responses | regex, webhook and moderation guards at the gateway; Rubble ships a webhook classifier | agentgateway + Rubble |
| Knowledge Bases | S3 → chunks → embeddings → vector store → `retrieve` | bucket-per-KB ingestion job, Qdrant collection, `kb` MCP server | Rubble |
| **AgentCore Runtime** | host agent containers, sessions, `/invocations` + `/ping`, A2A | a base image and a Deployment template; sessions persisted to S3 | Rubble + Strands |
| **AgentCore Gateway** | tools as MCP, A2A passthrough, inbound auth, tool policy | agentgateway: MCP federation, A2A backends, tool authorization | agentgateway |
| **AgentCore Memory** | short-term sessions, long-term extracted records | Strands sessions on S3; a memory service with extraction strategies (facts, summaries, preferences) | Strands + Rubble |
| **AgentCore Identity** | workload identity, inbound OAuth, credential vault | SPIFFE/SPIRE workload identity, a bundled pre-configured Keycloak as identity broker, API keys | SPIRE + Keycloak + [spiffe-ext-auth](https://github.com/janfuu/spiffe-ext-auth) + Rubble |
| **AgentCore Observability** | traces per session, gen-ai span attributes | OpenTelemetry from gateway and SDK → Tempo, Grafana dashboards | OTel + Grafana stack |
| **AgentCore Policy** | deterministic tool-call rules | a Cedar policy server on agentgateway's MCP guardrail hook; `.cedar` files in git per tenant | Rubble + [cedar-go](https://github.com/cedar-policy/cedar-go) |
| **AgentCore Registry** | catalog of agents, MCP servers, skills, prompts | [agentregistry](https://github.com/agentregistry-dev/agentregistry) (catalog only, no deploy controller) | agentregistry |
| Console, `agentcore` CLI, SDK defaults | the convenience | the Rubble console, the `rubble` CLI, `rubble-agents` | Rubble |
| **AgentCore Code Interpreter** | sandboxed code execution per session, files in and out | a `code_interpreter` tool backed by the Kubernetes `Sandbox` API, gVisor or Kata isolation, warm pools | [agent-sandbox](https://github.com/kubernetes-sigs/agent-sandbox) + Rubble |
| Browser, Payments | — | not in scope for v1 | — |

## What you bring

Rubble runs anywhere these exist. The reference instance is a homelab: a small k3s
cluster, two AMD APUs and one CUDA box running llama.cpp, an S3 store on a NAS.

- **Kubernetes** — any distribution; developed on k3s. 1.37+ for scale-to-zero; gVisor on the workers for the code interpreter; a KVM node only if you want Kata microVMs.
- **A model server** with an OpenAI-compatible API — [llama.cpp server](https://github.com/ggml-org/llama.cpp) is the reference; vLLM and Ollama work too.
- **S3-compatible object storage** — Garage, MinIO, SeaweedFS, or real S3.
- **Identity** — Rubble ships a pre-configured Keycloak (realm, organizations, the SPIFFE
  exchange, clients for console, CLI and agents). Bring your own IdP if you have one; it
  plugs into that Keycloak as an identity provider, the way AgentCore Identity federates.
- **A git host and ArgoCD** — deploying an agent means committing manifests; ArgoCD applies them.
- Optional but part of the full recipe: **SPIRE** for workload identity, **Qdrant** for vectors, **Postgres** (CloudNativePG) for memory events, **Prometheus + Grafana + Tempo** for observability.

## How it fits together

```
        your clients: OpenAI SDKs, Strands, curl, the rubble CLI, the console
                              │
                        agentgateway  ── auth (API key or workload identity)
                              │          policies: guardrails, tool RBAC, rate limits, tracing
       ┌──────────────────────┼──────────────────────────┐
  models (/v1)           tools (/mcp)                 agents (/agents/<name>)
  llama.cpp etc.         MCP servers, OpenAPI→MCP     one Deployment per agent
                         kb · memory · registry       Strands + rubble-agents
                                                      sessions → S3
  S3: models/ · sessions/ · kb-*/        Qdrant: kb_*, memory_*        Postgres: memory events
  Registry: agentregistry                Observability: OTel → Tempo/Prometheus → Grafana
```

## The first 15 minutes (target experience)

```bash
pip install rubble-agents
rubble init                     # writes ~/.rubble/config: gateway URL, API key, S3, git remote
rubble create my-agent --template tool-agent
rubble dev                      # run it locally against your gateway
rubble deploy                   # build image, commit manifests to your GitOps repo, wait for sync
rubble invoke my-agent "what is on my task list this week?"
rubble logs my-agent
```

The agent itself:

```python
from rubble import Agent, tools, serve

agent = Agent(
    name="my-agent",
    system_prompt="You help with the task list. Be brief.",
    tools=tools.from_gateway("tasks", "calendar"),   # MCP tools behind the gateway
)

serve(agent)   # A2A + /invocations + /ping, sessions on S3, tracing on — all from config
```

No URLs, no keys, no socket paths in the code. Inside the cluster the same code
authenticates with its workload identity; on your laptop it uses the key from
`~/.rubble/config`.

## Principles

- **Git is the deploy API, and the audit trail.** The CLI and the console produce commits;
  ArgoCD applies them. Every agent that ever ran is a commit with an author, a diff and a
  timestamp. No agent is created from a runtime API call.
- **Agents are workloads with no cluster credentials.** A plain Deployment with a workload
  identity and no service-account token. Whatever an agent can cause in the cluster — a
  code sandbox, say — it asks a Rubble service for, and that service enforces tenant, quota
  and policy. Runtime-created things (sandboxes, scaled replicas) come from templates in git;
  the model can change how many run, never what runs.
- **Open protocols only.** OpenAI API, MCP, A2A, OpenTelemetry. Rubble owns none of them and
  can be taken apart along those seams.
- **Nothing AWS-specific is required.** Strands runs against any OpenAI-compatible model.
  A wire-compatible `bedrock-runtime` shim is planned for people who want unmodified
  `boto3` clients to work.
- **Convenience is a deliverable.** The console, the CLI, the templates and the quickstart
  are in scope, not a someday. A phase is done when a newcomer follows the docs cold and
  has an agent answering in 15 minutes.

## Showcase use-cases

1. **Ask the paperwork** — a knowledge base over a bucket of manuals, invoices and docs;
   answers with citations; PII masked by a guardrail.
2. **Incident → pull request** — an alert invokes an SRE agent that reads metrics, logs and
   runbooks and opens a PR with the fix. It may read everything and write only a PR.
3. **Research desk** — three agents cooperating over A2A, one trace across all of them,
   the result as a draft document.
4. **Weekly ops brief** — a scheduled agent that learns operator preferences across weeks
   through long-term memory.

## Roadmap

| phase | delivers |
|---|---|
| 0 | gateway model routing, tracing backend, GitOps bootstrap |
| 1 | runtime base image, first agent, sessions on S3, **`rubble-agents` SDK** |
| 1.5 | **`rubble` CLI**, templates, the 15-minute quickstart |
| 2 | registry as code |
| 3 | knowledge bases, **console** (playground, agents, KBs, keys) |
| 4 | long-term memory service |
| 5 | guardrails, tool policy, budgets |
| 6 | optional Bedrock wire-compat shim |

## Planned layout

```
sdk/         rubble-agents: SDK + CLI
console/     the Rubble console
runtime/     base image
templates/   agent / kb / mcp-server scaffolds and manifest templates
charts/      Helm chart: memory, kb-ingest, guardrail webhook, policy (Cedar), console, configured Keycloak
mcp/         generic MCP servers: kb, observability, argocd
shim/        Bedrock wire-compat (optional)
docs/        the recipe
```

## Multi-tenant from the start

A tenant is a Keycloak Organization. Each tenant gets its own Kubernetes namespace, its own
S3 bucket and key, its own vector collections, its own budgets and its own policies, and can
only ever deploy into its own namespace. Agents carry the tenant in their workload identity.
This is designed in from the first commit because retrofitting it is the one change that
touches everything.

## Policy is Cedar

Tool-call policy is written in [Cedar](https://www.cedarpolicy.com) — the same open-source
language AgentCore Policy uses — and enforced by a policy server that agentgateway consults
on every `tools/call` and `tools/list`. An agent does not see a tool it may not call.
Policies are files in git, per tenant.

## Honest differences from Bedrock

Code execution runs in gVisor sandboxes out of the box; microVM isolation (Kata) needs a
KVM-capable node and is opt-in per tenant. Scale-to-zero uses the
Kubernetes 1.37 HPA feature and the first request after idle pays a cold start; a request
buffer (KEDA HTTP add-on) is optional. Multi-tenancy is organisations inside one identity
provider, not separate accounts.

## License

Apache-2.0 (proposed).
