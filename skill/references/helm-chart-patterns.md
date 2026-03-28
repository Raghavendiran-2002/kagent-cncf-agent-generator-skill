# Helm Chart Patterns Reference

This reference documents the exact patterns required for kagent agent helm charts.
All patterns are derived from the existing agents in `helm/agents/`.

---

## Directory Structure

```
helm/agents/<slug>/
├── Chart-template.yaml       ← processed by envsubst at build time
├── values.yaml               ← identical across all agents
└── templates/
    ├── agent.yaml            ← the Agent CRD manifest
    └── rbac.yaml             ← only when project introduces CRDs
```

---

## Chart-template.yaml

```yaml
apiVersion: v2
name: <slug>-agent
description: A <Project Name> Agent for kagent
type: application
version: ${VERSION}
```

**Critical:** `version` must always be the literal string `${VERSION}`. The `make helm-agents`
target runs `envsubst` to substitute the real version at build time:

```makefile
VERSION=$(VERSION) envsubst < helm/agents/<slug>/Chart-template.yaml > helm/agents/<slug>/Chart.yaml
helm package -d $(HELM_DIST_FOLDER) helm/agents/<slug>
```

Never use a real semver (e.g. `0.1.0`) — the chart will not be packaged correctly.

---

## values.yaml

This file is **identical** across every agent. Do not add, remove, or rename any key.

```yaml
modelConfigRef: ""
imagePullSecrets: []

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 1Gi
```

Exactly three top-level keys: `modelConfigRef`, `imagePullSecrets`, `resources`.

---

## templates/agent.yaml — Full Pattern

### Metadata block

```yaml
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: <slug>-agent
  namespace: {{ include "kagent.namespace" . }}
  labels:
    {{- include "kagent.labels" . | nindent 4 }}
```

### spec layout (order matters)

```yaml
spec:
  description: <One sentence here — NOT inside declarative>
  type: Declarative
  declarative:
    systemMessage: |
      ...
    promptTemplate:
      dataSources:
        - kind: ConfigMap
          name: kagent-builtin-prompts
          alias: builtin
    modelConfig: {{ .Values.modelConfigRef | default (printf "%s" (include "kagent.defaultModelConfigName" .)) }}
    tools:
    - type: McpServer
      mcpServer:
        name: kagent-tool-server
        kind: RemoteMCPServer
        apiGroup: kagent.dev
        toolNames:
        - <tool_name_alphabetical>
    a2aConfig:
      skills:
        - id: ...
    deployment:
      {{- with coalesce (empty .Values.imagePullSecrets | ternary nil .Values.imagePullSecrets) .Values.global.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      resources:
        {{- toYaml .Values.resources | nindent 8 }}
```

**Order inside `declarative:`:**
1. `systemMessage`
2. `promptTemplate`
3. `modelConfig`
4. `tools`
5. `a2aConfig`
6. `deployment`

### Builtin includes (triple-backtick escaping)

These three must always be present in `systemMessage`, in this order:

```
{{ `{{include "builtin/kubernetes-context"}}` }}

{{ `{{include "builtin/tool-usage-best-practices"}}` }}

{{ `{{include "builtin/safety-guardrails"}}` }}
```

Rules:
- **No dot argument** — `{{include "builtin/kubernetes-context"}}` NOT `{{include "builtin/kubernetes-context" .}}`
- Wrapped in Go template literal escaping: `` {{ `{{include "..."}}` }} ``
- Placed inside the `## Operational Guidelines` section of systemMessage

### imagePullSecrets pattern

```yaml
{{- with coalesce (empty .Values.imagePullSecrets | ternary nil .Values.imagePullSecrets) .Values.global.imagePullSecrets }}
imagePullSecrets:
  {{- toYaml . | nindent 8 }}
{{- end }}
```

Do not simplify this. The `empty ... | ternary nil` part correctly handles an empty list `[]`
(which is truthy in Helm) vs a nil value.

### resources pattern

```yaml
resources:
  {{- toYaml .Values.resources | nindent 8 }}
```

Always `nindent 8` (not `nindent 6` or `indent`).

---

## templates/rbac.yaml — Full Pattern

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: {{ include "kagent.fullname" . }}-<slug>-role
  labels:
    {{- include "kagent.labels" . | nindent 4 }}
rules:
...
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: {{ include "kagent.fullname" . }}-<slug>-rolebinding
  labels:
    {{- include "kagent.labels" . | nindent 4 }}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: {{ include "kagent.fullname" . }}-<slug>-role
subjects:
  - kind: ServiceAccount
    name: {{ include "kagent.fullname" . }}
    namespace: {{ include "kagent.namespace" . }}
```

**Naming convention:**
- ClusterRole: `<fullname>-<slug>-role` — NOT `<slug>-agent-role`
- ClusterRoleBinding: `<fullname>-<slug>-rolebinding`
- Always separated by `---`

**When to create rbac.yaml:**
- Yes → project introduces custom CRDs (e.g., keda.sh, fluxcd.io, cert-manager.io)
- No → project only uses standard k8s resources (rare for CNCF projects)

---

## Helm Integration (Phase 4)

### helm/kagent/Chart-template.yaml — dependency entry

Append at end of `dependencies:` list:

```yaml
  - name: <slug>-agent
    version: ${VERSION}
    repository: file://../agents/<slug>
    condition: agents.<slug>-agent.enabled
```

### helm/kagent/values.yaml — agents block entry

Append at end of `agents:` block:

```yaml
  <slug>-agent:
    enabled: <true|false>
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: 1000m
        memory: 1Gi
```

### Makefile — helm-agents target

Append two tab-indented lines at end of `helm-agents:` target:

```makefile
	VERSION=$(VERSION) envsubst < helm/agents/<slug>/Chart-template.yaml > helm/agents/<slug>/Chart.yaml
	helm package -d $(HELM_DIST_FOLDER) helm/agents/<slug>
```

**Must use a real tab character.** Spaces will break the Makefile recipe.

---

## Helm Helper Reference

| Helper | Produces |
|--------|----------|
| `include "kagent.namespace" .` | Deployment namespace |
| `include "kagent.labels" .` | Standard kagent labels |
| `include "kagent.fullname" .` | Release fullname prefix |
| `include "kagent.defaultModelConfigName" .` | Default model config CRD name |

---

## Existing Agents (for cross-reference)

| Slug | API Groups | RBAC |
|------|-----------|------|
| `k8s` | core, apps, networking.k8s.io | No (standard k8s only) |
| `keda` | keda.sh, apps, autoscaling | Yes |
| `kyverno` | kyverno.io, wgpolicyk8s.io | Yes |
| `kgateway` | gateway.networking.k8s.io | Yes |
