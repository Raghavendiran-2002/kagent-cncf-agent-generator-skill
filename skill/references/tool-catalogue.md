# kagent-tool-server Tool Catalogue

This reference lists every real tool available in `kagent-tool-server`.
Only use names from this list in generated `toolNames:` arrays — never invent tool names.

---

## Kubernetes Tools (`k8s_*`)

| Tool | When to use |
|------|-------------|
| `k8s_annotate_resource` | Add or update annotations on any resource |
| `k8s_apply_manifest` | Apply a YAML manifest (create or update) |
| `k8s_check_service_connectivity` | Test TCP/HTTP reachability of a Service |
| `k8s_create_resource` | Create a new resource from structured input |
| `k8s_delete_resource` | Delete a resource by kind/name/namespace |
| `k8s_describe_resource` | Get full status, events, and conditions for a resource |
| `k8s_execute_command` | Run a command inside a pod container |
| `k8s_generate_resource` | Generate YAML for a resource without applying it |
| `k8s_get_available_api_resources` | List all API resource types in the cluster |
| `k8s_get_cluster_configuration` | Retrieve cluster-level config (nodes, version, etc.) |
| `k8s_get_events` | List events in a namespace (filtered by kind/name) |
| `k8s_get_pod_logs` | Fetch pod container logs (supports tail, since, previous) |
| `k8s_get_resource_yaml` | Retrieve the live YAML of a specific resource |
| `k8s_get_resources` | List resources of a given kind in a namespace |
| `k8s_label_resource` | Add or update labels on any resource |
| `k8s_list_api_resources` | List available API groups and resource types |
| `k8s_patch_resource` | Apply a strategic or JSON merge patch to a resource |
| `k8s_remove_annotation` | Remove a specific annotation from a resource |
| `k8s_remove_label` | Remove a specific label from a resource |

### Always include (read-only baseline)

Every agent should include at minimum these read-only tools:

```yaml
- k8s_describe_resource
- k8s_get_events
- k8s_get_pod_logs
- k8s_get_resource_yaml
- k8s_get_resources
```

---

## Helm Tools (`helm_*`)

| Tool | When to use |
|------|-------------|
| `helm_get_release` | Inspect a release's values, manifests, and notes |
| `helm_list_releases` | List all Helm releases in a namespace |
| `helm_repo_add` | Register a Helm chart repository |
| `helm_repo_update` | Refresh the chart repository index |
| `helm_uninstall` | Uninstall a Helm release |
| `helm_upgrade` | Install or upgrade a Helm release |

### When to include Helm tools

Include helm tools if the CNCF project:
- Is commonly installed via Helm (nearly all of them)
- Has upgrade workflows operators need to manage
- Exposes chart values the agent should help configure

---

## Project-Specific Tools

Some CNCF projects have dedicated tools in kagent-tool-server. Only include if confirmed to exist.

| Tool Prefix | Project | Status |
|-------------|---------|--------|
| `istio_*` | Istio | Check before using |
| `cilium_*` | Cilium | Check before using |
| `argo_*` | Argo CD | Check before using |

**Rule:** If you cannot confirm a project-specific tool exists in kagent-tool-server,
do NOT include it. Use `k8s_*` tools instead. Invented tool names produce broken agent configs.

---

## Tool Selection Guide

### Minimum set (all agents)
```yaml
- k8s_describe_resource
- k8s_get_events
- k8s_get_pod_logs
- k8s_get_resource_yaml
- k8s_get_resources
```

### Add for write operations (most agents)
```yaml
- k8s_apply_manifest
- k8s_create_resource
- k8s_delete_resource
- k8s_patch_resource
```

### Add for Helm-managed projects
```yaml
- helm_get_release
- helm_list_releases
- helm_repo_add
- helm_repo_update
- helm_upgrade
```

### Add for debugging / exec
```yaml
- k8s_execute_command        # only when agent needs to run commands in pods
- k8s_check_service_connectivity  # network troubleshooting
- k8s_get_cluster_configuration   # cluster-wide info
```

### Add for labelling / annotation workflows
```yaml
- k8s_annotate_resource
- k8s_label_resource
- k8s_remove_annotation
- k8s_remove_label
```

---

## Tool Names in agent.yaml

Tool names go under `toolNames:` sorted **alphabetically**:

```yaml
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
```

Note: `helm_*` sorts before `k8s_*` alphabetically.
