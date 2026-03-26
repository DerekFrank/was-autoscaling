# Node Group Scheduling — Design Scratch Notes

**Date:** March 26, 2026
**Context:** Atomic provisioning of multiple nodes for workloads that require co-scheduled capacity (e.g., distributed training, MPI jobs).

---

## Relevant Functional Requirements

- **Shared fate:** A set of nodes can be declared as sharing a fate — provisioned atomically, and potentially disrupted atomically in the future. Partial sets are useless capacity.
- **Individual node manageability:** After provisioning, individual nodes in the group must remain independently observable and manageable.
- **Heterogeneous groups:** Shared fate must apply across nodes with different requirements (e.g., 1 coordinator + N workers with different resource profiles).
- **Topology awareness:** Grouped workloads may require co-located nodes (same zone, same rack, same network spine) for performance. The group primitive must be able to express topology constraints across the group, not just per-node.

---

## What This API Must Express

1. **Shared fate declaration** — a single object that declares a set of nodes share a fate.
2. **Heterogeneous node requirements** — nodes in the group may have different requirements.
3. **Allocation strategy** — all-or-nothing, or a minimum count threshold below which the group is not satisfied.
4. **Topology constraints** — co-location requirements across the group (e.g., same zone, same rack), expressed as a topology label key that all nodes must share.
5. **Group membership** — a way to identify which NodeClaims belong to a group, for status aggregation and cascading deletion.

---

## Design Questions

### 1. Group Relationship Model

How do NodeClaims relate to a group?

**Decision: NodeClaims reference a group via `spec.groupRef`**

The group is a shared fate declaration with shared requirements and allocation strategy. NodeClaims are created independently by the same creator and reference the group. Each NodeClaim has its own requirements which must be compatible with the group's requirements. The cloud provider fulfills NodeClaims as usual, using the group reference to coordinate atomic provisioning.

**Rejected alternatives:**
- *Group creates NodeClaims from a template* — simpler for homogeneous groups but requires the group to own the NodeClaim spec. Heterogeneous groups require either multiple templates or a separate grouping mechanism. Pushes NodeClaim creation into the cloud provider rather than the creator.
- *`spec.replicas` on NodeClaim* — simpler for creation but individual nodes can't be independently addressed for status tracking, partial recovery, or observability. Embedding multiple providerIDs, nodeNames, and conditions in a single object is unwieldy.

### 2. Allocation Strategy

Not all grouped workloads require strict all-or-nothing semantics.

**Options (both supported):**
- **AllOrNothing** — all nodes referencing the group must be provisioned, or the group is not satisfied.
- **MinCount** — at least M nodes must be provisioned for the group to be considered satisfied.

### 3. Lifecycle Ownership

**Decision: Cloud provider owns group status**

The creator creates the group and the NodeClaims. The cloud provider watches NodeClaims that reference the group, fulfills them, and updates the group's status (total count, ready count, satisfied condition). This is consistent with the NodeClaim lifecycle model — the creator declares intent, the cloud provider fulfills and reports.

---

## Proposed Spec

### NodeClaimGroup

```yaml
apiVersion: capacity.k8s.io/v1
kind: NodeClaimGroup
metadata:
  name: training-job-xyz
spec:
  # Shared requirements — all NodeClaims referencing this group
  # must be compatible with these
  requirements:
    - key: topology.kubernetes.io/zone
      operator: In
      values: ["us-west-2a"]

  allocationStrategy: AllOrNothing  # or MinCount
  minCount: 2  # Only used with MinCount strategy

  # Topology co-location constraint
  # All nodes in this group must share the same value for this label
  topologyKey: topology.kubernetes.io/zone

status:
  # Updated by cloud provider
  totalCount: 3
  readyCount: 2
  conditions:
    - type: Satisfied
      status: "True"
```

### NodeClaims referencing the group

Created by the same creator. Each has its own requirements.

