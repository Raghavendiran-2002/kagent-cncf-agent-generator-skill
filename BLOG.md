# From Zero to Production-Ready CNCF Agent in 5 Minutes — For Any CNCF Project

The CNCF ecosystem includes over 200 projects — each one capable of becoming a kagent-powered AI agent. Previously, integrating any of them required hours of effort: understanding the project deeply, writing accurate system instructions, configuring the right tools, setting up proper RBAC policies, and wiring everything into the build pipeline.

Now, all of that is automated. I built a Claude Code skill that turns **any** CNCF project into a fully functional kagent agent with a single command — while also teaching you what each project does as it works.

---

## Highlights

- **One command** generates a complete, production-ready kagent agent for any CNCF project — including Helm chart, RBAC, system message, A2A skills, and full build pipeline integration
- **Works for all 200+ CNCF projects** — the same skill generated agents for KEDA, Kyverno, and Flux; it will work equally well for Argo CD, Cert-Manager, Crossplane, or any project you point it at
- **Research-backed:** Claude fetches live documentation and GitHub repos so agents reflect the actual current state of each project, not stale training data
- **End-to-end:** Generated agents aren't just scaffolding — the KEDA agent can deploy KEDA into your cluster conversationally, the Kyverno agent can immediately start authoring and enforcing policies

---

## The Problem

Adding a new CNCF project agent to kagent requires expertise across at least six domains simultaneously:

**1. Understanding kagent itself** — Before writing a single file, you need to understand how kagent deploys agents via Helm, how the Agent CRD is structured, and what the runtime expects. For a newcomer this alone is a multi-hour investment.

**2. Helm chart structure** — kagent has strict conventions for `Chart-template.yaml`, `values.yaml`, and template files that differ from standard Helm patterns.

**3. RBAC design** — each CNCF project has its own CRD API groups and verbs; getting least-privilege access right takes research and iteration. Kyverno alone spans three API groups: `kyverno.io`, `policies.kyverno.io`, and `wgpolicyk8s.io`.

**4. System message authoring** — an effective agent needs a grounded, accurate system message covering the project's CRDs, operational tasks, pain points, and best practices. You have to deeply understand the CNCF project first, then translate that knowledge into an instruction set the agent will actually follow.

**5. Tool analysis** — you need to evaluate the full kagent-tool-server catalogue and select the minimum set of tools the agent actually needs, avoiding both gaps (missing critical tools) and bloat (tools that create unnecessary permissions or confusion).

**6. Build pipeline wiring** — three separate files (`Chart-template.yaml`, `values.yaml`, `Makefile`) must be updated or `make helm-agents` silently skips the new agent.

A contributor with deep Kubernetes knowledge but no kagent experience can easily spend half a day on this. A newcomer might give up entirely. That's the barrier we're removing.

---

## The Solution

`/cncf-agent-generator` is a Claude Code skill that turns three arguments into a complete, deployable kagent agent. It works for **any** CNCF project — here are two real invocations:

```
# Generate an agent for KEDA (event-driven autoscaling)
/cncf-agent-generator https://keda.sh https://github.com/kedacore/keda true

# Generate an agent for Kyverno (policy engine)
/cncf-agent-generator https://kyverno.io https://github.com/kyverno/kyverno true
```

No template copying. No manual RBAC research. No Makefile spelunking. The skill drives a four-phase workflow — **Research → Design → Generate → Integrate** — and produces every file conforming to the kagent coding standard.

---

## How It Works

### Phase 1 — Research (Live, Not Training Data)

The skill fetches the project's official documentation URL and GitHub repository in real time. It adapts entirely to the project you give it.

For **KEDA**, it reads the operator guide, CRD reference, and troubleshooting pages to extract:
- **CRD kinds:** `ScaledObject`, `ScaledJob`, `TriggerAuthentication`, `ClusterTriggerAuthentication`
- **API groups:** `keda.sh`
- **RBAC surface:** full CRUD on KEDA CRDs, read access to workloads and HPAs
- **Pain points:** zero-scale trigger timing, cooldown misconfiguration, metrics API connectivity failures

