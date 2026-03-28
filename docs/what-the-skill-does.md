# What the Skill Does — Files Created and Modified

When you run `/cncf-agent-generator <docs-url> <github-url> <needs-rbac>`, the skill touches exactly **seven** things: four new files it creates from scratch, and three existing files it patches. Here is every change, explained.

---

## Example invocation used in this document

```
/cncf-agent-generator https://keda.sh https://github.com/kedacore/keda true
```

---

## New Files Created (4)

All four files are placed inside a new directory: `helm/agents/<slug>/`

### 1. `helm/agents/keda/Chart-template.yaml`

The Helm chart descriptor for the agent. The version field contains a **literal `${VERSION}`** — not a resolved version number. `make helm-agents` runs `envsubst` on this file to produce `Chart.yaml` before packaging.

```yaml
apiVersion: v2
name: keda-agent
description: kagent agent for KEDA — event-driven autoscaling for Kubernetes
type: application
version: ${VERSION}
appVersion: ${VERSION}
```

**Why `${VERSION}` and not a real version?**
The kagent build pipeline uses `envsubst` to stamp the version from the git tag at build time. Any hardcoded version here would be wrong on the next release.

---

### 2. `helm/agents/keda/values.yaml`

Default configuration values. The structure is identical for every generated agent — only the top-level key changes.

```yaml
keda-agent:
  modelConfigRef: ""          # filled in during helm install with your ModelConfig name
  imagePullSecrets: []
  resources:
    requests:
      memory: "128Mi"
      cpu: "100m"
    limits:
      memory: "1Gi"
      cpu: "1000m"
```

---

### 3. `helm/agents/keda/templates/agent.yaml`

The Agent Custom Resource. This is the heart of the generated output. It contains three significant parts:

#### a) System Message

Written directly from Phase 1 research — not generic Kubernetes advice. For KEDA it covers:

- What KEDA is and where it fits in the ecosystem
- All four CRD kinds: `ScaledObject`, `ScaledJob`, `TriggerAuthentication`, `ClusterTriggerAuthentication`
- Core capabilities: zero-scale management, trigger authentication, ScaledJob batch processing, metrics diagnostics
- The specific pain points discovered in research: zero-scale timing, cooldown misconfiguration, metrics API connectivity failures
- Builtin safety guidelines via `{{- include "kagent.builtins" . }}`

#### b) Tool List

Selected from the real kagent-tool-server catalogue. No invented names. For KEDA:

```yaml
toolNames:
  - helm_get_release
  - helm_list_releases
  - helm_repo_add
  - helm_repo_update
  - helm_upgrade
  - k8s_apply_manifest
  - k8s_create_resource
  - k8s_delete_resource
  - k8s_describe_resource
  - k8s_get_events
  - k8s_get_pod_logs
  - k8s_get_resources
  - k8s_patch_resource
```

