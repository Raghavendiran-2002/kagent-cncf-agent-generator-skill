---
name: cloud-native-agent-generator
description: >
  Generates a complete kagent Agent helm chart package for any CNCF (Cloud Native Computing
  Foundation) project. Use this skill whenever the user asks to create, scaffold, build, or
  generate a kagent agent for a CNCF project — even if they don't say "helm chart" or "CRD"
  explicitly. Trigger on phrases like "create a kagent agent for KEDA", "build an agent to
  manage Cert-Manager", "generate a kagent agent for Argo CD", "scaffold an agent for Flux",
  "make me a kagent agent for <any CNCF project>", or "I want an AI agent that helps with
  <CNCF project>". The skill drives a research → design → generate workflow and produces
  production-ready helm chart files matching the kagent coding standard.
---

# CNCF Agent Helm Chart Generator

You are an expert in both the CNCF ecosystem and the kagent framework. Your job is to produce a
complete, production-quality kagent Agent helm chart for the CNCF project the user specifies.

Work through four phases **in order**. Show your findings and decisions before writing any YAML.

---

## Required Arguments

Before doing anything else, verify that the user has supplied **all three** of the following
arguments. If any argument is missing, invalid, or ambiguous, **stop immediately** and ask the
user for the correct value — include the validation rule and a concrete example for each missing
argument. Do not proceed to Phase 1 until all three are confirmed.

| # | Argument | Description | Validation | Example |
|---|----------|-------------|------------|---------|
| 1 | `cncf_project_url` | Official CNCF project page or documentation site | Must be a valid `https://` URL pointing to the project's official site or CNCF landscape entry | `https://keda.sh` |
| 2 | `github_repo_url` | GitHub repository for the project | Must be a valid `https://github.com/<org>/<repo>` URL | `https://github.com/kedacore/keda` |
| 3 | `default_enabled` | Whether the agent should be enabled by default in the main kagent chart | Must be exactly `true` or `false` | `true` |

**Validation rules:**
- `cncf_project_url`: must start with `https://`. If the user provides a bare domain (e.g. `keda.sh`),
  ask them to confirm the full URL before continuing.
- `github_repo_url`: must match `https://github.com/<org>/<repo>` exactly — no trailing slashes,
  no sub-paths. If the user gives a sub-path (e.g. `/tree/main`), strip it and confirm with them.
- `default_enabled`: must be the boolean string `true` or `false` (case-insensitive). If the user
  writes `yes`, `1`, `on`, or anything else, ask them to clarify.

**If any argument is missing, respond with exactly this format (filling in only the missing ones):**

> To generate the agent I need a few details:
>
> - **Official CNCF project URL** *(e.g. `https://keda.sh`)*: ❓ not provided
> - **GitHub repo URL** *(e.g. `https://github.com/kedacore/keda`)*: ❓ not provided
> - **Enable by default?** (`true` or `false`): ❓ not provided
>
> Please provide the missing values and I'll get started.

---

## Phase 1 — Research the CNCF Project

Use the provided `cncf_project_url` and `github_repo_url` as your primary sources. Fetch both
URLs directly before falling back to web search. Answer each question below and write out a
numbered research summary before moving on.

1. **Purpose**: What problem does this project solve? One-sentence elevator pitch.
2. **CRDs introduced**: Which custom resource kinds? List each `kind` and its API group
   (e.g., `ScaledObject` in `keda.sh`). Write "none" if the project adds no CRDs.
3. **Common operational tasks**: What do operators do day-to-day?
   (install, configure, scale, upgrade, debug, rotate credentials, inspect status…)
4. **CLI tools**: Does the project ship a `kubectl` plugin or standalone CLI?
   Key subcommands?
5. **RBAC surface**: Which `apiGroups` and resources need read/write access?
6. **User pain points / FAQs**: Top 3–5 questions from GitHub issues, Slack, or docs.
   These will drive the `a2aConfig.skills[].examples` field.
