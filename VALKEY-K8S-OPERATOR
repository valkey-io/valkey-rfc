RFC: 21
Status: Proposed

# Valkey Kubernetes Operator RFC

## Abstract

This RFC proposes the development of an open-source Kubernetes Operator for Valkey. 
The Operator will automate the provisioning, configuration, and lifecycle management of Valkey clusters within Kubernetes environments. 
It will support standalone and Sentinel-based deployments, integrate with cert-manager for TLS, expose Prometheus metrics, and support persistent storage configurations, offering production-grade capabilities for cloud-native environments.

## Motivation

Operating Valkey at scale in Kubernetes environments remains complex and time-consuming. 
Most enterprise users currently manage Valkey deployments manually or via generic Helm charts, lacking production-readiness, automation, or compliance guarantees.

This Operator addresses these challenges by:
- Providing a Kubernetes-native way to deploy and manage cluster mode Valkey, which is the only deployment type that provides HA and horizontal scaling.
- Enabling self-service for developers via CRDs
- Automating TLS setup, monitoring, and failover
- Supporting advanced production use cases:
  - Cluster mode deployments for HA and scalability (primary focus)
  - Sentinel-based HA deployments for organizations already using Sentinel (later)
  - Persistent storage and backup/restore workflows
- Providing simplified standalone or Sentinel-only deployments for smaller or test environments where full cluster mode is unnecessary
- Matching enterprise expectations for security, observability, and compliance
- Operator SDK capability levels: aim for Level 5 long-term, starting at Level 3 and progressing incrementally

Unlike Redis, which has various third-party operators with unclear support models, Valkey has the opportunity to lead with an officially supported Kubernetes-native experience.

## Design considerations

The operator will be implemented using Kubebuilder and expose a Custom Resource Definition (CRD) named `Valkey`, allowing users to declaratively deploy Valkey instances via YAML. 

Key design principles:
- User-first: Apply 10 lines of YAML and get a working, HA Valkey deployment.
- Cluster-first focus: Prioritize cluster mode as the production-grade setup — HA, scalable, and resilient
- Fallback / simpler modes: Support:
  - Standalone: a single-node deployment (optionally with replicas) without Sentinel — intended for dev/test or simple use cases
  - Sentinel HA: quorum-based failover for smaller-scale production setups
- Composability: Work alongside tools like cert-manager, Prometheus Operator, and Everest.
- Security: TLS by default, with support for mTLS where required.
- Modularity: Start simple (standalone + Sentinel), then iterate (autoscaling, dashboards).
- Observability: Out-of-the-box integration with Prometheus (via redis-exporter sidecar and ServiceMonitor).
- Deployment approach:
  - A Helm chart may be provided to install the Valkey Operator itself into a Kubernetes cluster.
  - Once deployed, the Operator will manage Valkey resources directly via manifests and CRDs.
  - The Operator will not wrap a Helm chart for managing workloads.
- Reliability: The operator must mitigate failure scenarios by:
  - Distributing pods across nodes/availability zones (anti-affinity)
  - Handling Kubernetes worker failures gracefully
  - Supporting rolling updates of workers without downtime

Comparative notes:
- SAP’s internal `valkey-operator` is optimized for Sentinel and metrics but is not distribution-friendly.
- Hyperspike’s operator is more lightweight and public, but lacks Sentinel and TLS support.
This RFC proposes an open-source Valkey operator that balances ease of use and production-grade features.

## Requirements

The following requirements represent needs, wishes, and operational considerations gathered so far. 
Not all of them need to be implemented in the first version, but capturing them early helps guide design.