For **Kyverno**, it reads the installation guide, CRD reference, and policy library to extract:
- **CRD kinds:** `ClusterPolicy`, `Policy`, `PolicyReport`, `ClusterPolicyReport`, `PolicyException`, `ClusterPolicyException`, `CleanupPolicy`
- **API groups:** `kyverno.io`, `policies.kyverno.io`, `wgpolicyk8s.io`
- **RBAC surface:** full CRUD on Kyverno CRDs, read access to all workloads, webhook configuration access for recovery operations
- **Pain points:** webhook deadlock when Kyverno's own admission webhook is misconfigured; confusion between `Audit` and `Enforce` modes; debugging policy failures without the Kyverno CLI

This grounding is what makes the generated system message accurate. An agent trained only on Claude's knowledge cutoff would miss project-specific nuance and recent API changes.

### Phase 2 — Design

With research complete, Claude selects the minimum set of real kagent tools needed (no invented tool names — a 20-item quality checklist enforces this), decides whether RBAC resources are required, and plans 4–5 A2A skills with 3–5 realistic user utterances each.

For Kyverno, this produced five skills:
- `install-kyverno` — Deploy Kyverno via Helm with configurable HA settings
- `author-policy` — Write and apply ClusterPolicy/Policy with validate, mutate, or generate rules
- `audit-compliance` — Query PolicyReports and ClusterPolicyReports for compliance posture
- `manage-exceptions` — Create PolicyExceptions for known-good workloads
- `troubleshoot-kyverno` — Diagnose webhook failures, policy misconfigurations, and controller issues

### Phase 3 — File Generation

Four files are produced in full — no truncation, no placeholders. The directory structure is identical for every project:

```
helm/agents/<project>/
├── Chart-template.yaml   # with ${VERSION} literal for envsubst
├── values.yaml           # modelConfigRef, imagePullSecrets, resources
└── templates/
    ├── agent.yaml        # full system message, tool list, a2aConfig with skills
    └── rbac.yaml         # least-privilege ClusterRole + ClusterRoleBinding
```

The `agent.yaml` system message is grounded directly in Phase 1 findings — not generic Kubernetes advice. The `rbac.yaml` covers all API groups discovered during research.

### Phase 4 — Helm Integration

The skill edits three files automatically so the agent is immediately buildable:

```yaml
# helm/kagent/Chart-template.yaml — new dependency added
- name: kyverno-agent
  version: ${VERSION}
  repository: file://../agents/kyverno
  condition: agents.kyverno-agent.enabled
```

```yaml
# helm/kagent/values.yaml — new agent block added
kyverno-agent:
  enabled: true
  modelConfigRef: ""
```

```makefile
# Makefile — new envsubst + helm package lines added
envsubst < helm/agents/kyverno/Chart-template.yaml > helm/agents/kyverno/Chart.yaml
helm package helm/agents/kyverno --destination helm/charts/
```

Without Phase 4, `make helm-agents` silently skips the new agent — the most common silent failure when contributing agents manually.

---

## The Generated Agents in Action

The skill produces working agents, not just scaffolding. Here are real sessions from two different agents — both generated by the same skill, pointed at different CNCF projects.

### Session 1: KEDA — Deploying an Autoscaler

**User:**
```
deploy keda in the cluster
```

**KEDA Agent:**
```
I'll help you deploy KEDA. A few quick questions:

1. Which namespace should I install KEDA into? (keda is the standard choice)
2. Which version — latest stable (v2.16.0) or a specific version?
3. Any custom configuration needed? (high availability mode, custom metrics API
   server settings, image pull secrets)
```

**User:**
```
keda namespace, latest stable, default config
```

**Agent executes:**
```
helm_repo_add: name=kedacore, url=https://kedacore.github.io/charts
helm_repo_update
helm_upgrade: release=keda, chart=kedacore/keda, namespace=keda,
              create_namespace=true, version=2.16.0
```

**Agent confirms:**
```
KEDA is deployed. Both pods are running:

keda-operator                      1/1 Running
keda-operator-metrics-apiserver    1/1 Running

Your cluster is ready to use ScaledObjects and TriggerAuthentications.
```

---

### Session 2: Kyverno — Authoring a Security Policy

**User:**
```
create a validate policy that blocks privileged containers across the cluster
```

