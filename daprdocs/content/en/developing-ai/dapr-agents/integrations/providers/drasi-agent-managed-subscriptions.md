---
type: docs
title: 'Agent-managed Drasi subscriptions'
linkTitle: 'Agent-managed subscriptions'
weight: 11
description:
  'Let a durable agent select persistent monitoring from a Drasi query catalog'
---

{{% alert title="Preview" color="warning" %}}
This API is implemented on the `feature/gsoc-drasi-m2` development branch and is
not included in the published `dapr-agents-ext-drasi==0.1.0` package. The next
extension version and final minimum Dapr Agents version have not been selected.
Do not treat this page as released-package guidance yet.
{{% /alert %}}

Agent-managed subscriptions let a `DurableAgent` choose which operator-approved
Drasi query changes it should receive. Drasi performs deterministic change
detection, while Dapr Workflow provides durable agent execution after a matching
change.

The original agent task can finish while its subscriptions remain active. A
later Drasi delivery starts a new workflow with the saved handling instructions
and event data. It does not resume the original Python call stack or
conversation.

## Compare subscription modes

| Mode              | Subscription owner               | API                              | Use when                                                                         |
| ----------------- | -------------------------------- | -------------------------------- | -------------------------------------------------------------------------------- |
| Author-configured | Application author or deployment | `register_drasi_trigger()`       | Query IDs and task mapping are fixed before the agent is hosted.                 |
| Agent-managed     | Agent using generated tools      | `drasi_subscription_lifecycle()` | The agent selects persistent monitoring from an operator-curated router catalog. |

The two modes cannot be combined on the same logical agent.

## Review the feature implementation

The published extension does not contain this lifecycle yet. To review the
current implementation, use the feature checkout.

{{< tabpane text=true >}}

{{% tab header="Linux/macOS" %}}

```bash
git clone https://github.com/amansinghoriginal/dapr-agents.git
cd dapr-agents
git checkout feature/gsoc-drasi-m2
uv sync --group dev --group test --extra drasi \
  --config-settings-package dapr-agents:editable_mode=strict \
  --reinstall-package dapr-agents
```

{{% /tab %}}

{{% tab header="Windows" %}}

```powershell
git clone https://github.com/amansinghoriginal/dapr-agents.git
cd dapr-agents
git checkout feature/gsoc-drasi-m2
uv sync --group dev --group test --extra drasi `
  --config-settings-package dapr-agents:editable_mode=strict `
  --reinstall-package dapr-agents
```

{{% /tab %}}

{{< /tabpane >}}

The feature checkout uses the published `drasi-agent-router-contracts==0.1.0a1`
wire-contract package.

## Prerequisites

Before starting the lifecycle, configure:

- A fresh `DurableAgent` using the normal LLM and tool loop.
- An `AgentStateConfig.store` backed by a state component that supports ETags
  and strong reads.
- An `AgentPubSubConfig`, or an explicit lifecycle `pubsub` override.
- A deployed Drasi `DaprAgentRouter` with an operator-curated query catalog.
- Stable Dapr application and agent names.

The reference deployment uses one logical agent and one application replica. It
assumes a private, trusted environment.

## Host an agent-managed lifecycle

Use a FastAPI lifespan so preparation completes before application readiness is
exposed:

```python
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager

from fastapi import FastAPI

from dapr_agents import DurableAgent
from dapr_agents.agents.configs import AgentPubSubConfig, AgentStateConfig
from dapr_agents.ext.drasi import drasi_subscription_lifecycle
from dapr_agents.storage.daprstores.stateservice import StateStoreService
from dapr_agents.workflow.utils.core import wait_for_shutdown


agent = DurableAgent(
    name="CheckoutSRE",
    llm=...,       # Configure a supported chat client.
    tools=[...],   # Add the actions the agent may perform.
    pubsub=AgentPubSubConfig(
        pubsub_name="agent-pubsub",
        agent_topic="checkout-sre.requests",
        broadcast_topic="checkout-sre.broadcast",
    ),
    state=AgentStateConfig(
        store=StateStoreService(store_name="agent-state")
    ),
)


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    async with drasi_subscription_lifecycle(
        agent,
        router_id="drasi-system/sre-router-reaction",
        namespace="applications",
        app=app,
    ):
        yield


app = FastAPI(lifespan=lifespan)
```