```yaml
# GPU worker node
apiVersion: capacity.k8s.io/v1
kind: NodeClaim
metadata:
  name: training-job-xyz-worker-0
  labels:
    node.kubernetes.io/instance-type: p4d.24xlarge
    topology.kubernetes.io/zone: us-west-2a
spec:
  groupRef:
    name: training-job-xyz
  requirements:
    - key: capacity.k8s.io/instance-gpu-count
      operator: Gte
      values: ["8"]
  resources:
    requests:
      nvidia.com/gpu: "8"
  providerConfigRef:
    group: aws.capacity.k8s.io
    kind: EC2NodeClass
    name: gpu-training
status:
  nodeName: "ip-10-0-1-100"
  providerID: "aws:///us-west-2a/i-abc001"
  phase: Ready
  conditions:
    - type: Launched
      status: "True"
    - type: Registered
      status: "True"
    - type: Ready
      status: "True"
```

### Heterogeneous groups via nesting

A NodeClaimGroup can reference a parent group, allowing hierarchical shared fate.
For example, a training job with 1 coordinator and 3 GPU workers:

```yaml
# Parent group — shared fate across all roles
apiVersion: capacity.k8s.io/v1
kind: NodeClaimGroup
metadata:
  name: training-job-xyz
spec:
  requirements:
    - key: topology.kubernetes.io/zone
      operator: In
      values: ["us-west-2a"]
  allocationStrategy: AllOrNothing
  topologyKey: topology.kubernetes.io/zone
---
# Coordinator sub-group
apiVersion: capacity.k8s.io/v1
kind: NodeClaimGroup
metadata:
  name: training-job-xyz-coordinator
spec:
  groupRef:
    name: training-job-xyz
  allocationStrategy: AllOrNothing
---
# Coordinator NodeClaim
apiVersion: capacity.k8s.io/v1
kind: NodeClaim
metadata:
  name: training-job-xyz-coordinator-0
spec:
  groupRef:
    name: training-job-xyz-coordinator
  requirements:
    - key: capacity.k8s.io/instance-cpu
      operator: Gte
      values: ["4"]
  resources:
    requests:
      cpu: "4"
      memory: "16Gi"
  providerConfigRef:
    group: aws.capacity.k8s.io
    kind: EC2NodeClass
    name: coordinator
---
# Worker sub-group
apiVersion: capacity.k8s.io/v1
kind: NodeClaimGroup
metadata:
  name: training-job-xyz-workers
spec:
  groupRef:
    name: training-job-xyz
  allocationStrategy: MinCount
  minCount: 2
---
# Worker NodeClaims (3 of these)
apiVersion: capacity.k8s.io/v1
kind: NodeClaim
metadata:
  name: training-job-xyz-worker-0
spec:
  groupRef:
    name: training-job-xyz-workers
  requirements:
    - key: capacity.k8s.io/instance-gpu-count
      operator: Gte
      values: ["8"]
  resources:
    requests:
      nvidia.com/gpu: "8"
  providerConfigRef:
    group: aws.capacity.k8s.io
    kind: EC2NodeClass
    name: gpu-training
```

### Flow

1. Creator creates the NodeClaimGroup (and sub-groups if heterogeneous)
2. Creator creates individual NodeClaims referencing the group
3. Cloud provider fulfills NodeClaims, using group references to coordinate atomic provisioning
4. Cloud provider updates group status (totalCount, readyCount, satisfied condition)
5. Deleting a NodeClaimGroup cascades to child groups and NodeClaims

---

## Open Questions

1. **Topology constraint spec** — is a single `topologyKey` sufficient, or do we need multiple keys (e.g., same zone AND same rack)? The KEP supports only a single constraint in alpha.
2. **Nesting depth** — should nesting be limited to one level (parent group + sub-groups), or arbitrary depth? One level likely covers all practical use cases.
3. **Group requirement inheritance** — how strictly must NodeClaim requirements be compatible with the group's shared requirements? Intersection? Superset?

---

## Related Design Notes

- **NodeClaim API:** See `nodeclaim-api-scratch.md` for the base NodeClaim spec
- **Feedback Loop:** See `feedback-loop-design-scratch.md` for stockout handling
- **Pod Binding:** See `pod-nodeclaim-binding-scratch.md` for how pods reserve against NodeClaims
