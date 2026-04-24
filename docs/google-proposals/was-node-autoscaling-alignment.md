# \[Public\] WAS Node Autoscaling Alignment

**Authors:** [Paweł Kępka](mailto:pkepka@google.com)  
**Last Update:** Apr 2, 2026

This document summarizes the key points from discussions around the integration of node autoscaling logic into the Kubernetes Scheduler, held during KubeCon EU 2026 between the Cluster Autoscaler and Karpenter teams. It also incorporates and extends the concepts initially proposed in the [Workload Aware-Scheduler Cluster Autoscaling](https://docs.google.com/document/d/1ergKWH28EpGyYVISZbqPqIBS1wN7VNhjNa_Sl1pAFI8/edit?usp=sharing) proposal, and a [presentation](https://docs.google.com/presentation/d/1yWjnrK0bbWbh89CkDDVEZoAyux1quE-zLJCgX9jOnSY/edit?usp=sharing&resourcekey=0-Z_3jNa04Y9B_gLivSfC7rQ) on the topic.

# Summary

There is a consensus within both the Cluster Autoscaler and Karpenter communities on the value of integrating node autoscaling logic within the core Kubernetes Scheduler. The vision is to make the scheduler the primary component responsible for making provisioning decisions. This shift promises to address common challenges, reduce duplicated efforts, and allow teams to focus on core issues like cloud provider integrations and advanced provisioning/consolidation logic, while maintaining compatibility with new Kubernetes scheduling features.

# Common Problems

Both Cluster Autoscaler and Karpenter projects face common problems which could be addressed by the proposed solution:

* **Feature Parity and Engineering Overhead**: Maintaining parity with evolving Kubernetes scheduling features, such as DRA and WAS, necessitates substantial engineering investment from autoscalers developers. This results in significant duplication of effort between the Cluster Autoscaler and Karpenter projects and creates a lag in features availability across the ecosystem.

* **Split-Brain Architecture, Logic Misalignment, and Suboptimal Scheduling**: The decoupling of node autoscaling and scheduling control loops restricts optimization across existing and new cluster capacity, risking logic and priority misalignment. This "split-brain" architecture prevents the scheduler from maintaining a holistic view of potential capacity, which hinders efficient bin-packing and prevents workloads from effectively leveraging mixed-capacity scheduling across current and newly provisioned nodes. Consequently, the inability to account for potential new capacity restricts the Scheduler's ability to make globally optimal placement decisions, often forcing the use of existing nodes even when they are a poor fit for specific workload requirements regarding cost, performance, and instance types.

# Main Challenges

The following primary challenges were identified during the discussions, alongside potential mitigation strategies that require further exploration and verification:

* **Cardinality of Provisioning Options**: The extensive number of instance types offered by major cloud providers presents a significant challenge for scheduling simulation logic, potentially leading to performance degradation that impacts both scheduling latency and throughput. This complexity is further exacerbated by user-defined overlays that modify provider-offered instance properties.  The Karpenter project has developed specific strategies to mitigate these bottlenecks, which should serve as a foundational reference for the Kubernetes Scheduler implementation.

* **Batching of Pods**: To optimize instance type selection during provisioning, the Kubernetes Scheduler must move beyond single-pod or individual workload bin-packing, as this approach may result in an inefficient creation of smaller nodes. Instead, the Scheduler should attempt to batch multiple pods before initiating node provisioning. While Cluster Autoscaler and Karpenter have implemented their specific solutions for this, they may not be directly applicable to the Scheduler. A better approach could involve introducing a slight delay in the provisioning cycle, allowing the Scheduler to accumulate additional pods on potential nodes before finalizing the configuration and dispatching the actual provisioning requests.

* **Scheduling Simulation API**: To eliminate direct dependencies on scheduling implementation and prevent redundant logic re-implementation, Cluster Autoscaler and Karpenter require a mechanism for simulating different autoscaling scenarios. This includes modeling node removal and concurrent replacement while evaluating the impact on pod and workload rescheduling, which is essential for advanced provisioning and consolidation capabilities. A dedicated Scheduling Simulation API within the Kubernetes Scheduler could address this need. Ideally, this API should operate within a decoupled control loop or independent scheduler instance to preserve primary scheduling latency and throughput, while ensuring logic parity by leveraging the core scheduling codebase.

* **Modeling Hypothetical Pods and Buffers for Consolidation:** The scheduler's simulation capabilities need enhancement to accurately model capacity that isn't yet active and to incorporate buffers essential for safe cluster consolidation. This limitation hinders optimal autoscaling and consolidation decisions. Key aspects include:

* **Buffer Awareness for Consolidation:** To allow effective consolidation of nodes, the scheduler logic must recognize and respect buffers. This ensures that node removal during consolidation doesn't compromise cluster stability or workload availability.

* **Representing Hypothetical Pods:** A more generic mechanism is required within the Scheduler to represent "fake" or "hypothetical" pods. This would enable the scheduler to make more intelligent provisioning and consolidation decisions by considering workloads that are pending, expected, or being moved. The concept of "Reservations" could be a potential solution for modeling these not-yet-running pods.

# Next steps

* Establish a recurring open synchronization forum under SIG Autoscaling with dedicated Slack room and meeting slot for both communities, dedicated to the proposed integration, to facilitate ongoing discussions and drive the project forward.

* Develop a revised proposal for the Potential Capacity, Provisioning and Scheduling Simulation APIs to align with the integrated scheduling architecture.

* Develop a proof-of-concept implementation within the Kubernetes Scheduler to address the identified primary challenges. This initiative should aim to facilitate a deeper understanding of existing architectural gaps, identify potential performance bottlenecks, and clarify the overall scope of required modifications.

* Develop a plan for incorporating safety buffers and hypothetical pod representations into the core scheduler logic to support intelligent consolidation and provisioning decisions.