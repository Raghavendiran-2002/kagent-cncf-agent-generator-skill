# cloud-native-agent-generator

> A Claude Code skill that turns any CNCF project into a production-ready [kagent](https://github.com/kagent-dev/kagent) AI agent — with one command.

**Repository:** https://github.com/Raghavendiran-2002/kagent
**Skill location:** `.claude/skills/cloud-native-agent-generator/SKILL.md`

---

## The Problem

Adding a new CNCF project agent to kagent requires expertise across six domains simultaneously — kagent internals, Helm chart conventions, RBAC design, system message authoring, tool selection, and build pipeline wiring. A contributor with deep Kubernetes knowledge can easily spend half a day on this. A newcomer might give up entirely.

## The Solution

`/cncf-agent-generator` is a Claude Code skill that drives a four-phase **Research → Design → Generate → Integrate** workflow and produces every file needed to deploy a working agent:

```
# One command. Any CNCF project.
/cncf-agent-generator https://keda.sh https://github.com/kedacore/keda true
/cncf-agent-generator https://kyverno.io https://github.com/kyverno/kyverno true
```

The skill fetches live documentation and GitHub repos (not training data), selects the correct tools from the kagent catalogue, writes a grounded system message, and wires the agent into the build pipeline — all without human intervention.

---

## What Was Built

| Artifact | Description |
|----------|-------------|
| `.claude/skills/cloud-native-agent-generator/SKILL.md` | The Claude Code skill (810 lines) driving the four-phase workflow |
| `helm/agents/keda/` | Production-ready kagent agent for KEDA — event-driven autoscaling |
| `helm/agents/kyverno/` | Production-ready kagent agent for Kyverno — policy engine |
| `helm/agents/flux/` | Production-ready kagent agent for Flux — GitOps reconciliation |

Each agent includes:
```
helm/agents/<project>/
├── Chart-template.yaml   # ${VERSION} literal for envsubst
├── values.yaml           # modelConfigRef, imagePullSecrets, resources
└── templates/
    ├── agent.yaml        # system message, tool list, a2aConfig with A2A skills
    └── rbac.yaml         # least-privilege ClusterRole + ClusterRoleBinding
```

---

## Quick Start

### Prerequisites

- [Kind](https://kind.sigs.k8s.io/) or any Kubernetes cluster
- [Helm](https://helm.sh/) v3+
- [Claude Code](https://claude.ai/code) desktop or CLI

### 1. Clone both repos

```bash
# The upstream kagent runtime
git clone https://github.com/kagent-dev/kagent.git
cd kagent

# This repo — contains the skill and generated agents
git clone https://github.com/Raghavendiran-2002/kagent-cncf-agent-generator-skill.git ../kagent-hackathon
```

### 2. Copy the skill into your kagent directory

The skill must live at `.claude/skills/cloud-native-agent-generator/` for Claude Code to detect it automatically.

```bash
# Run from inside the kagent/ directory
cp -r ../kagent-hackathon/skill .claude/skills/cloud-native-agent-generator
```

Your directory should now look like:

```
kagent/
└── .claude/
    └── skills/
        └── cloud-native-agent-generator/
            ├── SKILL.md                      ← the skill definition (810 lines)
            ├── evals/
            │   └── evals.json                ← test cases for the skill
            └── references/
                ├── tool-catalogue.md          ← all valid kagent tool names
                ├── system-message-guide.md    ← how to author good system messages
                ├── helm-chart-patterns.md     ← exact Helm conventions for kagent
                └── cncf-projects-guide.md     ← pre-researched CRD data for common projects
```

### 3. Open kagent in Claude Code

```bash
code .     # VS Code + Claude Code extension
# or
claude     # Claude Code CLI
```

The `/cncf-agent-generator` skill is immediately available in the chat panel — no configuration needed.

### 4. Generate an agent for any CNCF project

```
/cncf-agent-generator https://keda.sh https://github.com/kedacore/keda true
```

See [docs/what-the-skill-does.md](docs/what-the-skill-does.md) for exactly which files get created and modified.

### 5. Set up the cluster and deploy

```bash
make create-kind-cluster
make helm-install
make helm-agents
helm upgrade --install kagent ./helm/kagent \
  --namespace kagent \
  --create-namespace \
  -f ./helm/kagent/values.yaml
```

### 6. Start a session

```bash
kubectl port-forward -n kagent svc/kagent-ui 3000:8080
# Open http://localhost:3000 — your new agent is ready
```

---

## How It Works

See [`docs/what-the-skill-does.md`](docs/what-the-skill-does.md) — exactly which files the skill creates and which existing files it patches, with before/after diffs.

See [`docs/architecture.md`](docs/architecture.md) for a deep dive into the four-phase workflow.

See [`BLOG.md`](BLOG.md) for the full project write-up including live demo sessions.

---

## Demo Video

[![CNCF Agent Generator Demo](https://img.shields.io/badge/Watch%20Demo-Click%20Here-blue)](assets/cncf-agent-generator-demo.mp4)

> Click the badge above or [download the demo video](assets/cncf-agent-generator-demo.mp4) to watch the skill in action.

---
