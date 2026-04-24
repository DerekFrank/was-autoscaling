# Scheduler Algorithm Design

## Core Approach: Greedy Packing

Adopt Karpenter's greedy scheduler model:

1. Start with "virtual NodeClaim" - all compatible InstanceTypes
2. Place first pod → requirements narrow (e.g., needs GPU → only GPU types remain)
3. Place second pod → requirements narrow further (e.g., zone affinity)
4. Continue until no more pods fit or requirements unsatisfiable
5. Finalize NodeClaim with accumulated requirements
6. Repeat for remaining pods

**Key insight:** NodeClaim doesn't pick specific InstanceType until cloud provider actuates. It accumulates constraints.

## Batching

Assume kube-scheduler will have/extend a batching mechanism (potentially from gang scheduling work). Provisioning runs as a batch operation, not per-pod.

> **TODO:** This section needs to be more specific about the batch *formation* mechanism — i.e., how the scheduler collects pods into a batch before running the greedy packing algorithm. The current text describes what happens within a batch (greedy packing, constraint narrowing) but is silent on how the batch is triggered. The WAS alignment doc (`google-proposals/was-node-autoscaling-alignment.md`) suggests a temporal delay in the provisioning cycle to accumulate pods before committing to NodeClaims. Key questions to address:
> - What triggers a provisioning batch? Time-based delay? Pod count threshold? Queue drain?
> - How long does the scheduler wait before cutting off accumulation?
> - How does this interact with the normal scheduling queue and existing gang scheduling work?
> - What prevents the delay from regressing scheduling latency for pods that could bind to existing nodes?

## NodePool Awareness

**Decision:** Scheduler does NOT have NodePool awareness.

InstanceTypes are pre-expanded per NodePool by cloud provider. Scheduler sees them as flat list with baked-in:
- Node labels/taints (in requirements)
- Capacity adjustments (already applied)
- Cost (in offerings)

**Information lost without NodePool awareness:**
- Limits enforcement → cloud provider rejects NodeClaims over budget
- Pool-level disruption policy → not scheduler's concern
- Pool priority/weight → encoded in cost
- Pool-scoped topology → baked into InstanceType requirements
- Cost attribution → use labels on InstanceType

## Topology Spread Constraints with In-Flight NodeClaims

**Decision:** Hybrid approach

- **Known topology** (single value in NodeClaim requirements): count normally toward that value
- **Unknown topology** (multiple values): count as worst-case toward ALL possible values

Example:
- NodeClaim A: `zone In [us-west-2a]` → counts as 1 in us-west-2a
- NodeClaim B: `zone In [us-west-2a, us-west-2b]` → counts as 1 in BOTH zones

This encourages greedy packer to narrow topology early when TSC is involved.

## Scoring / Prioritization

**Decision:** Use generic `cost` field, lower is better.

```yaml
status:
  offerings:
    - requirements: [...]
      cost: "0.096"    # Generic optimization value
      available: true
```

- Cloud provider sets base cost (could be price, or any preference signal)
- CapacityOverlay can adjust cost to encode user preferences
- Scheduler picks lowest cost among compatible options
- Tie-break on resource efficiency (smallest instance that fits)

**Why `cost`:**
- Intuitive direction (lower = better)
- Doesn't conflict with scheduler's existing `score` concept
- Works for monetary and abstract tradeoffs

## Open Questions

1. **Pod ordering in batch:** Use scheduler queue order? Sort by priority then resource size?
2. **Retry backoff:** After NodeClaim failure, how long before trying again?
3. **NodeClaim timeout:** When to give up on stuck NodeClaim?
4. **Preemption of Reserved pods:** Can higher-priority pod take a Reserved pod's spot?
5. **nodeName promotion:** Who copies NodeClaim.status.nodeName → pod.spec.nodeName?
6. **Multi-scheduler:** What if another scheduler tries to use same NodeClaim?