Tools are sorted alphabetically (a kagent convention enforced by the skill's quality checklist).

#### c) A2A Skills (Agent-to-Agent Protocol)

Four skills with realistic utterances that map to real operational workflows:

```yaml
a2aConfig:
  skills:
    - name: install-keda
      description: Deploy KEDA into a Kubernetes cluster via Helm
      tags: ["keda", "install", "helm"]
      examples:
        - "deploy keda in the cluster"
        - "install keda into the keda namespace"
        - "set up keda with 2 replicas for HA"
        - "install the latest stable version of keda"

    - name: manage-scaledobject
      description: Create, update, and troubleshoot ScaledObjects for workload autoscaling
      tags: ["keda", "scaledobject", "autoscaling"]
      examples:
        - "create a scaledobject for my deployment using a prometheus trigger"
        - "scale my deployment to zero when queue depth is below 5"
        - "show me all scaledobjects in the production namespace"
        - "why is my scaledobject not scaling?"

    - name: configure-trigger-auth
      description: Set up TriggerAuthentication and ClusterTriggerAuthentication resources
      tags: ["keda", "authentication", "triggers"]
      examples:
        - "create a triggerauthentication for azure service bus"
        - "set up cluster-scoped trigger auth using a kubernetes secret"

    - name: troubleshoot-keda
      description: Diagnose KEDA issues including metrics API failures and scaling problems
      tags: ["keda", "troubleshoot", "debug"]
      examples:
        - "keda is not scaling my deployment"
        - "the metrics api server pod is crashlooping"
        - "my scaledobject shows paused but i didn't pause it"
        - "how do i debug a failing prometheus scaler"
```

---

### 4. `helm/agents/keda/templates/rbac.yaml` (only when `needs-rbac=true`)

A `ClusterRole` and `ClusterRoleBinding` using least-privilege access. For KEDA, which has its own CRD API group (`keda.sh`):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: {{ include "kagent.fullname" . }}-keda-role
  labels:
    {{- include "kagent.labels" . | nindent 4 }}
rules:
  # KEDA custom resources
  - apiGroups: ["keda.sh"]
    resources:
      - scaledobjects
      - scaledjobs
      - triggerauthentications
      - clustertriggerauthentications
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # Read workloads to understand what's being scaled
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "replicasets"]
    verbs: ["get", "list", "watch"]
  # Read HPA objects (KEDA creates these under the hood)
  - apiGroups: ["autoscaling"]
    resources: ["horizontalpodautoscalers"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: {{ include "kagent.fullname" . }}-keda-rolebinding
  labels:
    {{- include "kagent.labels" . | nindent 4 }}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: {{ include "kagent.fullname" . }}-keda-role
subjects:
  - kind: ServiceAccount
    name: {{ include "kagent.fullname" . }}
    namespace: {{ include "kagent.namespace" . }}
```

**Why ClusterRole and not Role?**
KEDA CRDs are cluster-scoped (`ClusterTriggerAuthentication`) and namespace-scoped resources may exist in any namespace. A namespaced Role would prevent the agent from seeing KEDA workloads outside its own namespace.

---

## Existing Files Patched (3)

These files already exist in the kagent repo. The skill appends new entries to each one.

### 5. `helm/kagent/Chart-template.yaml`

**What it is:** The main kagent umbrella chart's dependency list. Every agent is a Helm subchart listed here.

**What the skill adds** (appended to the `dependencies:` list):

```yaml
- name: keda-agent
  version: ${VERSION}
  repository: file://../agents/keda
  condition: agents.keda-agent.enabled
```

**Before the skill runs**, this file has entries like `k8s-agent`, `istio-agent`, etc. The KEDA entry is missing.

**After the skill runs**, the new dependency is present. This tells `helm dependency update` to include the KEDA subchart when building the umbrella chart.

**What breaks without this:** `helm dependency update` won't include the agent, so the agent pod is never deployed even if `agents.keda-agent.enabled: true` is set.

---

### 6. `helm/kagent/values.yaml`

**What it is:** Default values for the umbrella chart, including the enable/disable toggle for each agent.

**What the skill adds** (appended to the `agents:` block):

```yaml
keda-agent:
  enabled: true
  modelConfigRef: ""
```

**Before the skill runs**, there is no `keda-agent` key. The `condition: agents.keda-agent.enabled` in Chart-template.yaml would evaluate to false (missing key = disabled).

**After the skill runs**, the agent is enabled by default. A user can override this in their local values file: `agents.keda-agent.enabled: false`.

**What breaks without this:** The subchart condition resolves to false. The agent is packaged but never rendered or deployed.

---

### 7. `Makefile`

**What it is:** The project's build orchestration. The `helm-agents` target packages all agent subcharts.

**What the skill adds** (two lines appended inside the `helm-agents` target):

```makefile
	envsubst < helm/agents/keda/Chart-template.yaml > helm/agents/keda/Chart.yaml
	helm package helm/agents/keda --destination helm/charts/
```

**Before the skill runs**, running `make helm-agents` processes all other agents but silently skips KEDA because there are no lines for it.

**After the skill runs**, `make helm-agents` produces `helm/charts/keda-agent-<version>.tgz` which is referenced by the umbrella chart's dependency.

**What breaks without this:** The `helm/agents/keda/Chart.yaml` file is never generated (it's derived from `Chart-template.yaml` by `envsubst`). `helm dependency update` fails or silently uses a stale chart.

---

## Summary

| # | File | Action | What would break without it |
|---|------|--------|---------------------------|
| 1 | `helm/agents/keda/Chart-template.yaml` | Created | Agent can't be packaged |
| 2 | `helm/agents/keda/values.yaml` | Created | Agent uses no config values |
| 3 | `helm/agents/keda/templates/agent.yaml` | Created | No agent CRD — nothing deployed |
| 4 | `helm/agents/keda/templates/rbac.yaml` | Created | Agent has no cluster permissions |
| 5 | `helm/kagent/Chart-template.yaml` | Patched | Subchart dependency missing — agent never included |
| 6 | `helm/kagent/values.yaml` | Patched | Agent disabled by default (missing condition key) |
| 7 | `Makefile` | Patched | `Chart.yaml` never generated — packaging fails |

All seven changes are necessary. Missing any one of them produces a silent failure or a broken deployment.
