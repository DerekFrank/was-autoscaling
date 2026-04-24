# Project Gluon: Proposal

*Gluon: Because we're **glue**ing Karpenter **on**to the scheduler.*

**Author:** Derek | **Date:** April 2026

## Executive Summary

This document requests dedicated engineering investment to co-drive [Workload-Aware Scheduler Autoscaling (WASA)](https://docs.google.com/document/d/1ergKWH28EpGyYVISZbqPqIBS1wN7VNhjNa_Sl1pAFI8/edit?usp=sharing), an upstream Kubernetes initiative that moves provisioning decisions into the kube-scheduler — making autoscaling a first-class scheduling action rather than a separate reactive control loop. At the WAS Summit at KubeCon EU 2026, the Cluster Autoscaler and Karpenter communities reached alignment on this direction. The design is being actively shaped now. AWS should co-drive this effort to ensure the resulting APIs serve EKS customers, to position Karpenter for the architectural shift, and to secure a seat at the table for how future Kubernetes features account for autoscaling.

## What Is WASA?

Today, the Kubernetes scheduler and node autoscaler operate as separate control loops with separate views of the world. The scheduler places pods onto existing nodes. When pods can't be placed, the autoscaler reacts — running its own scheduling simulation to decide what new capacity to provision. Neither component has full visibility into what the other is doing or could do.

WASA unifies these loops. Provisioning becomes a scheduling action: the scheduler sees both existing capacity and capacity that _could_ be provisioned, and makes globally informed decisions across both. When the scheduler decides a new node is the best option, it requests that node directly — no separate autoscaler simulation required. For customers, this means better placement, lower cost, and new scheduling features that work with autoscaling from day one.

## Why This Matters for AWS Customers

**Support for AI/ML training workloads.** Gang scheduling, topology-aware placement, and Dynamic Resource Allocation (DRA) are critical for distributed training. These are scheduling features that require deep autoscaling integration. Today, Karpenter must independently implement support for each one — reverse-engineering the scheduler's behavior to make correct provisioning decisions. WASA makes the community responsible for ensuring new scheduling features account for autoscaling, distributing this burden across SIG Scheduling, SIG Autoscaling, and the feature authors themselves.

**New scheduling capabilities for autoscaled capacity.** When the scheduler can treat provisioning as a scheduling action, it unlocks capabilities that are impossible in the current architecture. For example, customers today cannot express "autoscale for new pods, but if capacity isn't available, preempt lower-priority workloads instead." This requires the scheduler to evaluate provisioning and preemption as alternatives in a single decision — which is impossible when they're handled by separate control loops. Anthropic has expressed interest in exactly this capability in direct conversations with the Karpenter team. WASA makes these kinds of cross-cutting scheduling decisions possible for the first time.

**Community-shared maintenance burden.** The Karpenter team must proactively engage in the design of every new scheduling feature across multiple SIGs to catch cases where a feature won't be compatible with autoscaling. Sometimes those issues are missed. Sometimes the community hears the input and deprioritizes it (see [Appendix: KEP-5328](#kep-5328-a-concrete-example)). And when features do ship, the team must independently reimplement each one for provisioning decisions. WASA shifts autoscaling into the scheduler itself, making compatibility with autoscaling the community's shared responsibility rather than ours alone.

**A seat at the table for shaping features.** With provisioning as a first-class scheduling concern, AWS gains an active role in how new Kubernetes features are designed. Today, features are developed in SIG Node, SIG Scheduling, and other working groups without accounting for autoscaling implications. In the WASA model, autoscaling is part of the scheduler's core responsibility — which means features that affect provisioning will be designed with provisioning in mind from the start. AWS can shape those designs to serve EKS customers rather than adapting after the fact.

## Why Now

**Community alignment is live.** The CAS and Karpenter communities reached consensus on this direction at the WAS Summit at KubeCon EU 2026. See the [Node Autoscaling Alignment](https://docs.google.com/document/d/1syaoTkK1E6aWhxTjY-AZfDt5tViJr5_6_3w8W-y1pg8/edit?resourcekey=0-Me75lgJQLdjVKSgqou80xQ&tab=t.0#heading=h.y1fqrbqx9fco) document for details.

**The design is being shaped, not finalized.** APIs and architectural boundaries are in active development. Google has at least four senior or staff-level engineers working on WASA — sufficient resources to design and implement it independently. Early participants shape the architecture; late participants adapt to it.

**The alternative is adoption without influence.** If AWS is not at the table, the work proceeds without us. We adopt community-standard APIs we had no hand in designing, built around assumptions that may not match EKS's architecture or our customers' needs.

## What Karpenter Becomes

Karpenter pioneered many of the ideas that WASA is promoting into the core Kubernetes scheduler — validating that the approach works at scale for AWS customers.

In the WASA model, Karpenter evolves from "scheduler + cloud provider actuator + lifecycle manager" to "cloud provider actuator + lifecycle manager." It no longer reimplements scheduling logic for every new upstream feature. Instead, it focuses on lifecycle management — disruption handling, drift detection, and consolidation — while the AWS cloud provider layer continues to deliver deep integration with EC2 features like Spot, Reserved Instances, Placement Groups, EFA, Nested Virtualization, and Pod Isolation.

The internal scheduling logic is Karpenter's most expensive maintenance surface. Keeping it compliant with upstream Kubernetes requires continuous effort — every scheduler change must be tracked and mirrored. Moving it into the scheduler lets the Karpenter team concentrate on differentiated value.

The main area where Karpenter and CAS diverge is the consolidation algorithm — the logic that decides when and how to reorganize workloads onto fewer, better-suited nodes. WASA does not subsume consolidation. The scheduling simulation API is a necessary building block for it, but consolidation's value comes from the strategies built on top. Karpenter is actively investing in these strategies with the science team (candidate selection, local consolidation, disruption tradeoffs, and cost optimization) and retains full freedom to innovate independently.

The scheduling simulation is the primary barrier to growing new maintainers on the project. Karpenter has historically struggled to sustain more than two active reviewers. It can take years to develop the expertise, and normal attrition erases progress. Project Gluon is uniquely positioned to address this structural problem by removing the simulation from Karpenter's codebase, lessening the long-term maintainer burden without invalidating the short-term mitigations we are already performing.

## Cost of Inaction

**APIs designed without AWS input become APIs AWS must adopt.** This is not hypothetical. Google's initial API design carried a cardinality that could not scale to AWS's breadth of instance type offerings. Because we were already engaged, we altered the course of the design before it solidified. This effort will proceed with or without our continued participation — the engineers driving it have the proposals, the resources, and the motivation to design and implement these APIs independently. Sustained investment ensures we remain in the room as the architecture moves from design to implementation.

**The maintainer burden may compound.** Karpenter's scheduling simulation is already the primary bottleneck for growing new maintainers. If the upstream APIs are designed without our input, Karpenter may need to maintain both its internal simulation and an integration with APIs that don't fit its architecture — or adapt to APIs that make the integration itself burdensome. Inaction doesn't guarantee the status quo; it risks making the hardest part of the codebase harder.

**Karpenter gets disintermediated.** If AWS doesn't participate in shaping the upstream architecture, the project moves on without us — designing APIs, making integration assumptions, and building community momentum around approaches that don't account for Karpenter's design or its users. New features and community engagement increasingly center on the upstream path, and Karpenter shifts from leading the autoscaling conversation to adapting to decisions made without its input.

## Project Scope

This is upstream open-source work — deliverables and milestones will be shaped by community feedback and the KEP process. We're targeting alpha in 1.39 (~February 2027) and beta in 1.40 (~June 2027). These are ambitious targets intended to set a pace of engagement, not promise delivery dates. The scope of WASA is substantial — we estimate 5 to 12 KEPs spanning multiple SIGs and working groups, coordinated across alpha and beta milestones.

### The Ask

We are requesting at least two dedicated engineers through the initial phase of the project — defined as merging or closing the first KEPs. This phase encompasses disambiguation, authoring, and shepherding proposals through the community review process. Intensity varies: authoring requires dedicated focus, while shepherding — addressing feedback, attending SIG meetings, iterating on designs — requires sustained engagement at lower intensity.

- **Derek** leads project disambiguation initially — defining problem categories, establishing workstream boundaries, and driving alignment across SIGs — then transitions to authoring a proposal.
- **Jason** leads the Potential Capacity API proposal — the foundational API that defines how the scheduler discovers what capacity could be provisioned.

For context, Google has at least four senior or staff-level engineers working on WASA. Two engineers is the minimum to have a meaningful authoring presence, not parity.

We will flexibly reassess the level of dedicated investment as the initial KEPs reach resolution.

This is not a request for review bandwidth. AWS actively participated in the WAS Summit at KubeCon EU that produced the aligned architecture — and that participation is why the result is heavily inspired by Karpenter's design and reflects EKS customer needs. Authoring KEPs is the continuation of that engagement. The Karpenter team is in the best position to author these KEPs.

Authoring also ensures that concerns critical to our customers, such as migration paths for existing Karpenter users, are first-class design considerations, not afterthoughts.

#### Room to Grow

Two engineers is a floor, not a ceiling. Given the estimated 5 to 12 KEPs spanning multiple SIGs, additional headcount could accelerate the project once the disambiguation establishes clear workstream boundaries. As the project expands into implementation, the work becomes even more parallelizable — creating natural entry points to bring in additional engineers. This lets us grow Karpenter's upstream contributor bench, which is also a strategic goal in its own right regardless of where WASA lands.

### Risks

**Google shapes the design in ways that disadvantage AWS.** We have 2 engineers, they have 4+. Influence in the KEP process correlates with who's doing the work. We mitigate this by authoring proposals rather than just reviewing, and the aligned architecture is already Karpenter-inspired.

**The upstream effort stalls or fragments.** Multi-SIG coordination is hard and the KEP process is slow. If community priorities shift or consensus breaks down, the effort could stall. The 60-day checkpoint provides an early signal; if the upstream process is gridlocked, we reassess.

**Dual maintenance during transition.** When WASA reaches alpha, Karpenter still needs its internal scheduling simulation for clusters not yet on the latest Kubernetes version. This means maintaining both paths for a multi-year transition period. The old simulation can be frozen to bug fixes once the upstream path is viable.

**Karpenter review capacity during transition.** Derek and Jason will take on kube-scheduler reviews alongside Karpenter, increasing their review burden. A reviewer mentorship initiative is already in flight, and as implementation fans out, the project provides natural opportunities to grow additional reviewers.

### Next Steps

**Establish a regular upstream working meeting (Derek).** A weekly one-hour meeting with WASA contributors across SIGs to coordinate workstreams, resolve cross-cutting design questions, and keep the project unblocked.

**Propose a high-level disambiguation of the project both internally and upstream (Derek).** This is the first concrete deliverable. The disambiguation will establish workstream boundaries, identify which KEPs and problem spaces are most important for AWS to lead on, and help keep the upstream project moving. It provides the basis for defining success criteria and exit ramps.

**Collaborate with EKS Product to assess roadmap impact (Sergiy).** Jason is wrapping up DRA support and Derek is handing off IP-Aware Scheduling to the networking team. WASA is a natural next priority for both engineers without cutting a project short, but dedicating them means reprioritizing other planned work. Once the disambiguation establishes the shape of the work, collaborate with EKS Product to assess the impact of this strategic pivot on the Karpenter roadmap.

**60-day checkpoint (Derek, Jason).** Assess whether the investment is tracking: upstream working meeting cadence established, at least one KEP submitted for review, and a clear read on whether community momentum is real or stalling.

---

## Appendix: Technical Context

### The Split-Brain Problem

Today, the scheduler and autoscaler maintain independent views of the cluster. When a pod is pending, the scheduler evaluates existing nodes and finds no fit. The autoscaler then runs its own scheduling simulation — reimplementing the scheduler's filtering, scoring, and constraint logic — to determine what new node to provision.

These two simulations can disagree. The scheduler may reject a node that the autoscaler considers valid, or vice versa. The autoscaler may provision a node type that the scheduler then refuses to use. The scheduler may pack pods onto existing nodes inefficiently because it cannot see that a better-suited node could be provisioned alongside them. These disagreements cause wasted capacity, unnecessary cost, and difficult-to-diagnose scheduling failures.

### The Feature Parity Overhead

Every new Kubernetes scheduling feature must be independently reimplemented by every autoscaler. Consider DRA (Dynamic Resource Allocation): the scheduler must understand device claims, allocation policies, and device topology to place pods correctly. For the autoscaler to make correct provisioning decisions, it must _also_ understand all of these semantics — because it needs to predict whether the scheduler will actually use a node it provisions.

This creates a permanent feature lag. When DRA semantics change upstream, Karpenter must update its internal model to match. When topology spread constraints gain new capabilities, Karpenter must reimplement them. When any scheduling plugin changes behavior, Karpenter must detect and adapt. The Karpenter team is effectively maintaining a parallel scheduler that must stay in lockstep with upstream — a burden that grows with every Kubernetes release.

### What "Provisioning as a Scheduling Action" Means

In the WASA model, the scheduler evaluates three classes of options for every pending pod in a single decision cycle:

1. **Bind to an existing node** — the pod fits on a node that already exists in the cluster.
2. **Bind to an in-flight node** — the pod fits on a node that has been requested but hasn't joined the cluster yet.
3. **Request a new node** — no existing or in-flight node is suitable, so the scheduler creates a capacity request specifying the requirements the new node must satisfy.

The scheduler uses the same filtering, scoring, and constraint logic for all three options. There is no separate simulation, no second control loop, and no risk of disagreement between scheduler and autoscaler. The cloud provider (Karpenter, in AWS's case) receives the capacity request and fulfills it — selecting a concrete instance type, launching the instance, and reporting back — without needing to understand or replicate scheduling logic.

### KEP-5328: A Concrete Example

[KEP-5328](https://github.com/kubernetes/enhancements/pull/5797) (Node Declared Features) introduced a mechanism for nodes to explicitly declare their capabilities, enabling version-skew-safe scheduling. The autoscaling team raised during the review that this breaks scale-from-zero: autoscalers can't populate declared features for nodes that don't exist yet. Without that information, the scheduler won't consider those nodes viable, and pending pods that depend on declared features will never trigger scale-up.

The community acknowledged the concern and moved the KEP to beta anyway. The autoscaler teams were left to vendor a shared library, integrate it per cloud provider, and track which features are added or removed every Kubernetes release — indefinitely. The SIG Scheduling reviewers explicitly noted that scale-from-zero "will remain autoscaling SIGs concern even under unified scheduling" and that the KEP was "not a scheduling concern."

The operational burden compounds over time. CAS maintainer Kuba Tużnik [noted](https://github.com/kubernetes/enhancements/pull/5797#issuecomment-3886560750) that every cloud provider must update their scale-from-zero integration for each Kubernetes release to track new features entering the declared features framework. The set of affected pods grows with every release, existing documentation goes stale, and the failure mode is silent: clusters that work correctly at scale break when scaled to zero. There are already three features using the framework in 1.36, one of which affects scheduling. The autoscaling integration gap is not hypothetical — it exists today.

This is the dynamic WASA addresses: features designed in one SIG, with autoscaling implications treated as someone else's problem.
