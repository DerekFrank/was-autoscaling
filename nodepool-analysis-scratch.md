# NodePool Analysis - Design Scratch Notes

**Date:** March 25, 2026
**Context:** Analyzing NodePool as a primitive for the upstream Kubernetes capacity API

---

## What Functions Does NodePool Serve Today?

| Function | Description | Example |
|----------|-------------|---------|
| Instance type filtering | Constrains which instance types are eligible | `requirements: [{key: node.kubernetes.io/instance-type, operator: In, values: [m5.large]}]` |
| Node configuration | Labels/taints applied to provisioned nodes | `labels: {team: frontend}`, `taints: [{key: team, effect: NoSchedule}]` |
| Resource limits | Caps total resources provisioned through this pool | `limits: {cpu: 1000, memory: 1000Gi}` |
| Disruption policy | Controls consolidation, expiration, budgets | `disruption: {consolidationPolicy: WhenEmptyOrUnderutilized}` |
| Cloud provider config | Reference to provider-specific settings | `nodeClassRef: {kind: EC2NodeClass, name: default}` |
| Weight/priority | Preference ordering for fallback | `weight: 100` |

---

## Categorizing These Functions

| Function | Category | Who Needs It |
|----------|----------|--------------|
| Instance type filtering | Scheduling constraint | Scheduler |
| Node labels/taints | Scheduling constraint | Scheduler |
| Resource limits | Capacity governance | Admission/Quota controller |
| Disruption policy | Lifecycle management | Disruption controller |
| Cloud provider config | Actuation detail | Cloud provider |
| Weight/priority | Scheduling preference | Scheduler |

---

## Does the Scheduler Need to Know About NodePool?

| Function | Scheduler Needs It? | Why/Why Not |
|----------|---------------------|-------------|
| Instance type filtering | **No** | Implicit in published offerings—cloud provider only publishes valid combinations |
| Node labels/taints | **No** | Baked into offering requirements/taints |
| Resource limits | **No** | Separate controller updates offering availability when limits hit |
| Disruption policy | **No** | Post-scheduling lifecycle controller |
| Cloud provider config | **No** | Cloud provider's concern when publishing offerings |
| Weight/priority (fallback) | **No** | Encoded in cost field via ordered overlays |

**Conclusion:** The scheduler does not need to know about NodePool.

---

## Key Insight: NodePool Bundles Six Concerns

NodePool combines orthogonal concerns into one object:

1. **Scheduling constraints** (what pods can land here)
2. **Scheduling preferences** (which pool to prefer)
3. **Capacity governance** (how much total)
4. **Lifecycle management** (when to disrupt)
5. **Actuation configuration** (how to create)
6. **Ownership/grouping** (which nodes belong together)

Some are tightly coupled (labels/taints affect scheduling), others are orthogonal (disruption policy doesn't affect scheduling decisions).

---

## Fallback Ordering: The Core Remaining Function

After stripping away everything that's implicit, convenience, or belongs to a different controller, NodePool's essential function is:

**Ordered groups of offerings with fallback semantics.**

"Try these offerings first. If none work, try the next set."

### Can This Be Achieved Without a Grouping Primitive?

Yes, via cost:

| Approach | Description | Works? |
|----------|-------------|--------|
| Adjusted cost | User sets cost to encode preference | Soft preference only—scheduler can trade off |
| Extreme cost differentials | Set reserved to 0.001, on-demand to 100 | Arms race—user can always outbid |
| Ordered overlays | Cloud provider overlays applied last | **Yes**—cloud provider always wins |

### Ordered Overlays Solution

Cloud provider overlays are applied after user overlays, giving cloud providers the final word on cost:

```
Base cost → User overlays → Cloud provider overlays → Final cost
```

Example:
```yaml
# Cloud provider overlay (applied last)
- match: {capacity-type: cud}
  costAdjustment: "-999999"  # Guarantees CUD is preferred

# User overlay (applied first)
- match: {zone: us-west-2a}
  costAdjustment: "-0.01"    # Soft preference for zone
```

Result: CUD in us-west-2a beats CUD in us-west-2b, but all CUD beats non-CUD.

### What Does the User Lose?

The only thing lost: **User cannot override cloud provider's hard preferences.**

If cloud provider says "CUD first, always," user cannot say "actually, I want spot first."

For stakeholders with hard fallback requirements (contractual, compliance), this is a feature, not a bug.

---

## Resource Limits Without Scheduler Awareness

Limits can be enforced by updating offering availability:

1. Limits controller watches resource usage per tenant/config
2. When limit is reached, controller marks affected offerings `available: false`
3. Scheduler sees unavailable offerings, doesn't know why
4. When usage drops, controller marks offerings available again

This keeps the scheduler simple—it just filters by availability.

(Detailed limits design in separate document.)

---

## Provenance Without Scheduler Awareness

When something goes wrong, operators need to trace back to the source config.

This doesn't require scheduler awareness. The NodeClaim carries a reference:

```yaml
status:
  provenance:
    configRef:
      kind: EC2NodeClass
      name: frontend-config
```

Cloud provider sets this when creating the NodeClaim. Scheduler doesn't need to know.

---

## Summary

| Concern | Where It Lives | Scheduler Aware? |
|---------|----------------|------------------|
| Instance type filtering | Implicit in offerings | No |
| Node labels/taints | Offering requirements/taints | No |
| Resource limits | Limits controller → availability | No |
| Disruption policy | Disruption controller | No |
| Cloud provider config | Cloud provider internal | No |
| Fallback ordering | Cost field via ordered overlays | No |
| Provenance | NodeClaim metadata | No |

**NodePool becomes a cloud provider concern that produces offerings. The scheduler is NodePool-agnostic.**
