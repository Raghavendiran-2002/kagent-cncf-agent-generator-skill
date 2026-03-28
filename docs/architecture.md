# Architecture: cloud-native-agent-generator

The skill drives a four-phase workflow that adapts entirely to the CNCF project you point it at. It has no built-in knowledge of any specific project — only a repeatable process.

---

## Overview

```
User invokes skill with three arguments:
  /cncf-agent-generator <docs-url> <github-url> <needs-rbac>

        │
        ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│   Phase 1     │───▶│   Phase 2     │───▶│   Phase 3     │───▶│   Phase 4     │
│   Research    │    │   Design      │    │   Generate    │    │   Integrate   │
└───────────────┘    └───────────────┘    └───────────────┘    └───────────────┘
  Fetch live docs      Select tools         Write 4 files        Wire into build
  Extract CRDs         Design RBAC          No placeholders      pipeline
  Find pain points     Plan A2A skills      No truncation
```

---

## Phase 1 — Research

The skill fetches the project's documentation URL and GitHub repository using `WebFetch`. It reads:

- Installation and operator guides
- CRD reference documentation
- Troubleshooting pages

From these it extracts:

| Extracted | Used For |
|-----------|----------|
| CRD kinds | System message, RBAC rules |
| API groups | ClusterRole `apiGroups` fields |
| RBAC surface | Least-privilege access design |
| Pain points | System message operational guidelines |
| Common workflows | A2A skill design |

This is what makes the system message accurate. Training data alone would miss recent API changes and project-specific nuance.

---

## Phase 2 — Design

With research complete, the skill:

1. **Selects tools** from the kagent-tool-server catalogue — minimum set only, no invented names. A 20-item quality checklist rejects any tool not in the known catalogue.

2. **Designs RBAC** — if `needs-rbac=true`, plans a `ClusterRole` covering all discovered API groups with least-privilege verbs. If `needs-rbac=false`, skips the rbac.yaml entirely.

3. **Plans A2A skills** — 4–5 skills with 3–5 realistic user utterances each. Skills map directly to the project's core operational workflows (e.g., deploy, author policy, audit compliance, troubleshoot).

---

## Phase 3 — File Generation

Four files are written in full — no truncation, no `# TODO` placeholders:

### `Chart-template.yaml`

```yaml
apiVersion: v2
name: <project>-agent
description: kagent agent for <Project>
type: application
version: ${VERSION}          # literal — envsubst replaces this at build time
appVersion: ${VERSION}
```

### `values.yaml`

```yaml
<project>-agent:
  modelConfigRef: ""
  imagePullSecrets: []
  resources:
    requests:
      memory: "128Mi"
      cpu: "100m"
    limits:
      memory: "256Mi"
      cpu: "200m"
```

### `templates/agent.yaml`

The Agent CRD with:
- `systemMessage` grounded in Phase 1 research
- `tools` list drawn from the kagent-tool-server catalogue
- `a2aConfig.skills` — 4–5 skills, each with `name`, `description`, `tags`, and `examples`

### `templates/rbac.yaml` (when needed)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: <project>-agent
rules:
  # One entry per API group discovered in Phase 1
  - apiGroups: ["<project>.io"]
    resources: ["<crd-kinds>"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
...
```

---

## Phase 4 — Helm Integration

Three files in the main kagent chart are patched automatically:

**`helm/kagent/Chart-template.yaml`** — new subchart dependency:
```yaml
- name: <project>-agent
  version: ${VERSION}
  repository: file://../agents/<project>
  condition: agents.<project>-agent.enabled
```

**`helm/kagent/values.yaml`** — new agent toggle:
```yaml
<project>-agent:
  enabled: true
  modelConfigRef: ""
```

**`Makefile`** — new build targets:
```makefile
envsubst < helm/agents/<project>/Chart-template.yaml > helm/agents/<project>/Chart.yaml
helm package helm/agents/<project> --destination helm/charts/
```

Without these three edits, `make helm-agents` silently skips the new agent.

---

## Quality Checklist (20 items)

Before completing, the skill verifies:

- [ ] `${VERSION}` literal present in `Chart-template.yaml` (not a resolved version number)
- [ ] All YAML files pass indentation check
- [ ] No invented tool names — every tool exists in the kagent-tool-server catalogue
- [ ] 3–5 A2A skills present
- [ ] Each skill has 3–5 example utterances
- [ ] `rbac.yaml` present when `needs-rbac=true`
- [ ] ClusterRoleBinding references the correct ClusterRole
- [ ] All discovered API groups present in the ClusterRole
- [ ] All four Helm integration points updated (Chart-template.yaml, values.yaml x2, Makefile)
- [ ] System message references actual CRD kinds from Phase 1
- [ ] No placeholder text (`TODO`, `FIXME`, `...`) in any generated file

---

## A2A Protocol Integration

Every generated agent exposes its skills via the kagent Agent-to-Agent (A2A) protocol. This makes agents composable — an orchestrator agent can delegate to the KEDA agent for scaling configuration and the Kyverno agent for policy compliance in the same workflow, without any additional wiring.

---

## Security Considerations

- RBAC uses least-privilege: only the API groups and verbs the project actually requires
- `imagePullSecrets` is configurable in `values.yaml`
- No secrets or credentials are embedded in generated files
- All generated YAML is namespaced or uses cluster-scoped resources explicitly
