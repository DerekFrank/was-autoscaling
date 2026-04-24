# [Public] Scheduler-Integrated Node Autoscaling — Problem Decomposition

**Status:** Review
**Author:** DerekFrank
**Date:** 04/14/2026

**Goal:** Establish a shared understanding of how to decompose the scheduler-integrated autoscaling problem into tractable design areas. This document defines the problem categories and their boundaries, and proposes next steps for each — not the solutions. Each category can then be designed, reviewed, and iterated on semi-independently.

**Audience:** sig-autoscaling, sig-scheduling, sig-node, sig-cloud-provider, WG-node-lifecycle, WG-batch, WG-device-management.

---

## Context

At KubeCon EU 2026, sig-autoscaling aligned on a shared vision: **make kube-scheduler the primary component responsible for provisioning decisions**. This builds on the original [WAS Cluster Autoscaling proposal](google-proposals/was-cluster-autoscaling-proposal.md) and the [alignment discussions](google-proposals/was-node-autoscaling-alignment.md) between the CAS and Karpenter communities.

This document decomposes the problem space into categories, with next steps to drive each toward a design proposal.

---

## Shared Principles

These principles were either explicitly agreed upon in the alignment discussions or follow directly from the shared vision. They apply across all categories.

1. **Provisioning is a scheduling action.** The scheduler treats "provision new capacity" as a first-class action alongside "bind to existing node" and "preempt." It is not an external side-effect triggered by pending pods.

2. **Autoscaler-agnostic primitives.** The APIs and mechanisms must be general enough that both CAS-style and Karpenter-style autoscalers can be built on top of them. No API should assume a specific autoscaler architecture. Existing CAS and Karpenter users must have a viable migration path to the new system, including the ability to incrementally adopt new primitives without a hard cutover.

3. **No scheduling logic in autoscalers.** Autoscalers must not need to implement or maintain their own scheduling algorithm. The scheduler owns all placement and provisioning decisions. Autoscalers focus on lifecycle management (disruption, drift, consolidation) and capacity provider integration.

---

## Problem Categories

The alpha/beta split identifies which categories need design proposals for the initial milestone. Individual proposals may scope specific requirements to later graduation stages.

- **Alpha**
  - 1. Potential Capacity Discovery
  - 2. Scheduling Algorithm
  - 3. Capacity Request
  - 4. Node Lifecycle Tracking
  - 5. Scheduling Policy
- **Beta**
  - 6. Scheduling Simulation API
  - 7. Capacity Governance
  - 8. Gang Scheduling

### Project Next Steps

1. **Owner:** @DerekFrank, @towca — Start regular working group meetings with maintainers and contributors
2. **Owner:** _unassigned_ — Discuss and align on this proposal with initial stakeholders
3. **Owner:** _unassigned_ — Assign/choose owners for categories
4. Circulate with the wider SIG and working group audience:
   - [ ] **Owner:** _unassigned_ — sig-autoscaling
   - [ ] **Owner:** _unassigned_ — sig-scheduling
   - [ ] **Owner:** _unassigned_ — sig-node
   - [ ] **Owner:** _unassigned_ — sig-cloud-provider
   - [ ] **Owner:** _unassigned_ — WG-node-lifecycle
   - [ ] **Owner:** _unassigned_ — WG-batch
   - [ ] **Owner:** _unassigned_ — WG-device-management

---

### 1. Potential Capacity Discovery
*Interest groups: sig-autoscaling, sig-scheduling, sig-cloud-provider, WG-device-management*

**Proposed stage:** alpha

**Problem:** The scheduler needs to know what capacity *could* be provisioned — instance types, their resources, zones, purchase options, availability signals, and cost — without triggering any actual provisioning. Today this information is locked inside autoscaler implementations and opaque to the scheduler.