### Cluster deployment
- Orchestrate starting up a cluster.
- Orchestrate taking down a cluster.
- Orchestrate rolling upgrades without downtime, using the [Valkey Cluster Tutorial upgrade procedure](https://valkey.io/topics/cluster-tutorial/#upgrade-nodes-in-a-valkey-cluster).
- Orchestrate scale-out and scale-in by adding/removing nodes (primaries and replicas) and migrating slots without downtime.
- When adding nodes to an existing cluster:
  - Create a pod and add the node with `CLUSTER MEET`.
  - K8s readiness probe after `CLUSTER MEET`.
  - Verify cluster convergence (`CLUSTER NODES` or similar).
  - Use readiness gates until replicas finish loading/replicating.
- Anti-affinity: ensure primaries and replicas are on different workers; spread primaries evenly across workers.
- Support Kubernetes worker node cordon and drain.
- Handle TLS cert reload (`CONFIG SET`).
- Handle etcd failures/restarts and API server unavailability.
- Handle network partitioning of the operator, primaries, or replicas.
- Emit JSON-structured logging for alarms/events.
- Ensure the operator never gets stuck in a non-working state.
- Support live config settings (e.g., `CONFIG SET` without full restart).
- Support module loading where provided by the container image.

### Sentinel deployment
- Support quorum-based failover (configurable quorum size).
- Allow scaling Sentinels independently of primaries/replicas.
- Provide service names for routing clients directly to primaries.
- Support AZ-aware scheduling for Sentinel pods.

### Standalone deployment
- Provide lightweight single-node deployment.
- Allow optional replicas (without Sentinel).
- Focus on simple dev/test environments where HA is not required.

## Specification

CRD Strategy

For the initial implementation, we propose a single `Valkey` CRD with a `mode` field 
(`cluster`, `sentinel`, `standalone`) to simplify adoption.  
This CRD will map to the appropriate controller logic depending on the mode defined.  

For future compatibility, we may split this into separate CRDs 
(e.g., `ValkeyCluster`, `ValkeySentinel`, `ValkeyStandalone`) if complexity or user demand grows.  
This phased approach allows us to start simple while leaving room to evolve the API surface.

CRD Schema Overview (example)
mode: cluster (recommended for production), sentinel (HA for small-scale), standalone (simple deployments or dev/test)

```yaml
apiVersion: cache.valkey.io/v1alpha1
kind: Valkey
metadata:
  name: my-valkey-cluster
spec:
  mode: cluster
  replicas: 3
  sentinel:
    enabled: true
  metrics:
    enabled: true
  tls:
    enabled: true
  persistence:
    enabled: true
    type: both
    storageClass: fast-disks
    size: 5Gi
  config:
    configMapName: custom-valkey-conf
  modules:
    - name: libvalkey_bloom.so
      path: /opt/valkey/modules/libvalkey_bloom.so
```

## Operator Behavior

- Manages Valkey resources directly via Kubernetes manifests and CRDs.  
  (Note: A Helm chart may be provided to install the Operator itself, but the Operator will not wrap the Bitnami chart.)
- Supports 3 modes:
  - Cluster: multi-node, HA, and scalable (primary production mode).
  - Sentinel: quorum-based HA with automatic failover for smaller-scale setups.
  - Standalone: single-node deployment (optionally with replicas) without Sentinel, intended for dev/test.
- Exposes Valkey connection info as a Kubernetes `Secret`.
- Handles TLS setup via cert-manager or self-signed certs.
- Optionally loads custom config via ConfigMap.
- Optional metrics sidecar for Prometheus scraping.

### Cluster mode
- Default and recommended mode for production deployments.
- Provides HA and scalability using native Valkey cluster features.
- Operator automates slot rebalancing, scaling, and rolling upgrades.

#### Design overview (Cluster mode)
- Multi-node deployment with slot-based sharding.
- Operator manages scaling, slot rebalancing, and rolling upgrades.
- Primaries and replicas distributed across nodes/availability zones for resilience.

### Sentinel mode
- Provides HA for smaller-scale deployments using Sentinel quorum-based failover.
- Sentinels run as decoupled StatefulSets for flexible scaling.
- Useful for organizations with existing Sentinel expertise.

#### Design overview (Sentinel mode)
- Each shard consists of one primary, N replicas, and a separate Sentinel StatefulSet.
- Sentinels monitor primaries and coordinate failover.
- Operator provisions dedicated Services for connection management and DNS discovery.
- Future: optional `Failover` CRD for controlled promotion of replicas.

### Standalone mode
- Simplest option, single primary (with optional replicas).
- No Sentinel or cluster slot management.
- Best suited for dev/test or lightweight environments.

#### Design overview (Standalone mode)
- Deploys a single StatefulSet with one primary pod.
- Optional replicas can be defined, but no automatic failover is provided.
- Upgrades and failover are manual (or rely on Kubernetes primitives like StatefulSet restarts).

Configuration

| Feature             | Config Options                                                                 |
|---------------------|--------------------------------------------------------------------------------|
| Deployment Mode     | `spec.mode: cluster|sentinel|standalone`                                       |
| TLS                 | `spec.tls.enabled: true` with cert-manager integration                         |
| Persistence         | `spec.persistence.type: rdb/aof/both`, `storageClass`, `size`                  |
| Modules             | `spec.modules`: list of `.so` modules to include                               |
| Monitoring          | `spec.metrics.enabled: true`                                                   |
| Custom Config       | `spec.config.configMapName`                                                    |
| Pod Customization   | node selectors, tolerations, resource limits, securityContext, etc.            |

Secret Output

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: valkey-test-binding
type: Opaque
stringData:
  host: valkey-test.svc
  port: "6379"
  password: ********
  sentinelHost: valkey-test.svc
  sentinelPort: "26379"
  tlsEnabled: "true"
```

### Authentication and Authorization

- No RBAC-specific changes are needed outside standard Kubernetes role management.
- Valkey credentials are stored in Kubernetes Secrets.
- TLS certs issued via cert-manager will follow existing RBAC rules.

### Appendix

- [ValkeyLDAP RFC as implementation reference]
- [Operator PRD (source for goals and user stories)]
- Valkey: k8s-operator comparison: https://docs.google.com/spreadsheets/d/1BiUkFmg0Iuo4vjK2yIN9O1b2KALZdqqJC6CqeZzIQiA/edit?usp=sharing 
