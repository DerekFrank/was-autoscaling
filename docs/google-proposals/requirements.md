# Scheduler-Driven Autoscaling PRD

Authors/collaborators: [Kuba Tużnik](mailto:jtuznik@google.com), \[add yourself\]  
Last edited: Apr 20, 2026  
Status: WIP

# Background

*Scheduler-Driven Autoscaling* is an effort aiming to solve a number of long-standing problems in the Node autoscaling space by redesigning how Node autoscalers (Cluster Autoscaler, Karpenter) interact with kube-scheduler. Scheduler-Driven Autoscaling is a part of a broader *Workload Aware Scheduling* project.

Context docs:

* [\[Public\] Workload Aware-Scheduler Cluster Autoscaling](https://docs.google.com/document/d/1ergKWH28EpGyYVISZbqPqIBS1wN7VNhjNa_Sl1pAFI8/edit?tab=t.77ykkkwdq8vp#heading=h.z57mquggx1b6): The first public proposal for Scheduler-Driven Autoscaling. This doc goes over high-level aspects (motivations, goals, non-goals etc.), and proposes a set of concrete APIs that could be used to achieve the goals.  
* [\[Public\] WAS Node Autoscaling Alignment](https://docs.google.com/document/d/1syaoTkK1E6aWhxTjY-AZfDt5tViJr5_6_3w8W-y1pg8/edit?resourcekey=0-Me75lgJQLdjVKSgqou80xQ&tab=t.0#heading=h.y1fqrbqx9fco) The doc listed above was extensively discussed by key CA and Karpenter stakeholders during Kubecon Europe 2026\. This doc captures the outcomes of these discussions.

The proposal doc linked above focuses on a concrete solution to the problems it identifies \- introducing a set of CRD-based APIs that Node autoscalers and kube-scheduler would use to communicate with each other. When discussing the proposal from the perspective of Node autoscalers, we started evaluating other solutions that could solve the same problems. There are multiple axes of trade-offs between the different solutions, and discussing them based on the initial proposal doc is challenging. We came to the conclusion that we need to take a step back \- focus on the problems we want to solve, before we get to discussing concrete solutions.

This document:

* Identifies and prioritizes the problems resulting from the current way that Node autoscalers and kube-scheduler interact with each other.  
* Identifies and prioritizes the requirements we have for any solution to the identified problems.

# Problems

Each following subsection describes a problem resulting from Node autoscalers’ interactions with kube-scheduler. The problems might impact Cluster Autoscaler and Karpenter to different degrees, so the priority is presented from different perspectives.

## Split brain: Inconsistent Pod placement decisions

Cluster Autoscaler priority:  
Karpenter priority:  
WAS priority:  
**Priority:** 

kube-scheduler uses its scoring logic to decide which Node is used when scheduling a single pending Pod. Node autoscalers use their own separate logic for the same decisions (e.g. when simulating rescheduling a Pod from a consolidated Node), plus they use additional selection logic which decides which groups of Nodes are provisioned for a group of pending Pods. Since the selection logic is different, Node autoscalers simulations can diverge from actual kube-scheduler behavior. This can result in corner cases where Node autoscalers make suboptimal or outright wrong scaling decisions.

Concrete problem examples:

*   
* Node autoscaler simulates consolidating a Node, tries rescheduling all of its Pods onto other Nodes in the cluster, finds a Node for each Pod, triggers consolidating the Node. Scheduler picks different Nodes for the recreated Pods than the autoscaler simulations, which results in one of the Pods not being schedulable on any Node. Node autoscaler notices that and provisions a new Node for the Pod. Then eventually it consolidates some Node back based on the same flawed simulations. In the end the Node autoscaler enters a cycle of provisioning and consolidating a Node completely unnecessarily.  
* (Not sure if this belongs here) Vertical pod autoscaling (especially in-place) causes some cycling/churn between scheduling/autoscaling. For example, someone has a JVM pod that starts out at a size then in-place scales down vertically after some short period. Then the autoscaler sees the drop in utilization and disrupts the node forcing the same start-up and scale down cycle again. To fix this you might need some awareness of the difference between the current pod request and what it would be if you disrupted/restarted the pod. This also applies to daemonsets – CAS subtracts the sum of the resources requested by the daemonset when calculating capacity for a new node, but the VPA recommendation can be different than the resources defined in the DS spec. That can lead to provisioning a node that is too small to handle the unschedulable pods.

## Costly kube-scheduler integration maintenance

Cluster Autoscaler priority:  
Karpenter priority:  
WAS priority:  
**Priority:** 

Node autoscalers have to either vendor, or repeat parts of kube-scheduler logic for the cluster simulations they do. Both of these approaches are very costly to maintain. Even vendoring the logic directly, major features like DRA require significant changes to the simulation logic around the vendored code. If a Node autoscaler doesn’t integrate with some feature well, the feature is a pain to use in autoscaled clusters (or can’t be used at all).

Concrete problem examples:

* Adapting Cluster Autoscaler to work with the DRA scheduler plugin required a \~SWE-year. Plus a high number of much smaller follow-ups.  
* Cluster Autoscaler has had very poor integration with the scheduler plugin checking CSI volume limits until very recently. This resulted in corner cases in provisioning (undercounting the number of new Nodes required), and even in kube-scheduler without autoscaling (admitting too many Pods on a Node before the CSINode is published).  
* Adapting Node autoscalers to the workload-based scheduling introduced by WAS seems like an even more complex change than DRA.  
* It takes time for new scheduling features to get to users with autoscaled clusters.  
* Confusing inconsistencies: some features are supported by autoscalers, some aren’t, users are confused.  
* Manageable performance issues in kube-scheduler plugins can explode into major issues when run much more frequently in autoscaling simulations.

## No compatibility with custom schedulers

Cluster Autoscaler priority:  
Karpenter priority:  
WAS priority:  
**Priority:** 

Kubernetes has supported running custom/multiple schedulers in a cluster for a long time. The scheduling simulation logic in Node autoscalers is only simulating the default kube-scheduler implementation (and only a single config in a given cluster).

* Custom extensions to kube-scheduler also don’t work

## Interesting features are blocked by the current interaction setup

Cluster Autoscaler priority:  
Karpenter priority:  
WAS priority:  
**Priority:** 

The current interaction setup between Node autoscalers and kube-scheduler prevents us from implementing a number of features on either side that could potentially benefit our users a lot.

Concrete feature examples:

* Pods could prefer provisioning a new, cheaper Node instead of being scheduled on an existing, more expensive one.  
* Pods could prefer provisioning a new Node instead of preempting a running Pod from an existing Node.  
* Scheduler could avoid scheduling a Pod on a Node about to undergo some maintenance event  
* DRA prioritized oneOf could be implemented in a straightforward way as scheduler scoring  
* Job users could have their jobs land on nodes that have a lifetime compatible with their job lifetime

## Tight coupling between scheduling and autoscaling is not apparent from the scheduling side

Cluster Autoscaler priority:  
Karpenter priority:  
WAS priority:  
**Priority:** 

People with Node autoscaling context are aware of how tightly coupled Node autoscalers are with kube-scheduler. The same isn’t necessarily true for the people on the scheduling side, because the interactions between the two are not explicit.

Concrete problem examples:	

* We’ve seen a number of KEPs that propose new scheduling features without taking Node autoscaling into account (or taking it into account on a “best effort” basis).  
* Example: [Node Declared Features](https://github.com/kubernetes/enhancements/pull/5797#issuecomment-3875026503)

## TODO: anything else?

Cluster Autoscaler priority:  
Karpenter priority:  
WAS priority:  
**Priority:** 

# Solution requirements

Each following subsection describes a requirement for any solution we come up with to the problems identified above. Cluster Autoscaler and Karpenter might have different priorities for the requirements, so the priority is presented from different perspectives.

## Node autoscalers stop simulating scheduling completely

Cluster Autoscaler priority:  
Karpenter priority:  
WAS priority:  
**Priority:** 

If we want to solve the problem of costly kube-scheduler integration maintenance, we need to ensure that the solution ultimately completely removes the need for Node autoscalers to simulate scheduler behavior. If any part of Node autoscaler logic needs to simulate scheduling, we still need to pay the maintenance cost so the problem isn’t solved.

## Existing Node autoscaling behavior is preserved

Cluster Autoscaler priority:  
Karpenter priority:  
WAS priority:  
**Priority:** 

There are two major kubernetes owned autoscalers, each with their own user facing consolidation and lifecycle management invariants. The proposed solution should allow both to retain their functionality. 

## Existing Node autoscaling performance is preserved

Cluster Autoscaler priority:  
Karpenter priority:  
WAS priority:  
**Priority:** 

TODO: Describe requirement

## Existing Pod scheduling performance is preserved

Cluster Autoscaler priority:  
Karpenter priority:  
WAS priority:  
**Priority:** 

TODO: Describe requirement

## TODO: anything else?

Cluster Autoscaler priority:  
Karpenter priority:  
WAS priority:  
**Priority:**   
