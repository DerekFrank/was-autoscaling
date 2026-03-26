# Potential Capacity API - Design Scratch Notes

**Date:** March 25, 2026
**Context:** Counter-proposal to the WAS (Workload Aware Scheduler) Potential Capacity API design

TODO: Call out CEL expressions in overlays and postpone design for a follow up KEP.

## Goals

1. Move scheduling decisions into kube-scheduler (deprecate Karpenter's internal scheduler)
2. Keep a thin cloud-provider actuator for node creation
3. Scheduler consumes a provider-agnostic capacity API without knowledge of provider-specific constructs

---

## Provisioning-Time Choices

When provisioning a new node, several choices must be made that affect the resulting node's properties:

| Use Case | Choice | Affects |
|----------|--------|---------|
| Multi-tenancy isolation | Tenant config (labels/taints) | Scheduling constraints, cost attribution, security boundaries |
| Workload reliability | Capacity type (spot/on-demand/reserved) | Interruption tolerance, availability guarantees |
| Zonal resiliency | Zone | Fault tolerance, latency, data locality, compliance |
| OS/runtime selection | AMI family | Allocatable capacity, workload compatibility |
| Resource reservations | Kubelet config | System/kube reserved, eviction thresholds, max pods |
| Optional devices | Attached accelerators (GPUs, TPUs) | Available resources, cost, workload compatibility |
| Hardware configuration | Device/driver settings | GPU overhead, instance store, network performance |
| Hard fallback ordering | Capacity type preference | Cloud providers enforce "exhaust CUD before on-demand" regardless of user preferences |

---

## Core Concepts

### InstanceType
The base hardware specification as reported by the cloud provider:
- Name (e.g., m5.large, n2-standard-8)
- Raw capacity (CPU, memory, pods, storage)
- Static requirements (architecture, instance family)

### Offering
A purchasable configuration representing a specific combination of choices. Each offering includes:
- Requirements (labels the resulting node will have)
- Taints (taints the resulting node will have)
- Allocatable capacity (after all adjustments, including optional devices)
- Cost (optimization metric)
- Availability (can this be purchased now)
- Optional devices (attached accelerators like GPUs/TPUs that modify capacity and cost)

### Overlay
A transformation applied to compute final node properties from base offerings. In the Hybrid approach (Proposal 1), overlays are part of the scheduler-facing API surface — the scheduler must watch and apply them to derive usable capacity data. In the Flat approach (Proposal 2), overlays are a cloud-provider-internal construct used to produce fully resolved offerings before they reach the scheduler. See the overlay design doc for composition semantics.

### Fallback Ordering
Strict fallback ordering (e.g., "exhaust CUD/reserved capacity before on-demand") cannot be sufficiently captured by cost ordering alone. A separate contract is needed to express ordering constraints that the scheduler must respect independent of cost. The design for this contract is TBD.

---

## Proposed Approaches

Both proposals below are the result of narrowing from a broader design space. They represent the only two approaches that can scale to production cluster sizes. See [Rejected Alternatives](#rejected-alternatives) for approaches that were considered and eliminated.

### Proposal 1: Hybrid via API Server (Recommended, pending POC scale testing)

Offerings encode choices where availability and cost depend on the combination (zone × capacityType × tenantConfig). Additive CapacityOverlay CRDs handle shared deterministic adjustments (e.g., GPU driver overhead). AMI/kubelet config is baked into offerings because it is typically tied to tenant configuration.

**Motivation:** InstanceType and CapacityOverlay are standard Kubernetes objects, inspectable via `kubectl`, watches, existing monitoring, and RBAC. End users can directly list and describe InstanceTypes to understand what capacity is available and at what cost.

```yaml
apiVersion: capacity.k8s.io/v1
kind: InstanceType
metadata:
  name: m5.large
spec:
  capacity:
    cpu: "2"
    memory: "8Gi"
  requirements:
    - key: kubernetes.io/arch
      operator: In
      values: ["amd64"]
status:
  offerings:
    # Each offering is: zone × capacityType × tenantConfig (includes AMI choice)
    - requirements:
        - key: topology.kubernetes.io/zone
          operator: In
          values: ["us-west-2a"]
        - key: capacity.k8s.io/capacity-type
          operator: In
          values: ["on-demand"]
        - key: team
          operator: In
          values: ["frontend"]
        - key: capacity.k8s.io/ami-family
          operator: In
          values: ["al2023"]
      # Base allocatable (before additive overlays)
      allocatable:
        cpu: "1980m"
        memory: "7950Mi"
      cost: "0.096"
      available: true
---
# Additive overlay for GPU driver (applies to GPU instances)
apiVersion: capacity.k8s.io/v1
kind: CapacityOverlay
metadata:
  name: gpu-driver-overhead
spec:
  requirements:
    - key: capacity.k8s.io/instance-gpu-count
      operator: Gt
      values: ["0"]
  capacityAdjustment:
    memory: "-500Mi"
```

**API consumption:** The scheduler watches InstanceType objects via the API server and separately watches CapacityOverlay objects. To derive usable capacity data, the scheduler must join these two sources: for each offering on an InstanceType, it applies all matching additive overlays to compute the final allocatable resources. This requires the scheduler to understand overlay matching semantics and maintain an in-memory index of overlays.

**Pros:**
- Offerings handle combination-specific properties (availability, base cost)
- Overlays handle shared adjustments (GPU driver overhead)
- Invalid combinations are implicit (not published as offerings)
- Moderate offering count: `zones × capacityTypes × tenantConfigs`
- Typical: 6 × 2 × 3 = 36 offerings per instance type
- Full observability — end users can `kubectl get instancetypes` to inspect available capacity

**Cons:**
- Scheduler must watch two object types (InstanceTypes + CapacityOverlays) and join them in-memory
- Scheduler must understand overlay matching and application semantics
- Two concepts to understand (offerings + overlays)

---

### Proposal 2: Flat Offerings via gRPC

All provisioning-time choices are fully materialized as offerings. The cloud provider computes the cross-product of valid combinations and serves them to the scheduler plugin over a gRPC endpoint. This avoids the etcd and API server object size limits that make a fully flattened approach infeasible via CRDs.

**Motivation:** Simplest model to reason about — every purchasable option is a single, fully resolved offering with no composition required. Overlays are a cloud-provider-internal concern used to produce the flattened offerings before they reach the scheduler; see the overlay design doc.

**Prior art:** Scheduler plugins already consume external data via gRPC (e.g., device plugins, DRA).

```yaml
# Conceptual offering structure served via gRPC
# (not a CRD — not stored in etcd)

instanceType: m5.large
capacity:
  cpu: "2"
  memory: "8Gi"
requirements:
  - key: kubernetes.io/arch
    operator: In
    values: ["amd64"]
  - key: node.kubernetes.io/instance-type
    operator: In
    values: ["m5.large"]
offerings:
  # Team frontend, AL2023, us-west-2a, on-demand
  - requirements:
      - key: topology.kubernetes.io/zone
        operator: In
        values: ["us-west-2a"]
      - key: capacity.k8s.io/capacity-type
        operator: In
        values: ["on-demand"]
      - key: team
        operator: In
        values: ["frontend"]
      - key: capacity.k8s.io/ami-family
        operator: In
        values: ["al2023"]
    allocatable:
      cpu: "1930m"
      memory: "7900Mi"
    cost: "0.096"
    available: true
  
  # Team frontend, AL2023, us-west-2a, spot
  - requirements:
      - key: topology.kubernetes.io/zone
        operator: In
        values: ["us-west-2a"]
      - key: capacity.k8s.io/capacity-type
        operator: In
        values: ["spot"]
      - key: team
        operator: In
        values: ["frontend"]
      - key: capacity.k8s.io/ami-family
        operator: In
        values: ["al2023"]
    allocatable:
      cpu: "1930m"
      memory: "7900Mi"
    cost: "0.035"
    available: true
  
  # Team backend, Bottlerocket, us-west-2a, on-demand
  - requirements:
      - key: topology.kubernetes.io/zone
        operator: In
        values: ["us-west-2a"]
      - key: capacity.k8s.io/capacity-type
        operator: In
        values: ["on-demand"]
      - key: team
        operator: In
        values: ["backend"]
      - key: capacity.k8s.io/ami-family
        operator: In
        values: ["bottlerocket"]
    allocatable:
      cpu: "1950m"
      memory: "8000Mi"
    cost: "0.096"
    available: true
  # ... more offerings
```

**API consumption:** The scheduler plugin queries a gRPC endpoint to receive fully resolved offerings. No additional composition or joining is required — offerings arrive ready to use with final allocatable resources, cost, and availability already computed. The scheduler does not need to understand overlays or any other intermediate construct.

**Pros:**
- Zero composition work for the scheduler — offerings are pre-resolved
- Invalid combinations are implicit (not served)
- Single concept (offerings only), no overlay semantics in the scheduler
- No etcd size constraints — gRPC can serve arbitrarily large offering sets
- Prior art in scheduler plugin ecosystem

**Cons:**
- High offering count: `zones × capacityTypes × tenantConfigs × amiChoices`
- Typical: 6 × 2 × 3 × 2 = 72 offerings per instance type
- Worst case: 6 × 3 × 10 × 3 = 540 offerings per instance type
- Offerings are opaque to end users — not inspectable via `kubectl` or standard Kubernetes tooling
- Requires custom debugging/introspection tooling to understand available capacity
- Introduces a non-Kubernetes dependency in the scheduling path

---

## Offering Structure Detail

```yaml
# Full offering structure
- requirements:
    # Scheduling constraints (pod must tolerate/select these)
    - key: topology.kubernetes.io/zone
      operator: In
      values: ["us-west-2a"]
    - key: capacity.k8s.io/capacity-type
      operator: In
      values: ["spot"]
    # Tenant labels (applied to node)
    - key: team
      operator: In
      values: ["frontend"]
    - key: cost-center
      operator: In
      values: ["eng-123"]
    # Configuration labels
    - key: capacity.k8s.io/ami-family
      operator: In
      values: ["al2023"]
  
  # Taints applied to the node
  taints:
    - key: team
      value: frontend
      effect: NoSchedule
  
  # Allocatable after all adjustments
  allocatable:
    cpu: "1930m"
    memory: "7900Mi"
    pods: "29"
  
  # Cost metric (lower is better)
  # Single optimization dimension
  cost: "0.035"
  
  # Current availability
  available: true
  
  # For reserved capacity
  reservationCapacity: 5
  reservationId: "cr-12345678"
```

---

## Rejected Alternatives

### Fully Flattened Offerings via API Server

The flat offering model (all provisioning-time choices materialized as offerings) stored as CRDs in etcd. This is the same model as Proposal 2 but transported via the Kubernetes API server instead of gRPC.

**Rejected because:**
- High offering count per InstanceType (typical: 72, worst case: 540)
- Each offering adds ~400 bytes to the InstanceType object
- At 700 instance types, total API footprint reaches ~21MB+, exceeding practical etcd object size limits
- API server watch bandwidth becomes prohibitive at scale

### Pure Choice Overlays

All provisioning-time choices modeled as grouped overlays. The scheduler computes the cross-product at scheduling time. No offerings on the InstanceType at all.

**Rejected because:**
- Cross-group dependencies are impossible to model correctly (price depends on zone + capacityType + instanceType simultaneously)
- Availability is per-combination, not per-choice — ICE marks a specific (zone, capacityType, instanceType) tuple unavailable, which can't be expressed on an individual overlay
- Debugging requires mental cross-product computation across all overlay groups

---

## Open Questions

### Deferred Topics
1. **Resource Limits** - How to express "these offerings share a CPU/memory budget"
2. **Disruption Policy** - Consolidation settings, drain behavior
3. **Provenance** - Tracing which user-facing config produced an offering

### API Design
4. **Offering Identity** - How does NodeClaim reference a specific offering? Requirements match or explicit ID?
5. **Partial Updates** - When one tenant config changes, update all ITs or just affected?
6. **ICE Granularity** - Does ICE affect all tenant configs for a (zone, capacityType, instanceType) or just the one that failed?

### Fallback Ordering
7. **Fallback Contract** - What is the contract for strict fallback ordering? How does the scheduler know to exhaust reserved/CUD capacity before on-demand, independent of cost?

---

## Related Design Notes

- **Overlay Design:** See overlay design doc for how cloud providers compose offerings from modular overlays
- **Feedback Loop Design:** See `feedback-loop-design-scratch.md` for ICE handling
- **Gang Scheduling:** See `gang-scheduling-design-scratch.md` for NodeClaimGang design
- **Pod Binding:** See `pod-nodeclaim-binding-scratch.md` for reservation mechanism
