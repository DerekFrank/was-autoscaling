# Resource Limits — Design Scratch Notes

**Date:** March 26, 2026
**Context:** Part of the Shared Autoscaling APIs category. See `top-level-doc-scheduler-autoscaling.md` for overall design tree.

## Problem

Cluster operators need to constrain the total resources that autoscaling can provision — globally, per-team, per-zone, per-hardware-class, or across any arbitrary set of nodes. Without limits, a misconfigured workload or burst can scale a cluster to the cloud provider's account limits, generating unbounded cost.

## Requirements

1. Must constrain aggregate resources (cpu, memory, node count, DRA device classes) across arbitrary subsets of provisioned capacity
2. Subsets must be identified by label selector, matching against the labels/requirements of offerings and resulting nodes
3. Multiple overlapping quotas must all be respected — provisioning is blocked if any matching quota would be exceeded
4. Must account for in-flight capacity requests (NodeClaims not yet fulfilled) to avoid over-provisioning during concurrent scheduling
5. Best-effort is acceptable — stale caches and race conditions may cause transient violations
6. Must be consumable by the scheduler, since provisioning is a scheduling action in our model
7. Status should report current usage for observability

## Prior Art

The sig-autoscaling CapacityQuota API (`autoscaling.x-k8s.io/v1alpha1`) addresses this problem directly: a cluster-scoped CRD with a label selector and resource limits. It is merged into CAS and will need to be reworked for upstreaming into a `v1` API. Existing alternatives (Karpenter NodePool limits, Kubernetes ResourceQuota) are either too narrowly scoped or not applicable to node provisioning.

Reference: https://github.com/kubernetes/autoscaler/pull/8894

## Proposal

Upstream CapacityQuota into `capacity.k8s.io/v1` alongside the other objects in this proposal.

```yaml
apiVersion: capacity.k8s.io/v1
kind: CapacityQuota
metadata:
  name: team-ml-gpu-limit
spec:
  selector:
    matchLabels:
      team: ml
  limits:
    resources:
      cpu: 256
      memory: 1Ti
      examplegpu.deviceclass.resource.k8s.io/devices: 32
status:
  used:
    resources:
      cpu: 144
      memory: 576Gi
      examplegpu.deviceclass.resource.k8s.io/devices: 18
```

This example limits the ML team's provisioned capacity to 32 GPUs, 256 CPUs, and 1 TiB of memory across all nodes labeled `team: ml`. The status reports current usage. Device resources use the DRA device class naming convention (`<device-class>.deviceclass.resource.k8s.io/devices`) for consistency with DRA ResourceQuota.

### Separation of Concerns

The quota CRD must be observed by the scheduler, since provisioning is a scheduling action. It may also be observed by other controllers — for example, a consolidation controller could choose not to remove a node if doing so would trigger re-provisioning that approaches a quota limit. However, this is optional. The scheduler is the only required consumer; other controllers can use quotas to make better decisions but are not obligated to.

### DRA Integration

DRA ResourceQuota and CapacityQuota are complementary — they operate at different layers. DRA ResourceQuota limits how many devices a namespace can *request* (admission-time, per-namespace). CapacityQuota limits how many devices the cluster can *provision* (provisioning-time, cluster-wide). A cluster would use both: DRA ResourceQuota to constrain what teams can claim, CapacityQuota to constrain what the cluster can grow to.

To support DRA-managed devices, CapacityQuota's resource list uses the same naming convention as DRA ResourceQuota: `<device-class>.deviceclass.resource.k8s.io/devices`, as shown in the example above. This keeps resource naming consistent across the quota ecosystem.