With no `app` argument, the context starts workflow hosting and yields an
`AgentRunner` for direct calls. Retain the lifecycle object so unresolved
cleanup can be retried:

```python
lifecycle = drasi_subscription_lifecycle(
    agent,
    router_id="drasi-system/sre-router-reaction",
    namespace="applications",
)

async with lifecycle as runner:
    await runner.run(
        agent,
        {"task": "Monitor checkout errors and assess relevant changes."},
        wait=False,
    )
    await wait_for_shutdown()
```

## Lifecycle arguments

| Argument                  | Default           | Description                                                                                        |
| ------------------------- | ----------------- | -------------------------------------------------------------------------------------------------- |
| `router_id`               | Required          | Router identity in `<namespace>/<app-id>` form.                                                    |
| `namespace`               | Required          | Namespace of the subscriber application.                                                           |
| `app`                     | `None`            | If supplied, host with `AgentRunner.serve()` on the application-owned FastAPI app.                 |
| `workflow_client`         | `None`            | Optional caller-owned workflow client. The lifecycle does not close an injected client.            |
| `pubsub`                  | Agent message bus | Override the application-facing Pub/Sub component used by the derived inbox and dead-letter topic. |
| `dapr_http_port`          | `DAPR_HTTP_PORT`  | Override the local Dapr HTTP port used for router service invocation.                              |
| `router_timeout_seconds`  | `30.0`            | Deadline for each router MCP exchange.                                                             |
| `startup_timeout_seconds` | `30.0`            | Deadline for workflow-worker readiness.                                                            |

The lifecycle derives subscriber identity, workflow name, inbox topic, and
dead-letter topic from trusted application configuration. The model cannot
choose infrastructure identities or broker destinations.

## Startup and readiness

On entry, the lifecycle:

1. Validates the supported agent profile.
2. Loads the router catalog through Dapr service invocation.
3. Builds query-specific subscription tools.
4. Loads and reconciles durable subscription intent.
5. Starts normal agent hosting.
6. Waits for the workflow worker to become ready.
7. Opens the Drasi inbox last.

No startup model turn is performed. An empty catalog still provides
`list_drasi_subscriptions()`.

## Shutdown and recovery

Ordinary shutdown stops intake and runtime resources without deleting durable
intent or router rules. A fresh lifecycle reconciles that state after restart.

The lifecycle is one-shot. After failure or shutdown, create fresh agent and
lifecycle objects. If context exit reports unresolved cleanup, retain the
lifecycle object from the example above. After addressing the reported
dependency or shutdown problem, retry cleanup as a separate operation:

```python
# Run later from the application's shutdown supervisor.
await lifecycle.aclose()
```

`aclose()` retries cleanup only. It never rehosts the agent or unsubscribes
persistent monitoring.

## Supported restrictions

The preview lifecycle:

- Requires a fresh agent constructed without `runtime=`.
- Does not support `RuntimeSubscriptionConfig` on the same agent.
- Does not support pre-existing activation callbacks on the same agent.
- Does not support external `executor=` runtimes or orchestrator mode.
- Supports one active dynamic lifecycle per Dapr application.
- Does not mix author-configured and agent-managed Drasi modes on one agent.

These are integration restrictions, not general Dapr Agents framework rules.

## Delivery behavior

- The router stores delivery rules; the agent state store retains handling
  instructions and lifecycle status.
- The agent acknowledges a delivery after workflow scheduling is accepted, not
  after workflow completion.
- Delivery is at least once where the configured runtime and broker permit it.
- Duplicate workflows and repeated activities remain possible.
- External actions must enforce their own business idempotency.
- Retry exhaustion or dead-letter publication failure can still result in loss.
- There is no historical replay, permanent deduplication ledger, automatic
  query-recreation recovery, or scale-to-zero guarantee.

## Reference example

The proposed
[agent-managed Drasi subscriptions example](https://github.com/dapr/dapr-agents/tree/main/examples/ext-drasi-agent-managed-subscriptions-k8s)
demonstrates:

- Autonomous selection between two Drasi queries.
- Durable subscription intent and independent event workflows.
- Follow-up monitoring selected by an event workflow.
- An idempotent PostgreSQL service-assessment action.
- Inspection, unsubscribe, and ownership-checked cleanup.

Do not publish this page until the extension API and example are available from
the upstream Dapr Agents repository.