**Scope:**
- How the scheduler discovers available instance types, their properties, and their offerings (zone × purchase option × availability)
- How overlays and adjustments (GPU driver overhead, DaemonSet reservations, user-defined modifications) compose with base instance type definitions
- How availability is surfaced
- How offering cardinality is managed at scale (hundreds of instance types × dozens of offerings each)

**Requirements:**
- **Cardinality.** Major capacity providers expose hundreds of instance types. Combined with zone, purchase option, and overlay dimensions, the scheduler could face tens of thousands of offerings. Similarly, a stockout or zonal event or pricing update might mean that hundreds of instance types need to be modified. How do we handle that problem at scale?
- **Overlay composition.** Instance type properties are modified by multiple independent sources. The discovery mechanism must define how these compose.
- **Device capability representation.** The discovery mechanism must represent DRA device capabilities (attributes, capacity, partition structures) for potential nodes alongside traditional resources.

**Prior art and proposals:**
- The original WAS proposal defines `ClusterPotentialInstanceType` and `ClusterPotentialCapacityPool` CRDs with per-variation availability signals and cost
- Karpenter's internal `InstanceType` / `Offering` model with requirements-based constraint narrowing
- Two approaches under discussion: CRD-based (InstanceType + CapacityOverlay objects via API server) and gRPC-based (pre-resolved flat offerings served directly to the scheduler plugin)

**Follow-on concerns:** The discovery API must not preclude:
- Gang scheduling — group-level queries for co-provisionable offerings across topology domains
- Scheduling simulation — querying hypothetical capacity states (e.g., "what offerings remain if I remove this node?")

**Next steps:**
- **Owner:** @jmdeal — POC analysis of the cardinality and scale of potential solutions to inform the eventual proposal

---

### 2. Scheduling Algorithm
*Interest groups: sig-scheduling*

**Proposed stage:** alpha

**Problem:** The scheduler must decide *how* to pack pods onto potential capacity, *when* to commit to provisioning, and *how* to bind pods to capacity that doesn't yet exist. Today's kube-scheduler processes one pod at a time against existing nodes. Provisioning-aware scheduling requires reasoning about multiple pods simultaneously against capacity that doesn't exist yet, and reserving those pods against in-flight capacity requests until nodes are ready.

**Scope:**
- How the scheduler packs multiple pods onto a single virtual node (bin-packing / greedy packing)
- How the scheduler batches pods before committing to provisioning — the triggering mechanism, accumulation window, and cutoff criteria
- How the scheduler scores and selects among competing instance types and offerings (cost, waterfall fallback)
- How inter-pod scheduling constraints (topology, pod affinity) interact with in-flight capacity that hasn't resolved to a specific domain
- Pod-to-capacity binding: how the scheduler reserves pods against in-flight capacity requests before a node exists, how that reservation persists across scheduler restarts, and how it promotes to a real node binding
- How DRA device constraints interact with instance type selection and constraint narrowing during packing

**Requirements:**
- **Batching.** Pod-by-pod provisioning creates many small nodes instead of fewer efficient large ones. The scheduler must accumulate pods before dispatching provisioning requests. The batching mechanism (time-based delay, queue drain, pod count threshold) directly impacts both scheduling latency and provisioning efficiency.
- **Constraint narrowing.** A virtual node starts as a superposition of all compatible instance types. As pods are packed onto it, their requirements narrow the set of compatible types. The scheduler must track this narrowing efficiently and know when to stop packing and start a new virtual node.
- **Topology under uncertainty.** In-flight capacity may not yet be pinned to a specific topology domain (zone, rack). The scheduler must account for this uncertainty when evaluating topology spread constraints.
- **Pod binding gap.** Between "capacity requested" and "node ready," pods have no node to bind to. The reservation mechanism must persist across scheduler restarts, prevent duplicate provisioning, and handle timeout when capacity never arrives.