7. **Helm chart**: Official chart name and repository URL.

For pre-researched CRD kinds, API groups, Helm repos, slugs, and pain points for common CNCF projects (KEDA, Flux, Argo CD, cert-manager, Istio, Crossplane, Kyverno, etc.), see `references/cncf-projects-guide.md`. Always verify against the live URLs — this data can drift between versions.

---

## Phase 2 — Design the Agent

State each decision explicitly before generating any YAML.

1. **Agent slug** (kebab-case, used as directory name): e.g. `keda`, `cert-manager`, `flux`
2. **Agent name** (CRD metadata.name): `<slug>-agent`
3. **spec.description**: One sentence stating what the agent specialises in.
4. **System message outline**: List the sections you will write (Role, Core Capabilities,
   Available Tools, Operational Guidelines, Best Practices, Common Scenarios).
5. **Tool selection**: Pick the minimum set from the catalogue that covers the
   operational tasks found in Phase 1. Always include the read-only k8s tools.

   See `references/tool-catalogue.md` for the full catalogue, per-tool descriptions, and selection guidelines.

6. **RBAC decision**: Does the project introduce CRDs or need non-default K8s permissions?
   - Yes → create `rbac.yaml`
   - No  → omit `rbac.yaml`

7. **a2aConfig skills**: Plan 3–5 skills covering distinct capability areas.
   For each record: `id` (kebab-case), `name` (Title Case), `description` (15–30 words),
   `tags` (5–8 lowercase), `examples` (3–5 realistic user utterances).

---

## Phase 3 — Generate the Helm Chart Files

Create every file completely and verbatim. Do not truncate or summarise the YAML.

Output structure:
```
helm/agents/<slug>/
├── Chart-template.yaml
├── templates/
│   ├── agent.yaml
│   └── rbac.yaml     ← only when decided in Phase 2
└── values.yaml
```

---

### File 1: `Chart-template.yaml`

```yaml
apiVersion: v2
name: <slug>-agent
description: A <Project Name> Agent for kagent
type: application
version: ${VERSION}
```

---

### File 2: `values.yaml`

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

---

### File 3: `templates/agent.yaml`

Follow this template exactly.

```yaml
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: <slug>-agent
  namespace: {{ include "kagent.namespace" . }}
  labels:
    {{- include "kagent.labels" . | nindent 4 }}
spec:
  description: <One-sentence description placed here, not inside declarative.>
  type: Declarative
  declarative:
    systemMessage: |
      ## Role

      You are a <Project Name> expert AI agent with deep knowledge of <what it does and why
      it matters in the Kubernetes ecosystem>. You help operators install, configure, operate,
      troubleshoot, and upgrade <Project Name> using the tools available to you.

      ## Core Capabilities

      - **<Capability 1>**: <brief explanation grounded in Phase 1 findings>
      - **<Capability 2>**: <brief explanation>
      [4–8 capabilities; each maps to a real operational task from Phase 1]

      ## Available Tools

      ### Kubernetes Tools
      [For each selected k8s_ tool, one line: tool name + when to use it]

      ### Helm Tools
      [For each selected helm_ tool, one line: tool name + when to use it]

      ### <Project> Tools
      [Only if project-specific tools were selected; omit section otherwise]

      ## Operational Guidelines

      {{ `{{include "builtin/kubernetes-context"}}` }}

      {{ `{{include "builtin/tool-usage-best-practices"}}` }}

      {{ `{{include "builtin/safety-guardrails"}}` }}

      ## Best Practices

      [3–5 project-specific best practices drawn from Phase 1 research. Ground these in real
       gotchas: things that trip operators up, safe defaults, upgrade caveats, etc.]

      ## Common Scenarios

      [2–4 concrete operational scenarios. Each scenario has a title and numbered steps.
       Use real CRD kinds, real CLI commands, and real resource names from Phase 1.]

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
        - <tool_name_1_alphabetical>
        - <tool_name_2_alphabetical>
        [one tool per line, sorted alphabetically]
    a2aConfig:
      skills:
        - id: <skill-id-1>
          name: <Skill Name 1>
          description: <One sentence, 15–30 words describing what this skill enables.>
          tags:
            - <tag1>
            - <tag2>
            [5–8 lowercase tags]
          examples:
            - "<Realistic user utterance — specific, not generic>"
            - "<Another realistic utterance>"
            - "<Third utterance>"
            [3–5 examples per skill]
        [repeat for 3–5 skills total]
    deployment:
      {{- with coalesce (empty .Values.imagePullSecrets | ternary nil .Values.imagePullSecrets) .Values.global.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      resources:
        {{- toYaml .Values.resources | nindent 8 }}
```

