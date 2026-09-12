# The model catalog

Status: design, 2026-09-12.

Rubble ships no model-serving layer. Any OpenAI-compatible endpoint is a model. That means
an endpoint URL is all the platform knows unless you tell it more, and a URL does not say
whether the model sees images, calls tools, returns JSON, reasons, or how much context it
has. The agent builder, the playground, the knowledge-base wizard and the SDK all need
those facts. So they live in **one file**, the model catalog, and everything reads from it.

This is Bedrock's model catalog and `ListFoundationModels`, in a file you own.

## Where it lives

`models.yaml` at the root of the instance repository. The CLI and the console edit it
through commits like everything else; the console's *Models* page is a view of it, with
live health from the gateway laid over the declared facts. From this one file the
platform renders:

- one `AgentgatewayModel` per entry (routing, aliases, failover, per-model policies);
- the catalog API, `GET /api/models`, that the console, CLI and SDK read;
- the OpenAI-shaped `GET /v1/models` list at the gateway for plain clients.

Per-tenant visibility is part of the entry, so a tenant only sees and can only route to
what it is allowed.

## Schema

```yaml
# models.yaml — the instance's model catalog
providers:
  local-gpu-a:                     # a name you choose; referenced by models below
    type: openai-compatible        # openai-compatible | openai | anthropic | bedrock | vertex | gemini
    baseURL: http://llama-chat.ai.svc:8000/v1
    auth: none                     # none | secretRef
  local-gpu-b:
    type: openai-compatible
    baseURL: http://llama-utility.ai.svc:8000/v1
  cloud-anthropic:
    type: anthropic
    auth:
      secretRef: { name: provider-anthropic, key: api_key }

models:
  - id: chat                       # what callers put in "model"; also the AgentgatewayModel name
    aliases: [llm-chat, qwen3.6-35b]
    provider: local-gpu-a
    upstreamModel: default         # the "model" value sent upstream, if the server cares
    description: >-
      Biggest local tier. Vision and video. Reasoning available but off by default.
    roles: [chat]                  # chat | utility | aux | embedding | rerank — what it is for
    capabilities:
      modalities:
        input: [text, image, video]
        output: [text]
      tools: true                  # function / tool calling
      structuredOutput: json_schema   # none | json_object | json_schema
      streaming: true
      reasoning:
        available: true
        default: off
        enable: { chat_template_kwargs: { enable_thinking: true } }   # how to switch it on, per request
        minOutputTokens: 800       # advice the SDK enforces when reasoning is on
      contextLength: 65536
      maxOutputTokens: 8192
      parallelSlots: 2             # optional: context is shared across slots on llama.cpp
    limits:
      imageMaxTokens: 2048         # optional, server-side guard the SDK should respect
    defaults:
      temperature: 0.7
    cost:                          # relative or absolute; used for budgets and the planner
      unit: per-1k-tokens
      input: 0
      output: 0
      class: local                 # local | cloud — budgets treat these differently
    latency:
      class: interactive           # interactive | batch
      coldStartSeconds: 3          # optional, e.g. a router that sleeps when idle
    visibility:
      tenants: ["*"]               # "*" or a list
    health:
      probe: /v1/models            # what the gateway checks
    notes: >-
      Free-text operator notes surfaced in the console: quirks, measured numbers,
      "do not use for X". The planner and the agent builder show this to humans.

  - id: embed
    aliases: [embedding]
    provider: local-gpu-b
    roles: [embedding]
    capabilities:
      embeddings:
        dimension: 1024
        maxInputTokens: 8192
    visibility: { tenants: ["*"] }

  - id: claude
    provider: cloud-anthropic
    upstreamModel: claude-sonnet-5
    roles: [chat]
    capabilities:
      modalities: { input: [text, image], output: [text] }
      tools: true
      structuredOutput: json_schema
      streaming: true
      contextLength: 200000
      maxOutputTokens: 64000
    cost: { unit: per-1k-tokens, input: 0.003, output: 0.015, class: cloud }
    visibility: { tenants: [acme] }

virtualModels:                     # optional: routing on top of concrete models
  - id: chat-ha
    strategy: failover             # failover | weighted
    targets: [chat, claude]
    visibility: { tenants: [acme] }
```

Everything under `capabilities` is **declared by you**, because the endpoint cannot be
asked. `health` is **observed** by the gateway. The console shows both and never confuses
them.

## Who reads what

| consumer | uses |
|---|---|
| gateway rendering | `provider`, `baseURL`, `auth`, `aliases`, `upstreamModel`, `virtualModels`, `visibility` → `AgentgatewayModel` resources and routing |
| agent builder (console) and `rubble create` | `roles`, `capabilities.tools`, `structuredOutput`, `modalities` — a tool-using agent cannot pick a model without `tools: true`; a vision KB cannot pick a model without `image` input; a KB cannot pick an embedding model of the wrong dimension |
| playground | `modalities` (show the image/video drop zone or not), `reasoning.enable` (the toggle), `contextLength`, `defaults` |
| SDK (`rubble-agents`) | `reasoning.enable` and `minOutputTokens` (sets them for you), `limits`, `contextLength` (conversation manager window), `structuredOutput` (picks the right mechanism) |
| policy and budgets | `cost.class`, `visibility` |
| knowledge-base wizard | `roles: [embedding]`, `embeddings.dimension` |
| `rubble models list` | all of it, as a table |

## What it replaces

In the reference instance this information was spread over a hand-kept access document,
comments in each tier's Deployment, a gateway values list, and people's heads. All of it
fits in this file, and the parts that were prose ("reasoning never terminates on the small
tier past triage size", "the router sleeps after 300 s and waking costs 3 s") go in
`notes`, `latency.coldStartSeconds` and `reasoning`, where the builder can act on them.

## Validation

`rubble models validate` (also run by the console before it commits): schema, every
`provider` referenced exists, no alias collides with an id, every embedding model declares
a dimension, every virtual model's targets exist and share the tenants that can see it.
Optionally `--probe` hits each endpoint and reports what answered.
