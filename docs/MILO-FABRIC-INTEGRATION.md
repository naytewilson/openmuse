# Milo Fabric Integration Plan

Status: controller design, not yet merged
Source binding: `naytewilson/openmuse@fed01e9d6411ab773d9adf1aa490a07dc8c64d0b`
Reviewed upstream: `CopilotKit/openmuse@82ff35b912e4a08c435d29278631c95445adf658`
Upstream delta at review time: README-only documentation changes, no execution-path changes.

## Goal

Use OpenMuse as a product-shell and interaction-pattern donor for Milo without creating a second independent brain, a second durable-work authority, or a broad credential path.

The intended control stack is:

```text
Milo client / OpenMuse-derived surfaces
              |
              v
      Milo agent gateway
              |
      +-------+--------+
      |                |
conversation       durable work
      |                |
      +-------+--------+
              |
      capability boundary
              |
    Paseo / SIEVE / ANVIL
              |
 local + cloud execution providers
```

OpenMuse may continue to own product state during the first convergence phase: tasks, plans, approvals, artifacts, browser presentation, files, goals, notifications, and SQL lease/checkpoint mechanics.

Milo owns routing policy, model selection, escalation, evidence policy, and the decision about which bounded capability should execute a requested operation.

## Current source truth

OpenMuse currently has three conversational backends:

- `sample`
- `model`
- `agui`

The `model` path uses CopilotKit `BuiltInAgent`.

There are two separate model loops in the server:

1. conversational execution in `apps/server/src/engine/conversation.ts`
2. durable delegated-task execution in `apps/server/src/engine/model.ts`

The durable task worker has the important machinery that should not be casually duplicated:

- SQL-backed task ownership
- leases and heartbeat
- checkpointed state
- ordered tool execution
- operation-result caching
- cancellation and lost-lease handling
- stored approvals
- fail-closed external writes
- saved artifacts and run events

### Critical integration finding

`AGENT_BACKEND=agui` replaces conversational routing only.

It does **not** replace durable open-ended task execution.

With the stock configuration it is therefore possible to create a split-brain deployment:

```text
chat                -> external AG-UI agent
delegated task      -> OpenMuse BuiltInAgent(MODEL)
```

That is not an acceptable final Milo architecture.

## Phase 1: one model gateway, no split brain

The smallest immediately viable convergence does not require replacing OpenMuse's tool loop.

Run OpenMuse in its normal model mode and point the OpenAI-compatible provider path at the Milo gateway:

```env
AGENT_BACKEND=model
MODEL=openai/milo
OPENAI_BASE_URL=http://<milo-gateway>/v1
```

The current model-worker test fixture proves the OpenAI-compatible path uses the Responses API endpoint:

```text
/v1/responses
```

Under this phase:

```text
OpenMuse ConversationAgent ----+
                               |
OpenMuse durable task agent ---+--> Milo Responses gateway
                                      |
                                      +--> local model
                                      +--> Luna / Sol
                                      +--> coding model
                                      +--> research model
                                      +--> escalation policy
```

This gives both OpenMuse agent loops the same model-routing authority and avoids the AG-UI chat-only split.

### What Milo owns in Phase 1

- model selection
- effort selection
- provider selection
- cost / latency policy
- evidence escalation
- local-versus-cloud model routing
- fallback and retry policy at the model boundary

### What OpenMuse still owns in Phase 1

- the agent tool loop
- durable task leases
- task checkpoints
- approvals
- tool execution
- browser and computer adapters
- Google/file credentials
- artifacts and task presentation

This is deliberately asymmetric. It creates one routing brain before attempting to move execution ownership.

## Phase 2: explicit execution backend

After Phase 1 is stable, introduce a single execution abstraction used by both conversation and durable tasks.

Target shape:

```ts
interface AgentExecutionBackend {
  runConversation(input: ConversationRun): AsyncIterable<AgentEvent>;
  runDurableTask(input: DurableTaskRun): AsyncIterable<AgentEvent>;
}
```

Backends can then be:

- `builtin` — current CopilotKit BuiltInAgent behavior
- `milo` — Milo agent fabric
- future test/fixture backends

Do not implement a durable Milo backend as a raw `HttpAgent` alias. A durable run needs task identity, lease identity, checkpoint semantics, cancellation, tool-result correlation, and an explicit completion outcome.

### Durable-run contract

A Milo durable execution request must bind at least:

- owner identity
- OpenMuse task id
- run id
- lease id or lease epoch
- task prompt
- prior durable state
- evidence ids
- artifact ids
- available tool descriptors
- cancellation signal identity
- provenance/source binding

A tool request from Milo must be treated as a request to the OpenMuse server, not as authority by itself.

OpenMuse validates and executes the bounded capability, persists the result, and sends the result back to Milo.

Milo never receives raw Google credentials, filesystem host authority, browser-worker credentials, or approval authority.

## Durability ownership

There must be one durable-work owner at a time.

### Initial convergence

OpenMuse owns:

- task row
- lease
- heartbeat
- checkpoint
- action proposal
- retry state
- final status