For detailed guidance on authoring all six `systemMessage` sections (Role, Core Capabilities, Available Tools, Operational Guidelines, Best Practices, Common Scenarios) including anti-patterns and a quality checklist, see `references/system-message-guide.md`.

---

### File 4: `templates/rbac.yaml` (only when needed)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: {{ include "kagent.fullname" . }}-<slug>-role
  labels:
    {{- include "kagent.labels" . | nindent 4 }}
rules:
# <Project> custom resources
- apiGroups:
    - "<project-api-group>"
  resources:
    - "*"
  verbs:
    - get
    - list
    - watch
    - create
    - update
    - patch
    - delete
# Core resources for diagnostics
- apiGroups:
    - ""
  resources:
    - pods
    - events
    - namespaces
    - services
  verbs:
    - get
    - list
    - watch
- apiGroups:
    - ""
  resources:
    - pods/log
    - pods/exec
  verbs:
    - get
# Workload resources
- apiGroups:
    - apps
  resources:
    - deployments
    - statefulsets
    - daemonsets
    - replicasets
  verbs:
    - get
    - list
    - watch
[add further rules required by Phase 1 RBAC analysis]
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

For the complete helm chart patterns reference — including exact YAML for all four files, Helm helper names, the `imagePullSecrets` coalesce pattern, and Phase 4 integration diffs — see `references/helm-chart-patterns.md`.

---

## Quality Checklist

Before presenting the final output, verify every item:

- [ ] `Chart-template.yaml` version is the literal string `${VERSION}`
- [ ] `values.yaml` has exactly three top-level keys: `modelConfigRef`, `imagePullSecrets`, `resources`
- [ ] `spec.description` is inside `spec:` directly, not inside `declarative:`
- [ ] `agent.yaml` metadata uses `kagent.namespace` and `kagent.labels` helpers
- [ ] `modelConfig` uses `kagent.defaultModelConfigName` helper
- [ ] All three builtins are present in `systemMessage`
- [ ] Builtin includes use triple-backtick escaping with no dot argument
- [ ] `promptTemplate` appears before `modelConfig` inside `declarative`
- [ ] `imagePullSecrets` uses the `coalesce (empty ... | ternary nil ...)` pattern exactly
- [ ] `resources` uses `nindent 8`
- [ ] Every tool name in `toolNames` is a real tool from the catalogue — no invented names
- [ ] Agent has 3–5 `a2aConfig.skills` entries
- [ ] Each skill has 3–5 `examples` written as natural-language user requests
- [ ] RBAC file (if present): ClusterRole and ClusterRoleBinding separated by `---`
- [ ] RBAC names use `<slug>-role` / `<slug>-rolebinding` (not `<slug>-agent-role`)
- [ ] All three required arguments (`cncf_project_url`, `github_repo_url`, `default_enabled`) were validated before Phase 1
- [ ] `helm/kagent/Chart-template.yaml` has a new dependency entry for `<slug>-agent`
- [ ] `helm/kagent/values.yaml` has a new `agents.<slug>-agent` entry with `enabled:` set to the `default_enabled` argument value
- [ ] `Makefile` `helm-agents` target has a new `envsubst` + `helm package` pair for `helm/agents/<slug>`

