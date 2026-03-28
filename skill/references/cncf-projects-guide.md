# CNCF Projects Quick Reference

Pre-researched data for common CNCF projects. Use this as a starting point in Phase 1
but always fetch the live URLs to verify CRD kinds and API groups — these change across versions.

---

## Graduated Projects

### KEDA (Kubernetes Event-Driven Autoscaling)
- **Slug:** `keda`
- **API groups:** `keda.sh`
- **CRDs:** ScaledObject, ScaledJob, TriggerAuthentication, ClusterTriggerAuthentication
- **Helm repo:** `https://kedacore.github.io/charts` (chart: `kedacore/keda`)
- **RBAC:** Yes — keda.sh + apps/autoscaling
- **Pain points:** workloads stuck at zero, TriggerAuthentication misconfigured, metrics API unreachable

### Argo CD
- **Slug:** `argocd`
- **API groups:** `argoproj.io`
- **CRDs:** Application, AppProject, ApplicationSet
- **Helm repo:** `https://argoproj.github.io/argo-helm` (chart: `argo/argo-cd`)
- **RBAC:** Yes — argoproj.io
- **Pain points:** sync stuck, health status confusion, RBAC misconfiguration, webhook setup

### Flux CD
- **Slug:** `flux`
- **API groups:** `kustomize.toolkit.fluxcd.io`, `helm.toolkit.fluxcd.io`, `source.toolkit.fluxcd.io`, `notification.toolkit.fluxcd.io`, `image.toolkit.fluxcd.io`
- **CRDs:** Kustomization, HelmRelease, GitRepository, HelmRepository, OCIRepository, ImageUpdateAutomation, Receiver
- **Helm repo:** `https://fluxcd-community.github.io/helm-charts`
- **RBAC:** Yes — all toolkit.fluxcd.io groups
- **Pain points:** reconciliation failures, dependency ordering, image update automation, alerts

### cert-manager
- **Slug:** `cert-manager`
- **API groups:** `cert-manager.io`, `acme.cert-manager.io`
- **CRDs:** Certificate, CertificateRequest, Issuer, ClusterIssuer, Order, Challenge
- **Helm repo:** `https://charts.jetstack.io` (chart: `jetstack/cert-manager`)
- **RBAC:** Yes — cert-manager.io + acme.cert-manager.io
- **Pain points:** ACME challenge failures, certificate renewal timing, DNS01 solver config, secret naming

### Istio
- **Slug:** `istio`
- **API groups:** `networking.istio.io`, `security.istio.io`, `telemetry.istio.io`
- **CRDs:** VirtualService, DestinationRule, Gateway, ServiceEntry, PeerAuthentication, AuthorizationPolicy, Telemetry
- **Helm repo:** `https://istio-release.storage.googleapis.com/charts`
- **RBAC:** Yes — networking.istio.io + security.istio.io
- **Pain points:** mTLS misconfiguration, traffic routing issues, sidecar injection, certificate rotation

### Cilium
- **Slug:** `cilium`
- **API groups:** `cilium.io`
- **CRDs:** CiliumNetworkPolicy, CiliumClusterwideNetworkPolicy, CiliumEndpoint, CiliumNode
- **Helm repo:** `https://helm.cilium.io/`
- **RBAC:** Yes — cilium.io
- **Pain points:** policy enforcement mode, BGP config, Hubble observability, kube-proxy replacement

---

## Incubating Projects

### OpenTelemetry Collector
- **Slug:** `otel-collector`
- **API groups:** `opentelemetry.io`
- **CRDs:** OpenTelemetryCollector, Instrumentation (from the operator)
- **Helm repo:** `https://open-telemetry.github.io/opentelemetry-helm-charts`
- **RBAC:** Yes — opentelemetry.io
- **Pain points:** pipeline configuration, receiver/exporter setup, resource detection, sampling

### Crossplane
- **Slug:** `crossplane`
- **API groups:** `crossplane.io`, `pkg.crossplane.io`, `apiextensions.crossplane.io`
- **CRDs:** Provider, Configuration, CompositeResourceDefinition (XRD), Composition, Claim types (user-defined)
- **Helm repo:** `https://charts.crossplane.io/stable`
- **RBAC:** Yes — multiple crossplane API groups
- **Pain points:** provider credentials, composition debugging, claim binding, package dependencies

### Kyverno
- **Slug:** `kyverno`
- **API groups:** `kyverno.io`, `wgpolicyk8s.io`
- **CRDs:** ClusterPolicy, Policy, AdmissionReport, ClusterAdmissionReport, PolicyReport
- **Helm repo:** `https://kyverno.github.io/kyverno/`
- **RBAC:** Yes — kyverno.io + wgpolicyk8s.io
- **Pain points:** policy mode (audit vs enforce), generate rules, mutation side effects, webhook timeouts

### Linkerd
- **Slug:** `linkerd`
- **API groups:** `linkerd.io`, `multicluster.linkerd.io`, `policy.linkerd.io`
- **CRDs:** ServiceProfile, HTTPRoute, Server, ServerAuthorization
- **Helm repo:** `https://helm.linkerd.io/stable`
- **RBAC:** Yes — linkerd.io + policy.linkerd.io
- **Pain points:** mTLS identity, certificate expiry, multicluster setup, CNI compatibility

### Prometheus
- **Slug:** `prometheus`
- **API groups:** `monitoring.coreos.com` (via Prometheus Operator)
- **CRDs:** Prometheus, PrometheusRule, ServiceMonitor, PodMonitor, Alertmanager
- **Helm repo:** `https://prometheus-community.github.io/helm-charts` (chart: `kube-prometheus-stack`)
- **RBAC:** Yes — monitoring.coreos.com (when using operator)
- **Pain points:** scrape target discovery, alert rule syntax, retention and storage, remote write config

---

## Slug Naming Rules

| Project Name | Correct Slug | Notes |
|---|---|---|
| KEDA | `keda` | lowercase acronym |
| Argo CD | `argocd` | no hyphen, no space |
| Flux CD | `flux` | drop the "CD" |
| cert-manager | `cert-manager` | keep hyphen |
| OpenTelemetry Collector | `otel-collector` | common abbreviation |
| Kubernetes Gateway API | `kgateway` | matches existing kagent convention |
| Crossplane | `crossplane` | single word |

**Rule:** Slug = kebab-case, as short as recognizable, matches directory name exactly.

---

## RBAC Decision Matrix

| Has custom CRDs? | Generate rbac.yaml? |
|---|---|
| Yes (most CNCF projects) | Yes |
| No (project only uses standard k8s resources) | No |

When in doubt → generate rbac.yaml. A ClusterRole with no extra permissions is harmless;
a missing one causes silent access denied errors.

---

## A2A Skill Planning by Project Type

### GitOps tools (Flux, Argo CD)
Typical skills: installation-management, application-sync, source-management, troubleshooting, notifications

### Autoscaling tools (KEDA, Prometheus adapter)
Typical skills: installation-management, scaledobject-configuration, trigger-authentication, scaling-troubleshooting

### Security tools (cert-manager, Kyverno, Linkerd)
Typical skills: installation-management, certificate-management, policy-management, troubleshooting

### Networking tools (Istio, Cilium, Linkerd)
Typical skills: installation-management, traffic-management, security-policy, observability, troubleshooting

### Observability tools (Prometheus, OpenTelemetry)
Typical skills: installation-management, metric-collection, alerting, pipeline-configuration, troubleshooting