**Kyverno Agent:**
```
I'll create a ClusterPolicy that blocks privileged containers in Enforce mode.
Here's what I'll apply:

---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged-containers
  annotations:
    policies.kyverno.io/title: Disallow Privileged Containers
    policies.kyverno.io/category: Pod Security Standards (Baseline)
    policies.kyverno.io/severity: medium
    policies.kyverno.io/description: >-
      Privileged mode disables most security mechanisms and must not be allowed.
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: privileged-containers
      match:
        any:
        - resources:
            kinds:
              - Pod
      validate:
        message: "Privileged mode is disallowed. Set securityContext.privileged to false."
        pattern:
          spec:
            =(initContainers):
              - =(securityContext):
                  =(privileged): "false"
            containers:
              - =(securityContext):
                  =(privileged): "false"

Applying now...
```

**Agent confirms:**
```
Policy applied successfully.

NAME                            ADMISSION   BACKGROUND   READY   AGE
disallow-privileged-containers  true        true         True    5s

Any new Pod requesting privileged mode will be blocked at admission.
Existing pods are not affected (background scanning will report violations
in PolicyReports).
```

---

Two completely different CNCF projects. One skill. Same quality of output.

---

## What Was Built

| Artifact | Description |
|----------|-------------|
| `.claude/skills/cloud-native-agent-generator/SKILL.md` | The Claude Code skill driving the four-phase workflow |
| `helm/agents/keda/` | Complete kagent helm chart for KEDA (generated by the skill) |
| `helm/agents/kyverno/` | Complete kagent helm chart for Kyverno (generated by the skill) |
| `helm/agents/flux/` | Complete kagent helm chart for Flux (generated by the skill) |

---

## Technical Highlights

**No hallucination by design.** Phase 1 uses WebFetch against real URLs. Tool names in the generated `agent.yaml` are validated against the live kagent-tool-server catalogue — the quality checklist explicitly rejects any name not in the known list.

**20-item quality checklist.** Before output, the skill verifies: `${VERSION}` literal present, YAML indentation correct, no invented tool names, 3–5 A2A skills with 3–5 examples each, RBAC wired into the ClusterRoleBinding, all four Helm integration points updated. This prevents the most common silent failures.

**A2A protocol native.** Every agent exposes skills via the kagent Agent-to-Agent protocol, making them composable. A future orchestrator agent could delegate to the KEDA agent for scaling configuration and the Kyverno agent for policy compliance in the same workflow.

**Zero extra tooling.** The skill runs entirely inside Claude Code. No separate scripts, no Python environment, no additional CLIs. If you have Claude Code open in the kagent repo, you have the skill.

---

## Demo Video

Watch the full workflow — from invoking the skill to deployed, queryable agents — in the demo video:

`<link to demo video>`

The demo covers: problem statement → skill invocation → research phase output → generated files → Helm integration diff → `make helm-agents` deployment → live KEDA deployment conversation → live Kyverno policy authoring session.

---

## GitHub & Getting Started

**Repository:** https://github.com/Raghavendiran-2002/kagent
**Skill location:** `.claude/skills/cloud-native-agent-generator/SKILL.md`

### Quick Start

```bash
# 1. Clone the repo and open in Claude Code
git clone https://github.com/Raghavendiran-2002/kagent-cncf-agent-generator-skill.git
cd kagent
code .  # or open in Claude Code desktop

# 2. Install kagent into a Kind cluster
make create-kind-cluster
make helm-install

# 3. Generate an agent for any CNCF project
# (In Claude Code chat)
/cncf-agent-generator https://keda.sh https://github.com/kedacore/keda true
# or
/cncf-agent-generator https://kyverno.io https://github.com/kyverno/kyverno true

# 4. Build and deploy the new agent
make helm-agents
helm upgrade --install kagent ./helm/kagent \
  --namespace kagent \
  --create-namespace \
  -f ./helm/kagent/values.yaml

# 5. Open kagent UI and start a session with your new agent
kubectl port-forward -n kagent svc/kagent-ui 3000:8080
```

---

## The Bigger Picture

KEDA and Kyverno are two of 200+ graduated and incubating CNCF projects. Every one of them manages some critical slice of your infrastructure — and every one of them is now one command away from becoming an AI agent that knows that project deeply, speaks its language, and can act on your cluster.

The skill doesn't have a list of supported projects. It has no built-in knowledge of KEDA or Kyverno. It only has a process: fetch the real docs, extract what matters, generate what works. Point it at anything and it adapts.

That's the shift. Not "here are the agents we built." But: the infrastructure to build *any* agent, on demand, correctly, in five minutes.

---
