# WAS Scheduler Provisioning

## Functional Requirements

These requirements apply across the entire design and must be satisfied by the subsystems collectively. They don't all need to be satisfied by the initial design, but they need to be satisfiable eventually.

- **Scheduling-aware autoscaling:** Future changes to kube-scheduler must account for autoscaling as a first-class scheduling action, not an external side-effect.
- **Autoscaler-agnostic primitives:** The API primitives must be general enough that both CAS and Karpenter functionality can be implemented on top of them.
- **No scheduling logic in autoscalers:** Autoscalers must not need to implement or maintain their own scheduling algorithm. Scheduling decisions live entirely in kube-scheduler.
- **Strict fallback ordering:** Capacity providers must be able to express ordering constraints (e.g., "exhaust reserved before on-demand") that the scheduler respects independent of cost. These constraints apply to both scheduling onto existing nodes and provisioning new ones — the scheduler must prefer placing a pod on an existing reserved node before provisioning a new on-demand one.
- **Scheduling Action Policy:** Users can specify a preferred ordering of scheduling actions — provision, bind to existing capacity, or preempt — with fallback when the preferred action fails.

---

## Design Tree

```
Counter-Proposal
├── Potential Capacity API
│   ├── Hybrid via API Server (Recommended)
│   │   └── Overlay Composition (scheduler-facing)
│   ├── Flat via gRPC
│   │   └── Overlay Composition (cloud-provider-internal)
│   └── Offering Identity
├── Scheduling Policy
│   ├── Fallback Ordering Contract
│   └── Scheduling Action Policy
├── Pod Binding / Reservation
├── Scheduling Simulation API
├── Shared Autoscaling APIs
│   ├── Resource Limits (CapacityQuota)
│   └── Capacity Buffers (CapacityBuffer)
└── NodeClaim / CapacityRequest API (name pending)
    ├── Group Scheduling
    └── Feedback Loop / Stockout Handling
```

---

## Potential Capacity API

`potential-capacity-api-scratch.md`

Two scalable approaches, narrowed from a broader design space. Other approaches rejected for failing to scale (see Rejected Alternatives in scratch doc).

**Proposal 1 — Hybrid via API Server (Recommended, pending POC scale testing):**
- InstanceType CRDs with offerings encoding zone × capacityType × tenantConfig
- CapacityOverlay CRDs for shared deterministic adjustments (e.g., GPU driver overhead)
- Scheduler watches both object types and joins them in-memory
- Typical: 36 offerings per instance type
- Full end-user observability via `kubectl`

**Proposal 2 — Flat Offerings via gRPC:**
- All provisioning-time choices fully materialized as pre-resolved offerings
- Served to scheduler plugin over gRPC (avoids etcd/API server size limits)
- Zero composition work for scheduler — offerings arrive ready to use
- Overlays are cloud-provider-internal only
- Typical: 72 offerings per instance type
- Offerings opaque to end users

### Subsystems

**Overlay Composition** — How offerings are produced or modified from modular building blocks. Scheduler-facing in Hybrid, cloud-provider-internal in Flat. Needs its own design doc.

**Offering Identity** — How a NodeClaim/CapacityRequest unambiguously references which (InstanceType, Offering) tuple was selected. Requirements match or explicit ID. No design yet.

---

## Scheduling Policy

How the scheduler decides what action to take.

**Fallback Ordering Contract** — No design yet.
- Strict fallback (e.g., "exhaust CUD/reserved before on-demand") cannot be captured by cost ordering alone
- Needs a separate contract the scheduler respects independent of cost
- Must compose with user preferences and cloud provider overrides

**Scheduling Action Policy** — No design yet.
- User-configured preferred ordering of scheduling actions (provision, bind, preempt)
- Defines fallback behavior when the preferred action fails

---

## Pod Binding / Reservation

`pod-nodeclaim-binding-scratch.md`

The interface between the scheduler and the capacity API — how the scheduler reserves capacity for pods on not-yet-ready nodes.

- Scheduler sets `pod.spec.nodeClaimName` to reserve capacity
- New phase: `Pending → Reserved → Running`
- Kubelet ignores pods with `nodeClaimName` but no `nodeName`
- Reservation survives scheduler restart
- Open: promotion controller, reservation timeout, preemption semantics

---

## Shared Autoscaling APIs

