# k3s Monitoring Implementation Plan (Pi 3B+ armv7 + Pi 4 + x86 VM)

## Objective
Deploy a low-overhead, mixed-architecture monitoring stack using `kube-prometheus-stack` managed by ArgoCD, with:
- 7-day retention
- heavy monitoring components pinned to Pi 4 nodes
- lightweight node-level collection on Pi 3B+ nodes
- Grafana exposed via `NodePort`
- alert rules enabled with no external notification channel in phase 1

## Constraints and Assumptions
- Cluster node mix includes:
  - Raspberry Pi 3B+ (`linux/arm/v7`, 32-bit)
  - Raspberry Pi 4 (`linux/arm64`)
  - Linux x86 VM (`linux/amd64`)
- ArgoCD already exists and is the source of truth for deployed resources.
- `metrics-server` already exists for HPA/basic metrics, but Prometheus is primary for observability.
- At least one Pi 4 node has capacity for Prometheus + Grafana + Alertmanager.

## Target Architecture

### Core Components
1. `kube-prometheus-stack` chart
2. Prometheus (PVC-backed TSDB, retention `7d`)
3. Alertmanager (deployed, no external receiver in phase 1)
4. Grafana (`NodePort`)
5. kube-state-metrics
6. node-exporter daemonset on all nodes (lightweight collector set for Pi 3)

### Scheduling Policy
- Pin heavy components to Pi 4 nodes only:
  - Prometheus
  - Alertmanager
  - Grafana
  - kube-state-metrics
- Keep node-exporter on all nodes to preserve node visibility.
- Enforce placement with:
  - `nodeSelector`
  - required node `affinity`
  - optional `tolerations` if Pi 4 nodes are tainted for infra workloads

### Required Node Label Contract
Use a stable label on Pi 4 nodes (example):
- `node-role.kubernetes.io/monitoring=true`

All scheduling rules in Helm values must key off this label.

## GitOps Structure (ArgoCD)

### Repository Layout
Use one path dedicated to monitoring:
- `monitoring/argocd-application.yaml`
- `monitoring/values.yaml`

Optional future environments:
- `monitoring/envs/prod/values.yaml`
- `monitoring/envs/lab/values.yaml`

### ArgoCD Application Contract
- Namespace: `monitoring`
- Release: `kube-prom-stack` (or similarly fixed release name)
- Source of truth: Git repo path above
- Sync policy: automated sync + prune + self-heal

## Helm Values Contract

### Global Requirements
- Every image used by enabled chart components must support:
  - `linux/arm/v7`
  - `linux/arm64`
  - `linux/amd64`
- If any subcomponent lacks `arm/v7` support, disable or replace only that subcomponent.

### Prometheus
- `prometheus.prometheusSpec.retention: 7d`
- Configure PVC storage size for 7-day retention with headroom.
- Conservative CPU/memory requests/limits tuned for Pi 4.
- Scrape interval tuned for low-power cluster profile.

### Grafana
- `service.type: NodePort`
- Use fixed nodePort in approved cluster range.
- Keep admin credentials in Kubernetes secret (not inline in values).

### Node Exporter
- Enabled on all nodes.
- Pi 3 optimization:
  - disable expensive collectors not required for first phase
  - use conservative scrape interval/timeout

### Alertmanager
- Enabled with default routing only.
- No SMTP/webhook receiver in phase 1.

## Preflight Checklist (Must Pass Before Deploy)
1. Confirm node architecture inventory:
   - `kubectl get nodes -o wide`
   - `kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.status.nodeInfo.architecture}{"\n"}{end}'`
2. Label Pi 4 nodes for monitoring placement:
   - `kubectl label node <pi4-node> node-role.kubernetes.io/monitoring=true`
3. Verify image manifest architecture support for enabled components.
4. Ensure storage class/PVC provisioning is functional in `monitoring` namespace.
5. Confirm NodePort policy and firewall path for Grafana access.

## Rollout Steps
1. Create namespace `monitoring` (or let ArgoCD create it).
2. Commit `monitoring/values.yaml` and `monitoring/argocd-application.yaml`.
3. Apply ArgoCD Application.
4. Wait for app sync and health to become green.
5. Confirm heavy pods are on Pi 4 nodes only.
6. Access Grafana via NodePort and verify dashboards.
7. Validate Prometheus targets and rule evaluation.

## Validation and Acceptance Criteria

### Scheduling and Footprint
- Prometheus, Grafana, Alertmanager, kube-state-metrics run only on Pi 4 nodes.
- node-exporter runs on Pi 3, Pi 4, and x86.
- Pi 3 nodes show no sustained monitoring-induced resource pressure.

### Data Collection
- Prometheus targets up for:
  - kube-state-metrics
  - node-exporter
  - kubelet/cadvisor endpoints (where enabled)
- Cluster, node, workload dashboards populate data.

### Retention and Storage
- Prometheus reports `7d` retention.
- TSDB growth rate is consistent with planned storage headroom.

### Alerts
- Built-in alerts visible and state transitions observable in Grafana/Alertmanager UI.
- No external delivery configured yet (intentional for phase 1).

## Test Scenarios
1. Mixed-arch image test: verify all enabled images resolve for armv7/arm64/amd64.
2. Placement enforcement: restart monitoring pods and confirm Pi 4-only rescheduling.
3. Pi 3 overhead test: compare node CPU/memory before and after rollout.
4. Target failure test: stop one node-exporter endpoint and validate target-down visibility.
5. Retention behavior test: confirm data aging/compaction follows 7-day policy.

## Security and Operations Notes
- Keep Grafana credentials in secrets and rotate after initial deployment.
- Restrict NodePort reachability with firewall/network policy where possible.
- Keep monitoring resources isolated in namespace `monitoring`.
- Do not manage monitoring resources manually once ArgoCD owns them.

## Phase 2 Extensions (Not in Current Implementation)
1. External alert receivers (email/Slack/Discord/webhooks).
2. Ingress + TLS + SSO for Grafana (replace NodePort).
3. Long-term retention or remote write backend.
4. Application-level SLO dashboards and custom business metrics.