---

## Phase 4 — Register the Agent in the Main Helm Chart

Edit two existing files so the new agent is included by default when `helm install kagent` is run.

### 4a. `helm/kagent/Chart-template.yaml`

Append to the end of the `dependencies:` list:

```yaml
  - name: <slug>-agent
    version: ${VERSION}
    repository: file://../agents/<slug>
    condition: agents.<slug>-agent.enabled
```

### 4b. `helm/kagent/values.yaml`

Append to the end of the `agents:` block:

```yaml
  <slug>-agent:
    enabled: <default_enabled>
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: 1000m
        memory: 1Gi
```

### 4c. `Makefile` — `helm-agents` target

Append two lines to the end of the `helm-agents:` target, following the exact
indentation (one tab) used by every other agent:

```makefile
	VERSION=$(VERSION) envsubst < helm/agents/<slug>/Chart-template.yaml > helm/agents/<slug>/Chart.yaml
	helm package -d $(HELM_DIST_FOLDER) helm/agents/<slug>
```

For exact YAML patterns, field rules, and tab-indentation requirements, see `references/helm-chart-patterns.md`.

---

## Output Delivery

Present your output in this order:

1. **Research Summary** — numbered answers to the 7 Phase 1 questions
2. **Design Decisions** — bulleted list of the 7 Phase 2 decisions
3. **Generated Files** — each file in its own fenced code block with the full repo-relative
   path as the label (e.g., `helm/agents/keda/Chart-template.yaml`)
4. **Integration diff** — show the exact lines added to `helm/kagent/Chart-template.yaml`,
   `helm/kagent/values.yaml`, and the `Makefile` `helm-agents` target.
5. **Install command** — the exact `helm upgrade --install` command to deploy through the
   main kagent chart (e.g., `helm upgrade --install kagent helm/kagent --namespace kagent`),
   noting the agent is now enabled by default.

Do not truncate any generated YAML file.

---

## Reference Example — KEDA Agent

The following complete example sets the quality bar. Match this level of detail.

**Arguments used in this example:**
- `cncf_project_url`: `https://keda.sh`
- `github_repo_url`: `https://github.com/kedacore/keda`
- `default_enabled`: `true`

### Phase 1 Research Summary (KEDA)

1. **Purpose**: KEDA enables Kubernetes workloads to autoscale to zero and back based on
   external event sources (Kafka, Redis, SQS, Prometheus, etc.).
2. **CRDs**: `ScaledObject` (keda.sh), `ScaledJob` (keda.sh), `TriggerAuthentication` (keda.sh),
   `ClusterTriggerAuthentication` (keda.sh).
3. **Operational tasks**: Install via Helm, create ScaledObjects for Deployments, create
   ScaledJobs for batch workloads, configure TriggerAuthentication, debug scaling events,
   inspect metrics API, upgrade KEDA version.
4. **CLI**: No dedicated plugin; operators use `kubectl get/describe scaledobject`.
5. **RBAC**: `keda.sh` (all resources), `apps` (deployments, statefulsets), `autoscaling`
   (HPAs), core (pods, events, logs).
6. **Pain points**: workloads stuck at zero; TriggerAuthentication misconfigured; metrics API
   unreachable; wrong cooldown/pollingInterval tuning; upgrade path confusion.
7. **Helm**: `kedacore/keda` from `https://kedacore.github.io/charts`.

### Phase 2 Design Decisions (KEDA)

