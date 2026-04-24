# Scheduler as a Library — An Alternative Architecture for Scheduler-Integrated Autoscaling

**Status:** Draft
**Author:** DerekFrank
**Date:** 04/13/2026

---

## Executive Summary

The Cluster Autoscaler and Karpenter maintainers have aligned on the need to integrate node autoscaling with kube-scheduler ([WAS Cluster Autoscaling](https://docs.google.com/document/d/1ergKWH28EpGyYVISZbqPqIBS1wN7VNhjNa_Sl1pAFI8/edit?tab=t.77ykkkwdq8vp#heading=h.z57mquggx1b6), [Node Autoscaling Alignment](https://docs.google.com/document/d/1syaoTkK1E6aWhxTjY-AZfDt5tViJr5_6_3w8W-y1pg8/edit?resourcekey=0-Me75lgJQLdjVKSgqou80xQ&tab=t.0)). Previous discussions identified two core problems the maintainers want to solve: a split-brain architecture leading to inconsistent, racy behavior, and the development overhead of maintaining a scheduler. Those discussions focused on a specific architecture — the scheduler gains provisioning as a first-class action (**Scheduler as an Autoscaler**). This document proposes an alternative — **Scheduler as a Library**.

---

## Requirements

These requirements are defined in the [Scheduler-Driven Autoscaling PRD](https://docs.google.com/document/d/1OMNKlGcdml4-ORMeCQpq_vWAKqQkIjVkHheUS0jf6og/edit?tab=t.0#heading=h.w1ntht2jyv50) and apply regardless of architecture.

### Problems

1. Split brain: Inconsistent Pod placement decisions
2. Costly kube-scheduler integration maintenance
3. No compatibility with custom schedulers
4. Interesting features are blocked by the current interaction setup
5. Tight coupling between scheduling and autoscaling is not apparent from the scheduling side

### Solution Requirements

1. Node autoscalers stop simulating scheduling completely
2. Existing Node autoscaling behavior is preserved
3. Existing Node autoscaling performance is preserved
4. Existing Pod scheduling performance is preserved

---

## Proposed Architecture: Scheduler as a Library

Solving the split-brain problem requires more than shared logic — it requires shared state. Two processes making independent decisions against their own views of the cluster will diverge, regardless of whether they implement the same algorithms. Consistency requires a single orchestrator that owns an authoritative cluster view and makes all scheduling and orchestration decisions against it.

This proposal introduces a single orchestrator that owns pod binding, node lifecycle management, consolidation/defrag, and provisioning. It consumes scheduling logic through a standard interface.

### The Scheduling Interface

```
Schedule(
    clusterState      ClusterState,
    pods              []Pod,
    podGroups         []PodGroup,
    potentialCapacity []InstanceOffering,
) → (
    podResults      []PodResult,
    nodeClaims      []NodeClaim,
    nodeClaimGroups []NodeClaimGroup,
)
```

The orchestrator calls this interface for every operation that requires scheduling logic. Provisioning, consolidation, drift, expiration, and maintenance all reduce to the same call with different inputs:

- **Provisioning.** The orchestrator passes pending pods, current cluster state, and the full offering set. The library returns pod results — which may include bindings, preemptions, and provisioning decisions — along with nodeclaims for pods that require new capacity.
- **Consolidation.** The orchestrator selects a candidate node for removal, constructs a cluster state with that node removed, and passes the displaced pods. The result tells the orchestrator whether the pods can be placed elsewhere and what the replacement cost looks like.
- **Drift / Expiration / Maintenance.** Same pattern as consolidation — the orchestrator removes the target node from the cluster state and evaluates placement of displaced pods.

The detailed type definitions for `ClusterState`, `PodGroup`, `NodeClaimGroup`, and others are follow-up design work. The interface shape — a single function that takes the full scheduling context and returns placement decisions — is the proposal.

---

## Properties

### Solves: Split brain — Inconsistent Pod placement decisions

One orchestrator, one source of scheduling logic, shared state in one process. Provisioning, binding, and lifecycle decisions are guaranteed consistent — not through coordination protocols, but because they share the same authoritative `ClusterState`.

### Solves: Costly kube-scheduler integration maintenance

Scheduling logic is implemented once in the library. Autoscalers consume it rather than reimplement it. New scheduling features (DRA, topology constraints, etc.) are added to the library and immediately available to every orchestrator — no per-autoscaler adaptation required.

### Solves: No compatibility with custom schedulers

The `Schedule` interface is a standard contract. Custom schedulers implement it. The orchestrator invokes whichever implementation matches the pod's `schedulerName`. Custom scheduler support is a property of the interface, not a per-autoscaler integration effort.

### Solves: Interesting features are blocked by the current interaction setup

Because the orchestrator owns lifecycle context (drift, expiration, maintenance) and passes it via `ClusterState`, scheduling decisions can account for the full picture. Features like "prefer new cheaper nodes over existing expensive ones" or "avoid scheduling on nodes about to be disrupted" become possible through the same interface without new coordination mechanisms.

### Solves: Tight coupling between scheduling and autoscaling is not apparent from the scheduling side

The library interface makes the dependency between scheduling and autoscaling explicit — it is a versioned, designed contract. New scheduling features must accommodate the library interface, making the coupling visible and deliberate rather than implicit.

### Solves: Node autoscalers stop simulating scheduling completely

Autoscalers invoke the library instead of simulating. The `Schedule` function *is* the scheduling logic — not an approximation of it.

### Solves: Existing Node autoscaling behavior is preserved

The orchestrator retains full control over lifecycle management — consolidation strategies, drift detection, disruption budgets. The library replaces only the scheduling simulation. CAS and Karpenter can implement different orchestrators around the same library, preserving their respective user-facing behaviors.

### Solves: Existing Node autoscaling performance is preserved

In-process integration eliminates cross-process coordination overhead. The library is invoked as a function call with shared state — no serialization, no API server round-trips for scheduling evaluation.

### Solves: Existing Pod scheduling performance is preserved

In static clusters, kube-scheduler continues to operate as it does today. In autoscaled clusters, pod scheduling performance depends on the library's implementation — which uses the same algorithms as kube-scheduler. The `ClusterState` pre-computed indexes support efficient filter/score evaluation at scale.

---

## Design Work

### ClusterState

**Requirements:**

- **Authoritative.** The single source of truth for all scheduling decisions within a `Schedule` call — this is the structural guarantee that eliminates the split-brain.
- **Complete.** Must contain all information any scheduler implementation needs to make placement decisions — nodes, pods, resource state, topology, DRA state, persistent volume state, in-flight NodeClaims, and namespace metadata.
- **Pre-computed indexes.** The library needs efficient lookups during filter/score evaluation — affinity matching, topology spread counting, resource availability per node, and taint/selector matching.
- **Modifiable.** The orchestrator must be able to construct hypothetical cluster states for consolidation and drift evaluation — "what does the cluster look like if I remove this node?"
- **Extensible.** Custom schedulers may need additional state or indexes beyond what the default implementation provides.

**Prior Art:**

- kube-scheduler's ClusterSnapshot
- Karpenter's cluster.go

### PodResult

**Requirements:**

- **Bindings.** Must express which pod goes on which node — either an existing node or a new NodeClaim.
- **Preemptions.** Must express which pods are evicted to make a placement possible. TODO: deep dive on preemption to iron out the full set of requirements and issues — including preemption chain depth (full resolution vs. one level), relationship between input and output pod counts, and execution ordering dependencies.
- **Unschedulable errors.** Must distinguish between retryable (e.g., transient resource pressure, in-flight NodeClaim not yet ready) and unretryable (e.g., no offering satisfies the pod's constraints) failures.
- **Partial success.** The library must be able to place some pods and return unschedulable errors for others within the same call.

**Prior Art:**

- Karpenter's SchedulingResult
- TODO: check kube-scheduler for equivalent

### NodeClaim

**Requirements:**

- **Requirements-based.** Express what the scheduler needs (CPU, memory, zone, capacity type), not a specific instance type. The capacity provider selects the concrete offering.
- **Provider configuration isolation.** Reference provider-specific configuration (OS image, network config, instance profiles) without baking provider concepts into the core API.
- **Fulfillment result contract.** The capacity provider must communicate what was actually provisioned — resolved instance type, zone, capacity type, and actual allocatable resources.
- **General-purpose.** Usable by any component that needs to request capacity — not only the orchestrator, but also static provisioning tooling or external controllers.
- **Full-lifetime tracking.** Track node state transitions throughout the full lifetime — from provisioning through drift, disruption, maintenance, and decommissioning.
- **Static node support.** Must represent all nodes in the cluster, including those not dynamically provisioned (bare-metal, static VMs, edge nodes).
- **Simulatable.** Hypothetical NodeClaims must be representable for what-if evaluations — the orchestrator needs to model in-flight capacity when constructing ClusterState.

**Prior Art:**

- Karpenter's NodeClaim CRD
- The original WAS proposal's CapacityRequest
- Existing Node conditions mechanism (Ready, MemoryPressure, DiskPressure, etc.)

### NodeClaimGroup

**Requirements:**

- **Shared fate.** All NodeClaims in the group succeed or fail together — during initial provisioning, and during ongoing lifecycle operations (disruption, consolidation, patching). If one member is disrupted, the group's shared fate semantics must be respected.
- **Atomic rollback.** If some NodeClaims in a group provision successfully but others fail, the system must roll back the successful ones to avoid stranded capacity.
- **Partial fulfillment.** Must support minimum count thresholds — e.g., at least N of M NodeClaims must succeed for the group to be considered fulfilled.
- **Hierarchical topology.** All nodes in a group must fulfill specified topology requirements — e.g., all nodes must be in the same zone, or the same rack. The specific topology value may not be known until provisioning time.
- **Heterogeneous requirements.** A single group may contain NodeClaims with different shapes (e.g., 1 CPU coordinator + N GPU workers). Must support NodeClaimGroups of NodeClaimGroups for nested heterogeneous groups.

**Prior Art:**

- PodGroup

### InstanceOffering / PotentialCapacity

**Requirements:**

- **Cardinality management.** The library must efficiently evaluate tens of thousands of offerings during filter/score — hundreds of instance types across zones, purchase options, and availability states.
- **Device capability representation.** Must represent DRA device capabilities (attributes, capacity, partition structures) alongside traditional compute resources.
- **Availability signals.** Offerings must carry availability information so the library can avoid scheduling against stocked-out or degraded capacity.
- **Cost.** Offerings must carry cost information so the library can optimize placement for cost efficiency.
- **Provider integration.** How the orchestrator integrates with capacity providers to construct offerings is up to each autoscaler implementation. The type must be flexible enough to accommodate any provider integration.

**Prior Art:**

- Karpenter's InstanceType / Offering model
- The original WAS proposal's `ClusterPotentialInstanceType` and `ClusterPotentialCapacityPool` CRDs

### Scheduling Framework Refactor

The `Schedule` function is kube-scheduler's scheduling cycle (PreFilter → Filter → PostFilter → PreScore → Score), extended with batching, virtual nodes from potential capacity, and NodeClaim creation integrated into the pipeline.

**Requirements:**

- **Batching.** The pipeline must support evaluating multiple pods in a single call, with inter-pod accounting — pod A's placement affects pod B's feasibility. Pod-by-pod provisioning creates many small nodes instead of fewer efficient large ones.
- **Constraint narrowing.** A virtual node starts as a superposition of all compatible instance types. As pods are packed onto it, their requirements narrow the set of compatible types. The library must track this narrowing efficiently and know when to stop packing and start a new virtual node.
- **Topology under uncertainty.** In-flight capacity may not yet be pinned to a specific topology domain (zone, rack). The library must account for this uncertainty when evaluating topology spread constraints.
- **No external side effects.** The library may mutate the `ClusterState` passed into it (tracking placements within a batch requires updating state as pods are placed), but must not make API calls or mutate state outside what was passed in. Extension points that currently have external side effects (PreBind, Bind, PostBind) move to the orchestrator.
- **Preemption as recommendation.** PostFilter currently executes preemptions (deletes victim pods). In the library model, preemption must be a recommendation in the output (PodResult), not an executed action.
- **Logic parity.** Existing in-tree scheduler plugins (Filter, Score) must be usable in the library without modification. If we have to rewrite plugins, we haven't solved the costly maintenance problem.
- **Plugin extensibility.** Third-party plugins must be supported. Plugins that span evaluation and side effects (like DRA: PreFilter + Filter + Reserve + PreBind) need a clear pattern for splitting — evaluation in the library, side effects in the orchestrator.
- **Action ordering.** When the library has multiple options — bind to an existing node, provision new capacity, or preempt — it needs a policy to decide which action to take and in what order. This must be configurable, not hardcoded.
- **Fallback ordering.** Cost-based scoring can express "prefer cheaper options" but cannot enforce "never use on-demand if reserved capacity is available." Strict ordering requires a separate mechanism.

**Prior Art:**

- kube-scheduler's scheduling framework (KEP-624)
- Karpenter's greedy packing algorithm (used for both provisioning and disruption/consolidation simulation)
- KEP-4671 (Gang Scheduling) — introduces a "Workload Scheduling Cycle" wrapping multiple pod scheduling cycles
- KEP-5732 (Topology-aware Workload Scheduling) — PlacementGeneratorPlugin, PlacementStatePlugin, PlacementScorerPlugin
- KEP-4832 (Asynchronous Preemption) — already decouples preemption API calls from the scheduling cycle
- DRA scheduler plugin (KEP-4381) — spans PreFilter, Filter, Reserve, PreBind
- VolumeBinding plugin — spans Filter, Score, Reserve, PreBind
- CAS's node group simulation model
- The original WAS proposal's "Pod Scheduling Preferences API" with tiered waterfall logic

### Deferred to Orchestrator Design

This proposal defines the orchestrator's responsibilities. The following areas require detailed design:

- **Pod binding semantics.** How the orchestrator handles the gap between NodeClaim creation and node readiness — bind pods to NodeClaims immediately (reserving them in a "provisioning" state) or defer binding until the node is ready and re-evaluate.
- **Lifecycle orchestration.** When and how the orchestrator evaluates nodes for consolidation, drift, expiration, and maintenance — candidate selection, ordering, budgets, and rollback.
- **Provider integration.** How the orchestrator discovers offerings, composes overlays, tracks availability, routes NodeClaims to capacity providers, and handles fulfillment failures.
- **Queue / batching strategy.** How the orchestrator decides which pods to accumulate before calling `Schedule` and the tradeoff between scheduling latency and packing efficiency.
- **Capacity governance.** How the orchestrator enforces resource limits, maintains capacity buffers, and ensures provisioning decisions respect governance constraints. Buffer demand may need representation in `ClusterState` so the library can account for it during scheduling.

---

## What We Need to Decide

### sig-scheduling

Both the initial design discussions and this proposal require significant changes to kube-scheduler's internals. The initial design discussions expand the scheduler's responsibilities. This proposal refactors the scheduler for reuse. sig-scheduling's input is critical to evaluating the feasibility and approach for either path.

### sig-autoscaling

This proposal converges scheduling logic into a shared library. The Kubernetes API surface is the same whether CAS and Karpenter maintain separate orchestrators or share one — no APIs need to be introduced that would later need to be deprecated if the communities converge. sig-autoscaling needs to discuss what path forward makes sense.

---

## Appendix

### Static Clusters

In static clusters without an autoscaler, kube-scheduler continues to serve its dual role as both orchestrator and scheduling logic — scheduling pods onto existing nodes exactly as it does today. The `Schedule` interface with an empty potential capacity list degrades gracefully to pure scheduling against existing nodes, producing no provisioning decisions. kube-scheduler's queue is an example of orchestration logic — deciding how and when to make scheduling decisions.

### Scheduling vs. Orchestration

This proposal distinguishes between scheduling and orchestration.

- **Scheduling** is the core logic for determining where a pod could or should go.
- **Orchestration** is almost everything else — which pod to schedule next, which node to evaluate for removal, binding pods to nodes, preempting pods from nodes, etc.

Autoscalers today reimplement scheduling because orchestration decisions require it.

### Provisioning Flow

A concrete walkthrough of the provisioning use case:

1. A batch of pods becomes pending.
2. The autoscaler invokes the library with the pending pods, current cluster state, and the potential capacity model.
3. The library evaluates existing nodes and potential capacity through the same filter/score pipeline — including autoscaler-provided plugins for patching/defrag, maintenance, and capacity type preferences. For pods that don't fit on existing nodes, the library creates virtual nodes as superpositions of compatible instance types and packs pods onto them, narrowing the compatible set as constraints accumulate.
4. The library returns placements: which pods should bind to existing nodes, and what new nodes need to be created (expressed as finalized instance type requirements).
5. The autoscaler acts on the results — binds pods to existing nodes and creates capacity requests for new nodes.

### Removal Evaluation Flow

A concrete walkthrough of the consolidation/patching/defrag/expiration use case:

1. The autoscaler selects a candidate node for removal (underutilized, out-of-spec, expired, or scheduled for maintenance).
2. The autoscaler constructs a modified cluster snapshot with the candidate node removed.
3. The autoscaler invokes the library with the displaced pods, the modified snapshot, and the potential capacity model — the same call as the provisioning flow.
4. The result tells the autoscaler: can these pods be placed, and what does the new state look like? If the pods land on existing nodes, the candidate can be removed with no replacement. If they require new capacity, the autoscaler compares the cost of the replacement against the current node. If some pods can't be placed at all, removal is unsafe.