**Prior art and proposals:**
- Karpenter's greedy packing algorithm: start with all compatible types, narrow as pods are placed, finalize when no more fit
- The WAS alignment doc suggests a "slight delay in the provisioning cycle" for batch accumulation
- Hybrid topology counting: known topology values count normally, unknown values count as worst-case toward all possible values
- Proposed `pod.spec.nodeClaimName` field with a `Provisioning` phase: `Pending → Provisioning → Running`
- The original WAS proposal describes a two-phase binding process via "Potential Node Instances"

**Follow-on concerns:** The Scheduling Algorithm must not preclude:
- Gang scheduling — the packing algorithm must be extensible to multi-node packing where a group of pods must be placed across co-provisioned nodes (see Category 8)
- Scheduling policy — the algorithm must support pluggable action ordering (bind vs. provision vs. preempt) without hardcoding a single strategy (see Category 5)
- Scheduling simulation — the algorithm should be reusable by the simulation API so that hypothetical evaluations use the same packing logic (see Category 6)

**Next steps:**
- **Owner:** _unassigned_ — Prototype provisioning-aware scheduling within kube-scheduler, using Karpenter's greedy packing algorithm as a starting point, to validate feasibility of the batch → pack → dispatch flow
- **Owner:** _unassigned_ — Write a design proposal for adapting Karpenter's scheduling algorithm (greedy packing, constraint narrowing, batch-then-dispatch) into a generic provisioning-aware scheduling solution for kube-scheduler
- **Owner:** _unassigned_ — Design the pod-to-capacity binding mechanism (pod API changes, kubelet behavior, reservation persistence)

---

### 3. Capacity Request
*Interest groups: sig-autoscaling, sig-scheduling, sig-cloud-provider*

**Proposed stage:** alpha

**Problem:** The scheduler needs a standard API object to request new capacity from a capacity provider. This object is the contract between the scheduler (which declares *what it needs*) and the capacity provider (which *fulfills* the request). How the scheduler makes a request can be designed in parallel to the node lifecycle tracking discussions (Category 4).

