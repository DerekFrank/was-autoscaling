# N:M Scheduler × Autoscaler × Capacity Provider Analysis (Scratch)

## Key Insight

Both architectures require a standard interface between scheduling logic and autoscaling orchestration. Both support custom schedulers and custom autoscalers. The difference is that the library eliminates the split-brain by going further in their integration — collocating scheduling logic and orchestration in the same process.

When multiple autoscalers exist, *something* must be partitioned to avoid races. The architectures differ in *what* naturally gets partitioned:

- **Scheduler as Autoscaler:** naturally partitions *offerings* (each autoscaler fulfills NodeClaims matching its offering space)
- **Scheduler as Library:** naturally partitions *pods* (each autoscaler owns a set of pods and invokes the library for them)

Both forms of partitioning reduce global optimization, and both race with overprovisioning if partitions overlap. Neither is strictly superior — they represent different dimensions of control.

## Distinction: Multiple Autoscalers vs. Multiple Capacity Providers

- **Multiple autoscalers** = different orchestration/lifecycle logic (CAS + Karpenter). Separate decision-makers with different strategies.
- **Multiple capacity providers** = different infrastructure backends (AWS + Azure). One decision-maker with multiple fulfillment options.

Both architectures can support multiple capacity providers. The single/multi provider dimension doesn't materially change the analysis — it doesn't introduce new coordination problems in either architecture. The meaningful coordination challenges arise in the multi-autoscaler dimension.

## Full 3D Matrix

| # | Schedulers | Autoscalers | Providers | Scheduler as Autoscaler | Scheduler as Library |
|---|-----------|-------------|-----------|------------------------|---------------------|
| 1 | Single | Single | Single | ✓ Trivial case | ✓ Trivial case |
| 2 | Single | Single | Multiple | ✓ Scheduler sees all offerings, one autoscaler routes to correct provider | ✓ Library gets all offerings in one call, autoscaler routes result |
| 3 | Single | Multiple | Single | NodeClaim race — must partition offerings | Pod race — must partition pods |
| 4 | Single | Multiple | Multiple | NodeClaim race — must partition offerings | Pod race — must partition pods |
| 5 | Multiple | Single | Single | ✓ Each scheduler creates NodeClaims, one autoscaler fulfills — no race | ✓ Autoscaler loads N library implementations, invokes correct one per pod |
| 6 | Multiple | Single | Multiple | ✓ Each scheduler creates NodeClaims, one autoscaler routes internally | ✓ Autoscaler loads N libraries, passes unified offerings |
| 7 | Multiple | Multiple | Single | NodeClaim race + lifecycle coordination across schedulers — must partition offerings | Pod race — must partition pods. Each autoscaler loads N libraries |
| 8 | Multiple | Multiple | Multiple | NodeClaim race + lifecycle coordination across schedulers — must partition offerings | Pod race — must partition pods. Each autoscaler loads N libraries with its own providers |

## Observations

1. **"Scheduler as Autoscaler" breaks when multiple autoscalers exist** (rows 3, 4, 7, 8). The NodeClaim fulfillment race is the fundamental issue — the scheduler can't control which autoscaler fulfills a NodeClaim when requirements span multiple autoscalers' offering spaces.

2. **"Scheduler as Autoscaler" works cleanly with a single autoscaler** (rows 1, 2, 5, 6). One fulfiller → no race.

3. **"Scheduler as Library" collapses the multi-scheduler and multi-provider dimensions cleanly.** Multiple schedulers are loaded in-process. Multiple providers are aggregated internally by the autoscaler. These dimensions don't introduce coordination problems.

4. **Both architectures require partitioning in multi-autoscaler cases.** The failure mode is symmetric — overlap in either partition dimension leads to duplicate provisioning / wasted capacity. The difference is *what* you partition, not whether partitioning is needed.

5. **Neither architecture provides global optimization in multi-autoscaler cases.** "Scheduler as Autoscaler" *appears* to offer it (one scheduler sees all offerings) but can't deliver because the NodeClaim race means executed results may diverge from the computed plan. The library model doesn't pretend — global optimization requires a single decision-maker (one autoscaler, potentially with multiple providers).

## On Offering Partitioning vs. Pod Partitioning

The most common real-world case is **single scheduler, multiple autoscalers** (e.g., Karpenter + CAS, or multiple Karpenter instances with different NodePools). Today, operators partition offerings between autoscalers — each autoscaler manages a disjoint set of instance types / node groups. This is well-understood and widely deployed.

However, it's worth asking: is offering partitioning what operators *want*, or is it the only lever currently available? Operators often think in terms of workload classes ("my batch jobs" vs. "my web services"), not instance type sets. Pod partitioning may be more aligned with operator *intent* — but offering partitioning is more aligned with the *existing operational model*.

Neither is clearly superior:

| Dimension | Offering Partitioning | Pod Partitioning |
|-----------|----------------------|-----------------|
| Familiarity | Mirrors today's model (NodePools, node groups) | Requires new operational patterns |
| What it restricts | Capacity choices per autoscaler | Which autoscaler handles which workloads |
| Race on overlap | Two autoscalers fulfill same NodeClaim | Two autoscalers schedule same pod |
| Consequence of overlap | Duplicate nodes | Duplicate nodes |
| Operator mental model | "This autoscaler owns these instance types" | "This autoscaler owns these workloads" |

## Where the Library Model Has Clear Advantages

The library model's strengths concentrate in dimensions *other* than multi-autoscaler:

1. **Multiple schedulers** — collapses entirely. Each autoscaler loads N library implementations in-process. No external coordination needed. In "Scheduler as Autoscaler," lifecycle operations (consolidation, drift) require the autoscaler to coordinate with *each* scheduler whose pods are on the affected node.

2. **Multiple providers** — handled cleanly by both architectures. Not a differentiator.

3. **The single-autoscaler case** — in-process integration eliminates the split-brain entirely. Decision and execution in the same process. This is the core argument of the proposal and applies regardless of the scheduler/provider dimensions.

## Multi-Autoscaler Use Cases

All legitimate multi-autoscaler use cases are about **isolation/independence** and already imply partitioning:

- Different lifecycle strategies (aggressive vs. conservative) → workloads already segmented
- Blast radius isolation → requires disjoint domains
- Organizational boundaries → partitioned by namespace/team
- Upgrade independence → partitioned by criticality
- Maturity levels → partitioned by environment

If you wanted global optimization, you'd deploy one autoscaler. The choice to deploy multiple is a choice to accept local optimization in exchange for independence. This is true in both architectures.

## Open Questions

- Does the proposal need to explicitly state the "custom schedulers must implement the library interface" constraint?
- Where does this analysis land in the doc? Properties section? New comparison section?
- Is the "mirrors today's model" property of "Scheduler as Autoscaler" an advantage (easier adoption) or a weakness (preserves the fundamental coordination problem)?
- For the multi-autoscaler case: is "both have tradeoffs, neither is strictly better" the right framing, or is there a tiebreaker we haven't identified?