APIs originating in sig-autoscaling (currently `autoscaling.x-k8s.io/v1alpha1` in CAS) that will need upstreaming into `capacity.k8s.io/v1` alongside the other primitives in this proposal. These are autoscaler-agnostic — they govern provisioning behavior without embedding scheduling logic, and must work with both CAS and Karpenter-style autoscalers.

**Resource Limits (CapacityQuota)** — `resource-limits-design-scratch.md`
- Cluster-scoped CRD that constrains aggregate provisioned resources (CPU, memory, node count, DRA device classes) across arbitrary subsets of nodes identified by label selector
- Multiple overlapping quotas all respected — provisioning blocked if any matching quota would be exceeded
- Accounts for in-flight NodeClaims to avoid over-provisioning during concurrent scheduling
- Must be consumable by the scheduler, since provisioning is a scheduling action in our model
- Prior art: sig-autoscaling CapacityQuota (`autoscaling.x-k8s.io/v1alpha1`), merged into CAS ([PR #8894](https://github.com/kubernetes/autoscaler/pull/8894))

**Capacity Buffers (CapacityBuffer)** — No design yet.
- Proactive buffer capacity provisioning — ensures spare capacity is available to handle burst workloads or speed up scaling events
- Currently implemented in CAS via placeholder pods: a CapacityBuffer references a PodTemplate (or a scalable resource) and maintains a desired number of buffer "chunks" that trigger the autoscaler to pre-provision nodes
- Supports fixed replica count, percentage-of-workload scaling, and resource-limit-based chunk calculation
- Prior art: sig-autoscaling CapacityBuffer (`autoscaling.x-k8s.io/v1alpha1`), merged into CAS
- Needs design work for upstreaming — the placeholder-pod mechanism is CAS-specific; the upstream API may need to express buffer intent differently given that provisioning is a scheduler action in our model
- Buffers and reservations must be respected during all scheduling calls — provisioning, placement on existing nodes, consolidation, and simulation. The scheduler must treat buffer demand as real demand that cannot be displaced. The design must address how buffer intent is represented to the scheduler (reservations, hypothetical pods, or another mechanism) so that it participates in scheduling decisions alongside real pending pods.

---

## NodeClaim / CapacityRequest API (name pending)

`nodeclaim-api-scratch.md`

The API for requesting and managing provisioned capacity. Post-provisioning lifecycle operations (drift detection, disruption, consolidation) may be built on top of this API in the future.

**Group Scheduling** — `gang-scheduling-design-scratch.md`
- NodeClaimGroup is a shared fate declaration with shared requirements, allocation strategy, and topology constraints
- NodeClaims reference a group via `spec.groupRef`; the creator creates both the group and the NodeClaims
- Groups can be nested for heterogeneous workloads (e.g., coordinator sub-group + worker sub-group under a parent group)
- Cloud provider owns group status and coordinates atomic provisioning

**Feedback Loop / Stockout Handling** — `feedback-loop-design-scratch.md`
- Cloud provider updates `offerings[].available = false` on stockout
- TTL-based recovery (3 minutes)
- NodeClaim has requirements (not specific instance type)
- Open: scheduler batching, failure propagation

---

## Scheduling Simulation API

Autoscalers like Karpenter implement disruption and drift — operations that require answering "if I remove this node, can its pods be placed elsewhere?" Today Karpenter maintains its own scheduling simulation to answer this. To satisfy the "no scheduling logic in autoscalers" requirement, the scheduler must expose a simulation API that autoscalers can query to evaluate hypothetical scheduling outcomes without maintaining their own scheduling algorithm. Consolidation and disruption are the primary use cases.

- Must support "what if" queries: given a set of pods and a modified cluster state, can they be scheduled?
- Must account for the full scheduling policy (fallback ordering, action preferences, topology constraints)
- Must be usable by any autoscaler, not Karpenter-specific
- **Performance isolation:** The simulation API must operate within a decoupled control loop or independent scheduler instance to preserve primary scheduling latency and throughput. Simulation is a high-volume, potentially expensive operation — it must not compete with the hot scheduling path for resources or introduce latency to real pod placement.
- No design yet.

---

## Design Docs Needed

- Overlay Composition
- Offering Identity
- Fallback Ordering Contract
- Scheduling Action Policy
- Scheduling Simulation API
- Capacity Buffers
- Migration Strategy
- Governance Model