**Scope:**
- The API for requesting provisioned capacity — what the scheduler specifies (requirements, not specific instance types) and what the capacity provider returns (specific instance, node registration)
- How requirements are expressed — i.e. label-selector constraints that narrow eligible offerings without prescribing a specific instance type
- How provider-specific configuration (AMI, network config) is referenced without baking provider concepts into the core API
- How requests can enforce flexibility from the capacity provider (Karpenter's minValues).

**Requirements:**
- **Flexible requests.** The scheduler should express *what it needs* (4 CPU, 16GB, zone us-west-2a, spot) not *what to buy* (m5.xlarge). The capacity provider selects the specific instance type, preserving the ability to optimize.
- **Provider configuration isolation.** Capacity providers require provider-specific configuration (AMI, network config, instance profiles) that doesn't belong in the core API. The request object must reference provider-specific configuration without baking provider concepts into the shared API surface.
- **General-purpose capacity primitive.** The API must be usable by any component that needs to request capacity — scheduler, autoscaler, disruption controller, or static provisioning tooling.
- **Fulfillment result contract.** The capacity provider must communicate what was actually provisioned back through the object — the resolved instance type, zone, capacity type, and actual allocatable resources. The requester needs to know what it got, and the mechanism for populating this information (labels, status fields) must be well-defined.

**Prior art and proposals:**
- Karpenter's `NodeClaim` CRD: requirements-based, capacity provider resolves to specific instance
- The original WAS proposal defines a `CapacityRequest` with fallback strategies and lifecycle tracking

**Follow-on concerns:** The Capacity Request API must not preclude:
- Gang scheduling — the object must be extensible to support group references and shared-fate declarations (see Category 8)
- Scheduling simulation — hypothetical capacity requests must be representable for "what-if" evaluations

**Next steps:**
- **Owner:** _unassigned_ — Write a design proposal for the capacity request API, addressing the requirements (flexible requests, provider config isolation, general-purpose usability, fulfillment result contract)

---

### 4. Node Lifecycle Tracking
*Interest groups: sig-autoscaling, sig-node, WG-node-lifecycle*

**Proposed stage:** alpha

**Problem:** Once a capacity request exists, the system needs to track the resulting node through its full lifecycle — from capacity provider fulfillment through node registration and eventually decommissioning. This is the status side of the capacity request: how the system communicates what was provisioned, what state it's in, and when capacity transitions between states.

**Scope:**
- The status contract on the capacity request object: phase transitions (Pending → Launched → Registered → Ready → Failed), conditions, and what each state means for consumers
- How lifecycle tracking extends beyond initial provisioning — whether this same status surface tracks drift, disruption, maintenance and decommissioning or whether those are separate concerns

**Requirements:**
- **Full-lifetime tracking.** The lifecycle primitive must track node state transitions throughout the full node lifetime — from initial provisioning through drift, disruption, maintenance, and decommissioning. Consumers need a consistent way to observe and react to these transitions regardless of what phase a node is in.
- **Static node support.** The lifecycle primitive must work for all nodes in the cluster, including nodes that were not dynamically provisioned (bare-metal, static VMs, edge nodes). A solution that only covers nodes created through a capacity request leaves a gap for statically managed infrastructure.

**Prior art and proposals:**
- Karpenter NodeClaim status tracks lifecycle conditions beyond provisioning: `Drifted`, `Consolidatable`, `Drained`, `InstanceTerminating`
- Existing Node conditions mechanism (Ready, MemoryPressure, DiskPressure, etc.)
- WG-node-lifecycle interest in a generic node lifecycle framework

**Follow-on concerns:** Node Lifecycle Tracking must not preclude:
- Gang scheduling — group-level lifecycle coordination, where failure of one node in a group may require rollback of the entire group (see Category 8)
- Scheduling simulation — the simulation API must be able to model in-flight capacity (nodes in provisioning state) to make accurate consolidation decisions (see Category 6)

**Next steps:**
- **Owner:** _unassigned_ — Gather and formally specify post-provisioning lifecycle use cases from WG-node-lifecycle
- **Owner:** _unassigned_ — Write a design proposal for how node state transitions (provisioning, drift, disruption, decommissioning) are tracked and surfaced, using Karpenter's NodeClaim as the starting point
- **Owner:** _unassigned_ — Evaluate lifecycle primitive placement options against gathered use cases

---

### 5. Scheduling Policy
*Interest groups: sig-scheduling, sig-autoscaling*

**Proposed stage:** alpha

**Problem:** When the scheduler has multiple options — bind to an existing node, provision new capacity, or preempt — it needs a policy to decide which action to take and in what order. Today there is no mechanism to express "try spot first, fall back to on-demand" or "prefer provisioning over preemption" as hard constraints that the scheduler respects.

**Scope:**
- How capacity providers express strict ordering between offerings (e.g., reserved → spot → on-demand) that cannot be overridden by cost or scoring
- How users express preferences over scheduling *actions* (provision vs. bind vs. preempt) with fallback behavior
- How users enforce flexibility in a capacity request (Karpenter's min values)
- How these policies compose — capacity provider ordering, user action preferences, and scheduler scoring must interact without conflict

**Requirements:**
- **Fallback ordering vs. cost.** Cost-based scoring can express "prefer cheaper options" but cannot enforce "never use on-demand if reserved capacity is available." Strict ordering requires a separate mechanism that the scheduler respects before cost enters the picture.
- **Action ordering.** Some workloads should prefer provisioning dedicated capacity over packing onto shared existing nodes. Others should prefer existing capacity to minimize latency. This is orthogonal to the offering ordering within a given action.
- **Composition.** Capacity providers, platform teams, and workload owners may all express policies. The system needs clear precedence rules for when these conflict.

**Prior art and proposals:**
- The original WAS proposal defines a "Pod Scheduling Preferences API" with tiered waterfall logic
- Karpenter NodePools encode some of this via weight and priority fields
- GKE Custom Compute Classes express intent ("Spot with On-Demand fallback") that gets translated into scheduling preferences

**Follow-on concerns:** Scheduling Policy must not preclude:
- Scheduling simulation — the simulation API must be able to evaluate hypothetical outcomes under different policy configurations (see Category 6)
- Capacity governance — policy fallback must compose with quota enforcement so that exhausting a preferred tier triggers fallback rather than quota violation (see Category 7)

**Next steps:**
- **Owner:** _unassigned_ — Gather concrete policy requirements from CAS and Karpenter maintainers — what ordering and fallback behaviors do users need today?
- **Owner:** _unassigned_ — Write a design proposal covering action ordering, fallback composition, and minValues

---

### 6. Scheduling Simulation API
*Interest groups: sig-scheduling, sig-autoscaling*

**Proposed stage:** beta

**Problem:** Autoscalers need to evaluate hypothetical scheduling outcomes — "if I remove this node, can its pods be rescheduled?" or "if I replace these three nodes with two larger ones, does everything still fit?" — without actually executing those changes. Today, both CAS and Karpenter maintain their own scheduling simulation to answer these questions, duplicating and diverging from kube-scheduler's actual logic. Consolidation and disruption are the primary use cases.

**Scope:**
- An API for querying hypothetical scheduling outcomes given a modified cluster state
- Modeling node removal, node replacement, and pod rescheduling across existing and potential capacity
- Accounting for the full scheduling policy (fallback ordering, action preferences, topology constraints, buffer requirements)
- Performance isolation from the primary scheduling path

**Requirements:**
- **Logic parity.** The simulation must use the *same* scheduling logic as the real scheduler — including all plugins, policies, and constraints.
- **Performance isolation.** Simulation is high-volume and potentially expensive (evaluating many hypothetical states). It must operate within a decoupled control loop or independent scheduler instance to preserve primary scheduling latency and throughput. It must not compete with the hot scheduling path.
- **Scope of hypotheticals.** The simulation must model not just real pods and nodes, but also buffer requirements and reservations. Consolidation decisions that violate buffer invariants are unsafe even if all real pods can be rescheduled.

**Prior art and proposals:**
- Karpenter's internal scheduling simulation for disruption and consolidation
- CAS's internal node group simulation model
- The WAS alignment doc specifies this should run in a "decoupled control loop or independent scheduler instance"

**Follow-on concerns:** Scheduling Simulation API must not preclude:
- Capacity governance — simulations must be able to account for buffer requirements and resource limits so that consolidation decisions respect governance invariants (see Category 7)
- Scheduling policy — simulations must respect the same policy rules (action ordering, fallback logic) as the real scheduling path (see Category 5)

**Next steps:**
- **Owner:** _unassigned_ — Define the API surface autoscalers need for consolidation and disruption decisions — what queries, what inputs, what outputs
- **Owner:** _unassigned_ — Begin early design discussions with sig-scheduling on how to expose scheduler logic for simulation

---

### 7. Capacity Governance
*Interest groups: sig-autoscaling*

**Proposed stage:** beta

**Problem:** Cluster operators need mechanisms to constrain and shape provisioning behavior — limiting total provisioned resources, maintaining buffer capacity for burst handling, and enforcing these constraints across all scheduling decisions (not just provisioning). These are autoscaler-agnostic concerns that apply regardless of whether CAS or Karpenter is managing the cluster.

**Scope:**
- **Resource limits:** Constraining aggregate provisioned resources (CPU, memory, node count, device classes) across arbitrary subsets of nodes. Multiple overlapping limits must all be respected. In-flight provisioning requests must count against limits to prevent over-provisioning during concurrent scheduling.
- **Capacity buffers:** Maintaining spare capacity to absorb burst workloads or speed up scaling. Buffer intent must be represented to the scheduler so that it participates in all scheduling decisions — provisioning, placement, consolidation, and simulation. Buffers and reservations must be respected during all scheduling calls; the scheduler must treat buffer demand as real demand that cannot be displaced.

**Requirements:**
- **Buffer representation.** The current CAS implementation uses placeholder pods to trigger provisioning. In a scheduler-integrated model where provisioning is a scheduling action, buffer intent may need a different representation — reservations, hypothetical pods, or a first-class scheduler concept. The mechanism must ensure buffers are visible to the simulation API so that consolidation decisions respect buffer invariants.
- **Limit enforcement timing.** Limits must be checked at provisioning decision time, not just after nodes exist. With concurrent scheduling, multiple provisioning decisions may be in flight — the system must account for in-flight capacity to avoid exceeding limits.
- **Governance scope.** These mechanisms must work with DRA device classes (not just CPU/memory), support label-selector-based scoping (not just global limits), and compose with the scheduling policy (a provisioning decision that would exceed a limit must fall back, not fail silently).

**Prior art and proposals:**
- sig-autoscaling `CapacityQuota` (`autoscaling.x-k8s.io/v1alpha1`): cluster-scoped, label-selector-based resource limits
- sig-autoscaling `CapacityBuffer` (`autoscaling.x-k8s.io/v1alpha1`): buffer provisioning via placeholder pods or scaled workloads
- Karpenter NodePool `limits` field for per-pool resource budgets

**Follow-on concerns:** Capacity Governance must not preclude:
- Scheduling simulation — buffers and limits must be visible to the simulation API so that consolidation decisions don't violate governance constraints (see Category 6)
- Scheduling policy — a provisioning decision that would exceed a quota must trigger policy fallback, not silent failure (see Category 5)

**Next steps:**
- **Owner:** _unassigned_ — Monitor existing CapacityQuota and CapacityBuffer proposals in sig-autoscaling and ensure alignment with the broader architecture

---

### 8. Gang Scheduling
*Interest groups: sig-scheduling, sig-autoscaling, WG-batch*
*Depends on: Capacity Request, Node Lifecycle Tracking*

**Proposed stage:** beta

**Problem:** Distributed training, HPC, and other tightly coupled workloads require atomic provisioning of multiple nodes — either all nodes in a group are provisioned, or none are. Partial provisioning leads to resource deadlocks where expensive capacity sits idle waiting for the rest of the group. This problem spans both the request API (declaring shared fate) and the lifecycle tracking (coordinating fulfillment and handling partial failure).

**Scope:**
- How shared fate is declared across multiple capacity requests — a group primitive that expresses "these requests must succeed or fail together"
- Heterogeneous groups: nodes in the group may have different requirements (e.g., 1 coordinator + N GPU workers)
- Allocation strategies: all-or-nothing vs. minimum count thresholds
- Topology constraints across the group — co-location requirements (same zone, same rack) that apply to the group as a whole, not just individual nodes
- Failure and rollback semantics: what happens when some nodes in a group provision successfully but others fail

**Requirements:**
- **Atomic rollback.** If 7 of 8 nodes provision successfully but the 8th fails, the system must roll back the successful 7 to avoid stranded capacity. This rollback must be coordinated with the capacity provider and any pods already reserved against the successful nodes.
- **Topology co-location.** Grouped workloads often require nodes in the same topology domain (zone, rack, network spine) for performance. The group primitive must express this constraint so the capacity provider can co-locate the nodes, but the specific topology value may not be known until provisioning time.
- **Heterogeneous requirements.** A single group may contain nodes with fundamentally different shapes (CPU coordinator + GPU workers). The group must support different requirements per member while maintaining shared fate.

**Prior art and proposals:**
- `NodeClaimGroup` for gang provisioning with AllOrNothing/MinCount strategies and nested heterogeneous groups
- The original WAS proposal defines atomic `CapacityRequest` with gang scheduling support
- Karpenter does not currently support gang provisioning

**Next steps:**
- **Owner:** _unassigned_ — Monitor Capacity Request and Node Lifecycle Tracking designs to ensure they don't preclude gang semantics
- **Owner:** _unassigned_ — Engage WG-batch on gang provisioning requirements from the Kueue/JobSet perspective 
