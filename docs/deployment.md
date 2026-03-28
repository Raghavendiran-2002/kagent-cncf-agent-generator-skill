# Deployment Guide

This guide walks through deploying kagent locally and using the `cloud-native-agent-generator` skill to generate and run your first CNCF agent.

---

## Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| Docker | 20.10+ | https://docs.docker.com/get-docker/ |
| Kind | 0.20+ | `brew install kind` |
| Helm | 3.12+ | `brew install helm` |
| kubectl | 1.28+ | `brew install kubectl` |
| Claude Code | latest | https://claude.ai/code |
| envsubst | any | pre-installed on macOS/Linux |

---

## Step 1 — Clone the Repository

```bash
git clone https://github.com/Raghavendiran-2002/kagent-cncf-agent-generator-skill.git
cd kagent
```

---

## Step 2 — Create a Kind Cluster

```bash
make create-kind-cluster
```

This creates a local Kubernetes cluster named `kagent`. Verify it's running:

```bash
kubectl cluster-info --context kind-kagent
```

---

## Step 3 — Configure an LLM Provider

kagent requires an LLM model config. Create a secret with your API key:

```bash
# For OpenAI
kubectl create secret generic openai-api-key \
  --from-literal=apiKey=<your-openai-api-key> \
  -n kagent

# For Anthropic
kubectl create secret generic anthropic-api-key \
  --from-literal=apiKey=<your-anthropic-api-key> \
  -n kagent
```

---

## Step 4 — Install kagent

```bash
make helm-install
```

Verify all pods are running:

```bash
kubectl get pods -n kagent
```

Expected output:
```
NAME                              READY   STATUS    RESTARTS
kagent-controller-xxx             1/1     Running   0
kagent-server-xxx                 1/1     Running   0
kagent-ui-xxx                     1/1     Running   0
```

---

## Step 5 — Generate a CNCF Agent

Open the `kagent` directory in Claude Code (desktop app or `claude` CLI). In the chat panel, invoke the skill:

```
/cncf-agent-generator https://keda.sh https://github.com/kedacore/keda true
```

**Arguments:**
| Argument | Description | Example |
|----------|-------------|---------|
| `<docs-url>` | Official project documentation URL | `https://keda.sh` |
| `<github-url>` | Project GitHub repository URL | `https://github.com/kedacore/keda` |
| `<needs-rbac>` | Whether to generate RBAC resources | `true` / `false` |

The skill will complete all four phases and write the following files:

```
helm/agents/keda/
├── Chart-template.yaml
├── values.yaml
└── templates/
    ├── agent.yaml
    └── rbac.yaml
```

It will also patch `helm/kagent/Chart-template.yaml`, `helm/kagent/values.yaml`, and `Makefile`.

---

## Step 6 — Build the Agent Helm Package

```bash
make helm-agents
```

This runs `envsubst` on each `Chart-template.yaml` and packages it into `helm/charts/`.

---

## Step 7 — Deploy the Agent

```bash
helm upgrade --install kagent ./helm/kagent \
  --namespace kagent \
  --create-namespace \
  -f ./helm/kagent/values.yaml
```

Verify the agent pod starts:

```bash
kubectl get agents -n kagent
```

---

## Step 8 — Access the UI

```bash
kubectl port-forward -n kagent svc/kagent-ui 3000:8080
```

Open http://localhost:3000, select your agent (e.g., `keda-agent`), and start a session.

---

## Generating Additional Agents

Repeat Step 5 for any other CNCF project:

```bash
# Kyverno — policy engine
/cncf-agent-generator https://kyverno.io https://github.com/kyverno/kyverno true

# Flux — GitOps
/cncf-agent-generator https://fluxcd.io https://github.com/fluxcd/flux2 true

# Argo CD — continuous delivery
/cncf-agent-generator https://argo-cd.readthedocs.io https://github.com/argoproj/argo-cd true
```

Then rebuild and upgrade:

```bash
make helm-agents
helm upgrade kagent ./helm/kagent -n kagent -f ./helm/kagent/values.yaml
```

---

## Troubleshooting

**Agent pod not starting**
```bash
kubectl describe agent <agent-name> -n kagent
kubectl logs -l app=<agent-name> -n kagent
```

**`make helm-agents` skips a new agent**
Verify Phase 4 ran correctly — check that `helm/kagent/Chart-template.yaml` contains the new dependency entry and `Makefile` has the new `envsubst` and `helm package` lines.

**Skill produces an error during research**
The skill requires network access to fetch live documentation. Ensure Claude Code has internet access and the URLs are reachable.
