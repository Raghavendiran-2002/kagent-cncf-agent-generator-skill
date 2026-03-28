# System Message Authoring Guide

Guidelines for writing high-quality `systemMessage` content in generated `agent.yaml` files.
The system message is the most important part of the agent — it determines quality and usefulness.

---

## Required Structure

```
## Role
## Core Capabilities
## Available Tools
### Kubernetes Tools
### Helm Tools
### <Project> Tools   ← only if project-specific tools selected
## Operational Guidelines
## Best Practices
## Common Scenarios
```

All six sections are required. Do not rename, reorder, or omit any section.

---

## ## Role

One short paragraph (3–5 sentences). Must include:
1. Project name and what it does (one sentence from Phase 1 research)
2. Where it sits in the Kubernetes ecosystem
3. What the agent helps operators do

**Good example (KEDA):**
```
You are a KEDA (Kubernetes Event-Driven Autoscaling) expert AI agent. KEDA extends the
Kubernetes Horizontal Pod Autoscaler to scale workloads based on external event sources
such as Kafka topics, Redis queues, Azure Service Bus, AWS SQS, and Prometheus metrics.
You help operators install, configure, troubleshoot, and upgrade KEDA and its autoscaling
resources.
```

**Anti-patterns:**
- Generic: "You are an expert AI agent that helps with Kubernetes."
- Vague: "You help users manage their cluster."
- Missing project context: doesn't name what the project does

---

## ## Core Capabilities

4–8 bullet points. Each bullet:
- Bold label followed by colon and explanation
- Grounded in real operational tasks from Phase 1 research
- Specific to the project (not generic k8s operations)

```
- **ScaledObject Management**: Create, update, and inspect ScaledObject resources to
  control how Deployments and StatefulSets respond to external event-source load.
- **TriggerAuthentication**: Set up TriggerAuthentication and ClusterTriggerAuthentication
  to securely connect scalers to external systems via Kubernetes Secrets.
```

**Anti-patterns:**
- "Help users with Kubernetes resources" — too generic
- More than 8 bullets — pick the most important
- Capabilities not backed by actual tools selected

---

## ## Available Tools

One line per selected tool: tool name in backticks, followed by when to use it.
Group under sub-headers matching the tool categories.

```
### Kubernetes Tools
- `k8s_get_resources`: List ScaledObjects, ScaledJobs, TriggerAuthentications,
  Deployments, pods, and events. Always specify the exact resource kind.
- `k8s_describe_resource`: Get full status conditions and events for a specific
  resource. Essential for diagnosing ScaledObject failures.
```

Rules:
- Only list tools that are in the `toolNames:` array
- The description says *when* to use it, not just *what* it does
- Reference real resource kinds from the project (e.g., "ScaledObject" not "custom resources")

---

## ## Operational Guidelines

This section always contains exactly the three builtin includes, nothing else:

```
{{ `{{include "builtin/kubernetes-context"}}` }}

{{ `{{include "builtin/tool-usage-best-practices"}}` }}

{{ `{{include "builtin/safety-guardrails"}}` }}
```

Do not add custom content here. Do not remove any builtin.

---

## ## Best Practices

3–5 bullet points. Must be:
- **Project-specific** — not generic k8s advice
- **Grounded in real gotchas** from Phase 1 research (pain points, FAQs, operator mistakes)
- **Actionable** — tells the operator what to do, not just what to know

**Good examples (KEDA):**
```
- Always inspect `ScaledObject` status conditions first when scaling fails — the
  `Active`, `Ready`, and `Fallback` conditions reveal the root cause before you dig
  into logs.
- Never set `minReplicaCount: 0` on a production ScaledObject without understanding
  the current queue depth — this will terminate all pods when the source goes idle.
```

**Anti-patterns:**
- "Always follow Kubernetes best practices." — generic
- "Make sure your cluster is healthy." — not actionable
- Advice that applies to any k8s workload

---

## ## Common Scenarios

2–4 scenarios. Each scenario:
- Has a descriptive title (### heading)
- Lists numbered steps
- Uses real CRD kinds, CLI commands, and tool names from the agent
- Addresses one of the pain points identified in Phase 1

**Template:**
```
### <Scenario title matching a Phase 1 pain point>
1. `<tool_name>` on <specific resource> — <what to look for>.
2. `<tool_name>` for <follow-up check> — <what it tells you>.
3. `<tool_name>` to <action> — <outcome>.
```

**Good example (KEDA — Workload not scaling up):**
```
### Workload not scaling up
1. `k8s_describe_resource` on the ScaledObject — check `Active` condition and `LastActiveTime`.
2. `k8s_get_resources` for HPAs in the target namespace — KEDA creates an HPA; verify it exists.
3. `k8s_get_pod_logs` for the `keda-operator` pod in `keda` namespace — look for scaler errors.
4. `k8s_get_events` in the target namespace — surface scheduling or admission blockers.
```

---

## systemMessage YAML formatting

The `systemMessage` uses literal block scalar (`|`). All newlines are preserved.
Indentation inside the message is 6 spaces (4 for `declarative:` + 2 content indent).

```yaml
    systemMessage: |
      ## Role

      You are a ...

      ## Core Capabilities

      - **Capability**: ...
```

Common mistake: using `>` (folded scalar) instead of `|` — this collapses newlines
and breaks all formatting.

---

## Quality Bar Check

Before finalising systemMessage, verify:

- [ ] Role paragraph names the project and explains what the agent does
- [ ] Core Capabilities count is 4–8, each grounded in Phase 1 research
- [ ] Available Tools section lists only tools in `toolNames:`, with when-to-use descriptions
- [ ] All three builtins are present in Operational Guidelines with correct escaping
- [ ] Best Practices are project-specific (not generic k8s advice)
- [ ] Common Scenarios use real CRD kinds and tool names from this agent
- [ ] No trailing whitespace on lines inside `systemMessage`
- [ ] Block scalar is `|` not `>`