Milo supplies reasoning and routing.

This is the preferred first implementation because OpenMuse already has tested durable-task semantics and it avoids a dual-ledger recovery problem.

### Possible later convergence

If Milo eventually becomes the durable-work owner, migrate the entire lifecycle coherently:

- task identity
- lease / generation
- checkpoint log
- approvals
- recovery
- replay
- notification publication

Do not leave half the lifecycle in OpenMuse and half in Milo.

## Capability boundary

All external execution must preserve the Milo / ANVIL architecture rules:

1. Reachability is not permission.
2. Identity is not a credential.
3. A credential is not authority.
4. Authority is explicit and narrow.
5. Source truth is bound before mutation.
6. External writes are fail-closed.
7. Approval is a separate capability from proposal.
8. Retries of ambiguous writes are forbidden.
9. Every durable mutation must be replayable or recoverable.
10. Receipts must bind the actor, source state, operation, and result.

A mobile client must never call machine-control ports directly.

The server/gateway remains the capability broker.

## Deterministic work remains deterministic

Do not route every OpenMuse task through a model.

The existing deterministic paths are useful architecture:

- document workflow
- monitor checks
- finance analysis
- other bounded transforms that can be proven without open-ended reasoning

Milo should preserve this property.

A routing decision of `no model required` is a first-class success case.

## Suggested Milo routing policy

The gateway may route by task class:

| Task class | Default lane |
| --- | --- |
| trivial transformation | deterministic/local |
| ordinary conversation | smallest capable model |
| software implementation | coding-agent lane |
| repository mutation | source-truth + coding lane |
| research | retrieval/research lane |
| local machine action | Paseo capability |
| high uncertainty | evidence-directed controller |
| expensive frontier reasoning | explicit escalation |
| irreversible external write | prepare + user approval |

This routing belongs in Milo, not in the UI shell.

## OpenMuse surfaces worth retaining or reimplementing

High-value product concepts:

- durable activity timeline
- plans and checkpoints
- pause/resume/cancel/retry
- user-input requests
- explicit approvals
- artifact/result cards
- persistent browser session with takeover
- files and PDF workflows
- goals and monitors
- memories/personal context
- notification inbox
- follow-up queue
- background-work presentation

These are product patterns, not an argument that Milo should inherit every OpenMuse runtime dependency.

## Conduit relationship

Conduit and OpenMuse occupy different layers.

Conduit remains valuable as a Hermes-native compatibility client and as native Swift source material for:

- WebSocket/session handling
- voice
- attachments
- approvals
- Keychain/Face ID
- push
- native iOS presentation

OpenMuse is more relevant to Milo's personal-agent product architecture.

The likely long-term Apple-quality path is:

```text
OpenMuse product concepts
        +
Conduit native components
        |
        v
native Milo Swift / SwiftUI client
        |
        v
Milo capability gateway
        |
        v
Paseo + SIEVE + ANVIL + Hermes + MCP
```

## What not to do

- Do not treat `AGENT_BACKEND=agui` as a complete Milo integration.
- Do not create a second independent durable task database in Milo during Phase 1.
- Do not expose OpenMuse browser/computer worker ports to the mobile app.
- Do not send provider, Google, machine, or GitHub credentials to an external reasoning service.
- Do not give a model an `approve` tool.
- Do not retry an ambiguous write automatically.
- Do not make ANVIL's source-truth authority reachable through a generic unrestricted shell.
- Do not merge upstream changes into the Milo fork without reviewing execution-path diffs.
- Do not rewrite the app in Swift before the execution contract is stable enough to preserve behavior.

## Acceptance gates

### Phase 1

- Conversation and durable model calls both reach the Milo Responses gateway.
- No direct provider call bypasses that gateway in the configured Milo mode.
- Existing durable-task tests still pass.
- Approval behavior remains unchanged.
- Deterministic workflows remain model-free.
- A provider outage fails closed and leaves a resumable task.
- Request logs prove no credentials are forwarded beyond the intended model request.

### Phase 2

- One backend interface serves both conversation and durable execution.
- A Milo durable run is bound to task id + lease identity.
- Lost lease cancels the external run.
- Duplicate tool requests are idempotent or rejected.
- Tool execution remains server-authorized.
- Ambiguous writes are never replayed.
- Paused/waiting/completed outcomes map exactly to the persisted OpenMuse task state.
- Crash/restart resumes from the persisted checkpoint without duplicating completed operations.
- Built-in backend remains available as a rollback path until Milo acceptance is complete.

## Immediate next implementation wave

1. Add an explicit fail-closed configuration guard so an external AG-UI chat backend cannot silently coexist with the built-in durable model backend unless that split is deliberately enabled.
2. Add a Milo deployment profile/documented example using the single Responses gateway.
3. Add a provider-contract test proving both conversation and durable paths hit the same configured endpoint.
4. Only after that proof, introduce the shared `AgentExecutionBackend` seam.
5. Keep OpenMuse task durability as the sole task owner during the first external-execution implementation.