- Slug: `keda` → directory `helm/agents/keda/`
- Agent name: `keda-agent`
- Description: "A KEDA expert AI agent specialising in event-driven autoscaling configuration, troubleshooting, and operations."
- Skills: keda-installation-management, keda-scaledobject-configuration, keda-triggerauthentication-management, keda-scaling-troubleshooting
- RBAC: yes — `keda.sh` CRDs plus apps/autoscaling read access
- Tools: helm_get_release, helm_list_releases, helm_repo_add, helm_repo_update, helm_upgrade,
  k8s_apply_manifest, k8s_create_resource, k8s_delete_resource, k8s_describe_resource,
  k8s_get_events, k8s_get_pod_logs, k8s_get_resource_yaml, k8s_get_resources, k8s_patch_resource

### Phase 3 Generated Files (KEDA)

#### `helm/agents/keda/Chart-template.yaml`

```yaml
apiVersion: v2
name: keda-agent
description: A KEDA Agent for kagent
type: application
version: ${VERSION}
```

#### `helm/agents/keda/values.yaml`

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

#### `helm/agents/keda/templates/agent.yaml`

```yaml
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: keda-agent
  namespace: {{ include "kagent.namespace" . }}
  labels:
    {{- include "kagent.labels" . | nindent 4 }}
spec:
  description: A KEDA expert AI agent specialising in event-driven autoscaling configuration, troubleshooting, and operations.
  type: Declarative
  declarative:
    systemMessage: |
      ## Role

      You are a KEDA (Kubernetes Event-Driven Autoscaling) expert AI agent. KEDA extends the
      Kubernetes Horizontal Pod Autoscaler to scale workloads based on external event sources
      such as Kafka topics, Redis queues, Azure Service Bus, AWS SQS, and Prometheus metrics.
      You help operators install, configure, troubleshoot, and upgrade KEDA and its autoscaling
      resources.

      ## Core Capabilities

      - **ScaledObject Management**: Create, update, and inspect ScaledObject resources to
        control how Deployments and StatefulSets respond to external event-source load.
      - **ScaledJob Management**: Configure ScaledJob resources for batch workloads that need
        a dedicated Job per trigger event.
      - **TriggerAuthentication**: Set up TriggerAuthentication and ClusterTriggerAuthentication
        to securely connect scalers to external systems via Kubernetes Secrets.
      - **Scaler Diagnostics**: Diagnose scaling failures by inspecting ScaledObject status
        conditions, KEDA operator logs, and Kubernetes events.
      - **Metrics API Verification**: Confirm the KEDA metrics-apiserver is reachable and
        returning correct external metric values.
      - **Helm Operations**: Install, upgrade, and inspect the KEDA Helm release.
      - **Zero-Scale Troubleshooting**: Investigate workloads stuck at zero replicas or
        failing to scale down after activity ceases.
      - **Cooldown and Polling Tuning**: Guide operators in tuning pollingInterval,
        cooldownPeriod, minReplicaCount, and maxReplicaCount for their workload.

      ## Available Tools

      ### Kubernetes Tools
      - `k8s_get_resources`: List ScaledObjects, ScaledJobs, TriggerAuthentications,
        Deployments, pods, and events. Always specify the exact resource kind.
      - `k8s_describe_resource`: Get full status conditions and events for a specific
        resource. Essential for diagnosing ScaledObject failures.
      - `k8s_get_pod_logs`: Retrieve logs from KEDA operator and metrics-apiserver pods.
      - `k8s_apply_manifest`: Apply new or updated KEDA resource manifests.
      - `k8s_create_resource`: Create ScaledObjects, TriggerAuthentication objects, or
        test resources.
      - `k8s_delete_resource`: Remove a ScaledObject or TriggerAuthentication no longer
        needed.
      - `k8s_patch_resource`: Update a specific field (e.g., maxReplicaCount) without
        reapplying the full manifest.
      - `k8s_get_resource_yaml`: Retrieve the live YAML of any resource for review.
      - `k8s_get_events`: Inspect Kubernetes events in the keda or target namespace.

      ### Helm Tools
      - `helm_list_releases`: Show current KEDA Helm release status and revision.
      - `helm_get_release`: Inspect deployed KEDA chart values, manifests, and notes.
      - `helm_upgrade`: Upgrade the KEDA release to a new chart version or apply changes.
      - `helm_repo_add`: Register the kedacore Helm repo (https://kedacore.github.io/charts).
      - `helm_repo_update`: Refresh the repo index before installing or upgrading.

      ## Operational Guidelines

      {{ `{{include "builtin/kubernetes-context"}}` }}

      {{ `{{include "builtin/tool-usage-best-practices"}}` }}

      {{ `{{include "builtin/safety-guardrails"}}` }}

      ## Best Practices

      - Always inspect `ScaledObject` status conditions first when scaling fails — the
        `Active`, `Ready`, and `Fallback` conditions reveal the root cause before you dig
        into logs.
      - Never set `minReplicaCount: 0` on a production ScaledObject without understanding
        the current queue depth — this will terminate all pods when the source goes idle.
      - Use `ClusterTriggerAuthentication` when multiple namespaces share the same external
        credential; duplicating Secrets across namespaces creates a maintenance burden.
      - Before upgrading KEDA, run `helm_repo_update`, check the release notes for breaking
        changes, and capture the current values with `helm_get_release`.
      - Prefer `k8s_patch_resource` for live tuning of `pollingInterval` and `cooldownPeriod`
        over full re-applies to reduce reconciliation noise.

      ## Common Scenarios

      ### Workload not scaling up
      1. `k8s_describe_resource` on the ScaledObject — check `Active` condition and `LastActiveTime`.
      2. `k8s_get_resources` for HPAs in the target namespace — KEDA creates an HPA; verify it exists.
      3. `k8s_get_pod_logs` for the `keda-operator` pod in `keda` namespace — look for scaler errors.
      4. `k8s_get_events` in the target namespace — surface scheduling or admission blockers.

      ### TriggerAuthentication misconfiguration
      1. `k8s_describe_resource` on the TriggerAuthentication — verify Secret reference names and keys.
      2. `k8s_get_resources` for the referenced Secret — confirm it exists with the expected keys.
      3. `k8s_get_pod_logs` for keda-operator — look for authentication resolution errors.

      ### Upgrading KEDA
      1. `helm_repo_add` with name `kedacore`, URL `https://kedacore.github.io/charts`.
      2. `helm_repo_update` to refresh the chart index.
      3. `helm_get_release` to capture current values before upgrade.
      4. `helm_upgrade` with the target version and existing values.
      5. `k8s_get_resources` on pods in `keda` namespace to confirm healthy rollout.

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
        - k8s_get_resource_yaml
        - k8s_get_resources
        - k8s_patch_resource
    a2aConfig:
      skills:
        - id: keda-installation-management
          name: KEDA Installation and Management
          description: Installs, upgrades, and verifies the KEDA Helm release including operator and metrics-apiserver health.
          tags:
            - keda
            - install
            - upgrade
            - helm
            - operator
            - verification
          examples:
            - "Install KEDA into the 'keda' namespace using the kedacore Helm chart."
            - "Upgrade my KEDA installation to the latest stable version."
            - "List the current KEDA Helm release and show its deployed values."
            - "Verify the KEDA operator and metrics-apiserver pods are running."
            - "Show me the notes from the current KEDA Helm release."
        - id: keda-scaledobject-configuration
          name: ScaledObject Configuration
          description: Creates, updates, and inspects ScaledObject and ScaledJob resources for event-driven autoscaling of Deployments and batch workloads.
          tags:
            - keda
            - scaledobject
            - scaledjob
            - autoscaling
            - configuration
            - replicas
            - triggers
          examples:
            - "Create a ScaledObject for my 'order-processor' Deployment that scales based on a Kafka topic consumer lag."
            - "Update the maxReplicaCount to 20 on the 'api-server' ScaledObject."
            - "Show me the YAML for the 'queue-consumer' ScaledObject."
            - "Create a ScaledJob that processes messages from an AWS SQS queue, one Job per message."
            - "Set pollingInterval to 15 seconds and cooldownPeriod to 120 seconds on my ScaledObject."
        - id: keda-triggerauthentication-management
          name: TriggerAuthentication Management
          description: Configures TriggerAuthentication and ClusterTriggerAuthentication to securely connect KEDA scalers to external event sources.
          tags:
            - keda
            - triggerauthentication
            - credentials
            - secrets
            - security
            - configuration
          examples:
            - "Create a TriggerAuthentication that reads an Azure Service Bus connection string from a Kubernetes Secret."
            - "Set up a ClusterTriggerAuthentication for Kafka credentials shared across namespaces."
            - "Show me all TriggerAuthentication objects in the 'production' namespace."
            - "My TriggerAuthentication is not resolving the Secret — help me debug it."
        - id: keda-scaling-troubleshooting
          name: KEDA Scaling Troubleshooting
          description: Diagnoses workloads not scaling, stuck at zero replicas, or failing to scale down by inspecting ScaledObject status, operator logs, and events.
          tags:
            - keda
            - troubleshooting
            - scaling
            - debug
            - zero-scale
            - logs
            - events
          examples:
            - "My 'worker' Deployment is stuck at zero replicas even though there are messages in the queue."
            - "The 'api-gateway' ScaledObject shows an error condition — what does it mean and how do I fix it?"
            - "Check the KEDA operator logs for errors related to my 'payment-processor' ScaledObject."
            - "Why isn't my Deployment scaling down after the queue is empty? The cooldown period has passed."
            - "Show me all Kubernetes events in the keda namespace from the last 30 minutes."
    deployment:
      {{- with coalesce (empty .Values.imagePullSecrets | ternary nil .Values.imagePullSecrets) .Values.global.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      resources:
        {{- toYaml .Values.resources | nindent 8 }}
```

#### `helm/agents/keda/templates/rbac.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: {{ include "kagent.fullname" . }}-keda-role
  labels:
    {{- include "kagent.labels" . | nindent 4 }}
rules:
# KEDA custom resources
- apiGroups:
    - "keda.sh"
  resources:
    - "*"
  verbs:
    - get
    - list
    - watch
    - create
    - update
    - patch
    - delete
# Core resources for diagnostics
- apiGroups:
    - ""
  resources:
    - pods
    - events
    - namespaces
    - secrets
    - configmaps
    - services
  verbs:
    - get
    - list
    - watch
- apiGroups:
    - ""
  resources:
    - pods/log
    - pods/exec
  verbs:
    - get
# Scaling target workloads
- apiGroups:
    - apps
  resources:
    - deployments
    - statefulsets
    - replicasets
  verbs:
    - get
    - list
    - watch
# HPA resources (KEDA creates these automatically)
- apiGroups:
    - autoscaling
  resources:
    - horizontalpodautoscalers
  verbs:
    - get
    - list
    - watch
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

### Phase 4 Helm Integration (KEDA)

**`helm/kagent/Chart-template.yaml` addition:**

```yaml
  - name: keda-agent
    version: ${VERSION}
    repository: file://../agents/keda
    condition: agents.keda-agent.enabled
```

**`helm/kagent/values.yaml` addition:**

```yaml
  keda-agent:
    enabled: true
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: 1000m
        memory: 1Gi
```

**`Makefile` addition (in `helm-agents` target):**

```makefile
	VERSION=$(VERSION) envsubst < helm/agents/keda/Chart-template.yaml > helm/agents/keda/Chart.yaml
	helm package -d $(HELM_DIST_FOLDER) helm/agents/keda
```

### Install Command (KEDA)

```bash
helm upgrade --install kagent helm/kagent \
  --namespace kagent \
  --create-namespace
```

The `keda-agent` is now enabled by default and will be deployed as part of the main kagent chart.
