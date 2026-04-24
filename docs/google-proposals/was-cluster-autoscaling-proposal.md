Workload Auto-Scheduler Cluster Autoscaling
[Public] Workload Aware-Scheduler Cluster Autoscaling
Authors: Paweł Kępka
Status: Draft
Last Update: Feb 12, 2026
This document provides a high-level overview for the proposal of Workload Aware-Scheduler Cluster Autoscaling architecture. It specifically details the role of three core integration APIs:
1. Potential Capacity API (Discovery)
2. Pod Scheduling Preferences API (Policy/Optimization)
3. Capacity Provisioning API (Actuation)
These APIs represent a fundamental shift in how Kubernetes handles cluster scaling, moving from opaque, reactive autoscaling to a transparent, proactive, and workload-aware provisioning model.
Context: The Workload Aware-Scheduler
The Workload Aware-Scheduler is an architectural evolution designed to unify Kubernetes scheduling and cluster autoscaling. Historically, these two functions have operated in separate control loops: the scheduler places pods on existing nodes, and the autoscaler reacts only when pods fail to schedule.
WAS integrates these functions to solve two critical problems:
1. The "Blind Scheduler" Problem[a]: The scheduler is unaware of what could exist, leading to suboptimal placement on fragmented resources rather than provisioning optimized hardware.
2. The "Preference Gap"[b]: Even if the scheduler sees potential capacity, it lacks the business logic to strictly prioritize between options[c] (e.g., "Always try Spot first, then On-Demand"). It often chooses the more expensive path simply because it fits or has a slightly better bin-packing score.
To achieve this, WAS decouples the monolithic logic of the current Cluster Autoscaler into distinct logical components, bridged by the APIs defined below.
Goals
The introduction of these APIs aims to achieve the following objectives for the scheduling and scaling ecosystem:
* Expose Unprovisioned Capacity: To provide a standardized mechanism for discovering purchasable but currently unprovisioned compute capacity within the cluster.
* Explicit Provisioning[d][e][f]: To move node provisioning from an implicit side-effect of pending pods to an explicit, declarative action driven by the scheduler.
* Deterministic Cost Optimization: To enforce strict ordering of infrastructure choices (e.g., prioritizing Spot over On-Demand) rather than relying on soft, unpredictable affinity weights.
* Easy Integration with Existing APIs: To facilitate seamless integration with established cloud provider configurations and autoscaling APIs, such as GKE Custom Compute Classes (CCC) and Karpenter NodePools.
* Decouple Discovery from Scheduling: To abstract the complexities of cloud inventory management (quotas, SKUs, zones) from the logic of workload scheduling.
* Enable Low-Cost Simulation: To allow schedulers to perform lightweight "fit checks" and "what-if" scenarios against non-existent hardware.
* Atomic (Gang) Provisioning: To support "all-or-nothing" provisioning requests, ensuring that groups of nodes are created together to prevent resource deadlocks.
* Provide Dynamic Signals: To expose real-time metadata such as availability probabilities inform intelligent placement decisions.
* Race Condition Mitigation: To solve "Time-of-Check to Time-of-Use" issues by accepting prioritized fallback options within a single atomic request transaction.
* Lifecycle Management: To provide a clear handle for tracking the status of a provisioning request from initialization to node readiness.
Non-Goals
To clearly define the boundaries of this proposal, the following are explicitly out of scope:
* End-User Consumption: These APIs are infrastructure integration surfaces for controllers and schedulers, not necessary direct interfaces for application developers.
* Workload Definition: These APIs are not intended to replace high-level workload specifications like Pods, Jobs, or custom Workload APIs.
* Manual Scaling: Intended for automated controllers, not for manual scaling operations by end-users.
* Replacing the Node API: These APIs do not represent existing, provisioned VMs; that remains the domain of the Kubernetes Node API.
* Actuation within Discovery: The Potential Capacity API is strictly read-only and does not handle purchasing or provisioning resources.
* Discovery within Actuation: The Capacity Provisioning API does not provide information about what can be bought; it assumes the caller has already validated feasibility via the Potential Capacity API.
The Architectural Shift: Decoupling Discovery, Policy, and Actuation
In the current Kubernetes ecosystem, the logic for discovering capacity, deciding what to buy, and requesting it is bundled tightly within the Cluster Autoscaler.
The proposed design splits these concerns into three distinct phases:
1. Discovery: A stateless, high-volume phase where the scheduler explores possibilities ("What can I buy?").
2. Policy[g][h][i]: A rule-based phase where the scheduler filters options based on business intent ("What should I buy first?").
3. Actuation: A stateful, transactional phase where the scheduler commits to a decision ("Buy this specific capacity").
High-Level Positioning of the APIs
These three APIs act as the bridge between the Workload Scheduler (the "brain" making placement decisions) and the Cluster Capacity Provider (the "muscle" managing cloud inventory and quotas).
Potential Capacity API (The "Menu")
* Role: Infrastructure Discovery & Feasibility
* Function: This API provides a standardized, read-only view of the "potential" nodes that could be created in the cluster. It abstracts away the complexities of cloud provider SKUs, quotas, and regional availability.
* Key Value: It empowers the scheduler to perform "fit checks" against non-existent hardware. It exposes dynamic signals-such as spot availability probabilities and relative cost scores-allowing the scheduler to make cost-aware and reliability-aware decisions before a workload is even bound to a node.
Pod Scheduling Preferences API (The "Policy")
* Role: Optimization & Business Logic
* Function: This API serves as the "Optimization Layer," allowing administrators to centrally define strict, prioritized scheduling preferences (e.g., "Try Spot, then On-Demand") for specific workloads.
* Key Value: It solves the "Preference Gap." Unlike soft Kubernetes affinity weights which are often ignored, this API enforces a deterministic waterfall logic. It ensures that the scheduler exhausts the most cost-effective or reliable infrastructure options defined by policy before falling back to expensive alternatives.
Capacity Provisioning API (The "Order")
* Role: Infrastructure Actuation & Lifecycle Management
* Function: This API is the mechanism for explicitly requesting the creation of new nodes. It accepts a specific configuration (selected from the "Menu" and filtered by "Policy") and manages the lifecycle of that provisioning request.
* Key Value: It moves provisioning from an implicit side-effect of pending pods to an explicit, declarative act. Crucially, it supports atomic (gang) provisioning, ensuring that for batch workloads (like distributed training), either all required nodes are provisioned together, or none are.
Strategic Fit in the WAS Ecosystem
Within the layered architecture of WAS, these APIs fulfill specific integration roles:
* Integration Surface: They serve as the standardized contract between the generic Workload Scheduler and the cloud-specific Cluster Capacity Provider. This allows the scheduler to remain cloud-agnostic while leveraging provider-specific optimizations (like AWS Karpenter or GKE Node Auto-Provisioning) through the provider implementation.
* Enabling Proactive Scheduling: By exposing potential capacity as a first-class citizen, the Workload Scheduler can simulate placement[j] on "virtual nodes." This allows it to optimize for bin-packing and cost across both existing and future capacity simultaneously, rather than optimizing locally for one and then the other.
* Safety & Stability: By separating the "read" path (Potential Capacity) from the "write" path (Capacity Provisioning), the system becomes more robust. Heavy simulation loops by complex batch schedulers can hammer the Discovery API without overloading the provisioning control plane or risking accidental scale-ups.
Implementation Model: The Unified Cloud Provider Component
While distinct in function, these APIs are designed to be implemented or consumed by a single, cohesive cloud-provider-specific component (referred to as the Cluster Capacity Provider). This component acts as a sophisticated adapter layer that translates high-level Kubernetes intent into low-level cloud infrastructure actions.
This unified component handles three distinct translation flows:
A. Bridging Configuration and Discovery (The "Menu" Generation[k])
This component consumes provider-specific configuration APIs that define the "intent" or "constraints" for the cluster. Examples include:
* GKE Custom Compute Classes (CCC): Defining classes of compute (e.g., "Performance," "Cost-Optimized").
* Karpenter NodePools: Defining constraints like allowed zones, instance families, and purchase options.
The Cluster Capacity Provider watches these configuration resources and combines their constraints with real-time cloud inventory data to populate the Potential Capacity API[l]. This transformation allows the Workload Scheduler to see a standardized "menu" of potential nodes (e.g., "I can create an 8-core node in us-west1") without needing to understand the underlying complexities or syntax of GKE CCCs or Karpenter NodePools.
B. Bridging Intent and Policy (The "Rules" Generation)
Just as the provider translates inventory, it can also translate operator intent into scheduling rules[m]. For example, a GKE Compute Class defining "Spot with On-Demand fallback" is translated into a Pod Scheduling Preference with a strict priority list. This ensures the scheduler's behavior matches the cloud provider's best practices.
C. Bridging Actuation and Infrastructure (The "Order" Fulfillment)
When the Workload Scheduler submits a request via the Capacity Provisioning API[n], the Cluster Capacity Provider intercepts this standardized instruction. It then performs the necessary translation to execute the request against the specific cloud provider's infrastructure API.
* For GKE: It might translate a Capacity Request into internal GKE calls or GCE instances.insert operations tailored to the specific Compute Class.
* For Karpenter: It would translate the request into AWS EC2 RunInstances calls matching the specific NodePool constraints.
This model encapsulates the complexity of cloud-specific provisioning logic within a single component, presenting a clean, unified interface to the Workload Scheduler while retaining the full power and flexibility of the underlying cloud provider's native tooling.




Potential Capacity API Design Proposal
[Public] Potential Capacity API Design Proposal
Authors: Paweł Kępka
Status: Draft
Last Update:  Mar 19, 2026
Summary
The Potential Capacity API is a proposed infrastructure-level integration API designed to expose the purchasable but currently unprovisioned compute capacity of a Kubernetes cluster.
In the Workload Aware-Scheduler (WAS) architecture, this API serves as the "discovery layer," decoupling the complex logic of cloud inventory management from the logic of workload scheduling. By providing a standardized, read-only view of potential nodes - complete with predictive availability signals - it enables schedulers (like the Workload Scheduler) to perform accurate "fit checks" and simulations without incurring costs or triggering stateful provisioning operations.
This API is not intended for direct consumption by end-users (application developers). It is an integration surface between the Cluster Capacity Provider (the implementation aware of cloud quotas and stock) and the Workload Scheduler (the consumer making placement decisions).
Problem Statement
Current Kubernetes scheduling and autoscaling architectures operate on a reactive, feedback-loop basis that is becoming increasingly insufficient for complex, cost-sensitive, and AI/ML workloads.
The "Blind Scheduler" Problem
The Kubernetes kube-scheduler effectively treats the cluster as a static entity. It only sees nodes that already exist. If a pod cannot fit, it marks it as Pending. The Cluster Autoscaler (CA) then reacts to this signal to provision new nodes.
This creates a fundamental information gap:
* No Visibility into "What If": The scheduler cannot weigh the trade-offs between provisioning Node Type A (cheaper, slower) vs. Node Type B (expensive, faster) because it doesn't know Node Type B is an option until the Autoscaler decides to buy it.
* Inefficient Cost Optimization[o][p]: Because the scheduler acts before the new capacity exists[q], it often makes suboptimal placement decisions for pods, leading to fragmentation once the new nodes arrive.
The Simulation Gap
Advanced orchestrators (like batch schedulers for HPC or AI training) need to simulate future cluster states to determine if a massive job can run. Currently, "expandable capacity" is hidden logic within the Cluster Autoscaler. External components cannot query the Cluster Autoscaler to ask, "If I were to submit this job, exactly what hardware would you buy?" without actually submitting the job and triggering potential scale up.
Opacity of Cloud Constraints
Cloud providers have complex constraints: regional quotas, stockouts, Spot preemption rates, and zonal restrictions. Currently, these are opaque to the scheduler. A scheduler might keep a pod Pending in Zone A indefinitely, unaware that Zone A is completely out of stock for the required GPU type, while Zone B has ample capacity.
Core Concepts
To clarify the distinction between the static definition of capacity and the dynamic simulation of it, we define two core concepts:
1. Potential Node Template
The Potential Node Template is the static definition of a node or group of nodes that can be provisioned. It is an infrastructure-level abstraction that aggregates compatible cloud SKUs, availability data, and constraints.
* Representation: In the API, this is represented by the ClusterPotentialCapacityPool and ClusterPotentialInstanceType resources.
* Nature: Read-only, shared across schedulers, and relatively stable (changes when stock/quota changes).
* Role: Acts as the "Menu" of purchasable options.
2. Potential Node Instance
The Potential Node Instance is a dynamic, transient representation created internally by the Workload Scheduler during the scheduling cycle. It represents a specific, hypothetical node that can be created from a Template.
* Representation: Internal scheduler object (mapping to a Kubernetes Node object in memory).
* Virtual Node Behavior: These instances are injected into the scheduling cycle as "virtual nodes”. The kube-scheduler treats them as valid targets for pod placement, allowing standard scheduling plugins (Filter, Score) to operate on them alongside regular nodes.
* Identification Label: Each virtual node carries a special label (e.g., capacity.k8s.io/type: potential). This allows workloads to express preferences via NodeAffinity - for example, a workload can prefer running on existing capacity (capacity.k8s.io/type: existing) to avoid spin-up latency, while still falling back to potential capacity if necessary.
* Nature: Ephemeral, unique to a specific simulation.
* Role: Acts as the "Simulated Node" used for feasibility checks.
* Lifecycle Transition: At the end of a successful scheduling cycle, any Potential Node Instances with assigned workloads are transformed into formal requests for actual nodes using the Capacity Provisioning API, ensuring the simulated capacity becomes physical infrastructure.
* Differentiation: Unlike the Template, an Instance has:
   * Identity: A unique placeholder name.
   * Specifics: While the Template might allow "Zone A or Zone B", an Instance might be tentatively bound to "Zone A" during simulation.
   * Additional Constraints: It may carry specific labels (e.g., team=frontend) injected during simulation if allowed by the Template's governance rules.
Property Classifications
To manage the complexity of metadata on potential nodes, we classify labels and taints into three distinct categories. These definitions are used throughout the API and Scheduler logic.
1. Capacity Pool Properties[r][s][t][u]
* Definition: Static, immutable properties (labels or taints) defined on the ClusterPotentialCapacityPool template.
* Scope: Shared by every Potential Node Instance derived from the pool.
* Purpose: Primarily used to identify the group of nodes or capabilities common to the entire pool.
* Examples: hardware-type=gpu, cloud.google.com/gke-nodepool=pool-v1, failure-domain=region-us-central1.
2. Instance Properties
* Definition: Properties that define the specific physical characteristics of a provisioned node.
* Scope: Defined as Constraints on the ClusterPotentialInstanceType but resolved to a specific value on a Potential Node Instance.
* Purpose: Express provisioning-specific properties like the type of VM, the zone, or the billing model. These directly impact the physical infrastructure creation.
* Examples: node.kubernetes.io/instance-type=n2-standard-8, topology.kubernetes.io/zone=us-central1-a, provisioning-model=spot.
3. User Properties
* Definition: Properties provided by the workload to organize nodes logically.
* Scope: Strictly governed by the pool's allowedAdditionalLabels and allowedAdditionalTaints policies. These properties are not defined in the pool's static template or constraints but are attached dynamically.
* Purpose: Allow users to assign nodes to specific teams, cost centers, or logical groupings without impacting the physical infrastructure specification (from the cloud provider's perspective).
* Examples: team=frontend, environment=staging, cost-center=1234.
API Definition
The Potential Capacity API defines a mechanism to discover what can be created. It does not represent specific, provisioned VMs (which are covered by the Node API) nor the intent to buy them (which is covered by the Capacity Provisioning API).
Conceptual Model: The "Menu"
Think of the Node API as the food currently on your table. Think of the Potential Capacity API as the menu. It tells you what can be ordered, what it costs, and the likelihood of the kitchen running out of ingredients (availabilitySignal).
Resource Model
The API relies on two primary resources to separate the detailed definition of an instance from the logical grouping of those instances.
This refactoring normalizes the representation of potential capacity, separating the concept of a pool (a collection of potential instance types) from the detailed characteristics of each type.
1. ClusterPotentialInstanceType Resource
This resource defines a specific type of instance that can be provisioned. It's a template for a set of homogeneous instances.


apiVersion: "capacity.k8s.io/v1alpha1"
kind: "ClusterPotentialInstanceType"
metadata:
 name: "n2-standard-2"
spec:
 capacity:
   cpu: "2"
   memory: "8Gi"
   # Add other capacity resources like ephemeral-storage, gpus, etc.
 allocatable:
   cpu: "1900m"
   memory: "7Gi"
   # Add other allocatable resources

 labels:
   instance-type: "n2-standard-2"
   cloud.google.com/machine-family: "n2"
   topology.kubernetes.io/region: "us-central1"
   # Add other identifying labels
 taints:
   # Taints specific for this instance type can be provided here.

 # These are constraints that must be met by the provisioner
 # or environment for this instance type.
 constraints:
    - key: "topology.kubernetes.io/zone"
      operator: In
      values: ["us-central1-a", "us-central1-b"]
    - key: "provisioning-model"
      operator: In
      values: ["on-demand", "spot"]
    # Placeholder Constraint for topology awareness
    - key: "topology.kubernetes.io/rack"
      placeholder: true[v][w]
status:
 systemOverhead:
   cpu: "50m"
   memory: "200Mi"

 # Defines overhead for DaemonSets expected on this GKE node[x][y]
 daemonSetOverhead:
   # Could be a list if multiple daemonsets are accounted for
   - name: "kube-proxy"
     resources:
       cpu: "20m"
       memory: "50Mi"
   - name: "metrics-agent"
     resources:
       cpu: "30m"
       memory: "100Mi"

 # Represents different purchasing options or slight variations
 # for this specific instance type.
 instanceVariations:
   - dimensions: # Specific attributes of this variation
       topology.kubernetes.io/zone: "us-central1-a"
       provisioning-model: "spot"
     availabilitySignal: "High"[z][aa][ab] # High, Medium, Low, None
     limit: 50 # Max instances of this variation
     cost: 4.20 # Relative or absolute cost indicator

2. ClusterPotentialCapacityPool Resource
This resource groups references to ClusterPotentialInstanceType resources, representing a collection of instance types available for provisioning, governed by shared policies or limits.


apiVersion: "capacity.k8s.io/v1alpha1"
kind: "ClusterPotentialCapacityPool"
metadata:
 name: "general-purpose-spot"
spec:
  # Capability Flag: Atomic Provisioning
  # Indicates whether the underlying infrastructure provider supports "all-or-nothing"
  # creation for a batch of nodes from this pool. 
  # If true, schedulers can safely request a "Gang" of nodes, preventing partial 
  # scale-ups and resource deadlocks for interdependent workloads.
  supportsAtomicProvisioning: true[ac][ad][ae]
  
  # Governance for User Properties: 
  # Control over what custom metadata can be attached during provisioning.
  # These fields accept a Regular Expression (Go RE2 syntax). [af][ag]
  # Only Label/Taint keys matching these patterns can be included in a 
  # CapacityRequest for this pool.
  #
  # Example: "^(team|cost-center|app\.kubernetes\.io/.*)$"
  # Use "^$" (matches only empty string) to forbid all User Properties.
 allowedAdditionalLabels: "^(team|project|cost-center)$"
 allowedAdditionalTaints: "^(team|project|cost-center)$"

 # Lists the instance types belonging to this pool.
 instanceTypeRefs:[ah][ai]:
   - name: "n2-standard-2"
   - name: "n2-standard-4"
   # Other ClusterPotentialInstanceType names

  # The Template defines the Capacity Pool Properties.[aj][ak]
 # Labels/Taints applied to all instances provisioned from this pool,
 # in addition to those on the ClusterPotentialInstanceType.
  template[al][am][an]:
   labels:
     pool: "general-purpose-spot"
   taints: []


  constraints:
    - key: "provisioning-model"
      operator: In
      values: [""spot"]
status:
 # Overall limit for the pool, if different from sum of instance types.
 limit: 200

Rationale for Structure:
* Normalization: Separating InstanceType from Pool avoids data duplication. Common attributes for a specific machine type, zone, and pricing model are defined once in ClusterPotentialInstanceType. ClusterPotentialInstanceType can be shared between ClusterPotentialCapacityPools.
* Clearer Abstraction: ClusterPotentialInstanceType represents a concrete, provisionable unit, while ClusterPotentialCapacityPool represents a logical grouping with optional additional constraints.
* Flexibility: Pools can group instance types based on various criteria (e.g., all spot instances, all instances in a region, instances for a specific team) by referencing the appropriate ClusterPotentialInstanceType objects.
* Granular Status: Dynamic signals like availabilitySignal and limit are naturally scoped to the ClusterPotentialInstanceType or even finer within its instanceVariations.
Dynamic Signals
A static list of machine types is insufficient. The API differentiates itself by exposing dynamic, computed signals per instance type variation:
* limit: hard limit on how many nodes can be bought from this pool or variation.
* availabilitySignal: A predictive score derived from:
   * Cloud Inventory: Real-time stock checks.
   * Quotas: Remaining project quotas.
   * Historical Data: Spot interruption rates for the specific instance types in the allowed zones.
Example: Mapping a GKE Node Pool
To illustrate how this API abstracts existing infrastructure concepts, consider a standard GKE Node Pool configured with autoscaling.
Source GKE Node Pool Configuration:
* Name: data-processing-pool
* Machine Type: n2-standard-8 (8 vCPU, 32GB Memory)
* Zone: us-central1-f
* Autoscaling: Min 0, Max 20
* Labels: workload=batch-processing
Corresponding Resources:


apiVersion: "capacity.k8s.io/v1alpha1"
kind: "ClusterPotentialInstanceType"
metadata:
  name: "n2-standard-8"
spec:
  capacity:
    cpu: "8"
    memory: "32Gi"
    # Add other capacity resources like ephemeral-storage, gpus, etc.
  allocatable:
    cpu: "7900m"
    memory: "28Gi"
    # Add other allocatable resources


  labels:
    instance-type: "n2-standard-8"
    cloud.google.com/machine-family: "n2"
    topology.kubernetes.io/region: "us-central1"
    # Add other identifying labels
  taints:
    # Taints specific for this instance type can be provided here.


  # These are constraints that must be met by the provisioner
  # or environment for this instance type.
  constraints:
    - key: "topology.kubernetes.io/zone"
      operator: In
      values: ["us-central1-a", "us-central1-b", "us-central1-f"]
    - key: "provisioning-model"
      operator: In
      values: ["on-demand", "spot"]
status:
  systemOverhead:
    cpu: "50m"
    memory: "200Mi"


  daemonSetOverhead:
    # Could be a list if multiple daemonsets are accounted for
    - name: "kube-proxy"
      resources:
        cpu: "20m"
        memory: "50Mi"
    - name: "metrics-agent"
      resources:
        cpu: "30m"
        memory: "100Mi"


  # Represents different purchasing options or slight variations
  # for this specific instance type.
  instanceVariations:
    - dimensions: # Specific attributes of this variation
        topology.kubernetes.io/zone: "us-central1-f"
        provisioning-model: "spot"
      availabilitySignal: "High" # High, Medium, Low, None
      limit: 50 # Max instances of this variation
      cost: 4.20 # Relative or absolute cost indicator



apiVersion: "capacity.k8s.io/v1alpha1"
kind: "ClusterPotentialCapacityPool"
metadata:
 name: "gke-data-processing-pool"
 labels:
   "cloud.google.com/gke-nodepool": "data-processing-pool"
spec: 
 # Standard GKE Node Pools (MIGs) do not support atomic scale-up (gang provisioning).
 supportsAtomicProvisioning: false

 # Strict Governance for User Properties on Legacy Node Pools
 allowedAdditionalLabels: "^$" # Matches empty string only - forbids all keys.
 allowedAdditionalTaints: "^$" # Matches empty string only - forbids all keys.

 template:
   # Capacity Pool Properties
   labels:
     "workload": "batch-processing"
     "cloud.google.com/gke-nodepool": "data-processing-pool"
   taints: [] 

 instanceTypeRefs:
   - name: "n2-standard-8"


  constraints:
    - key: "topology.kubernetes.io/zone"
      operator: In
      values: ["us-central1-f"]
    - key: "provisioning-model"
      operator: In
      values: ["spot"]

status:
 limit: 20

This mapping allows the Workload Scheduler to see this specific Node Pool as just another source of potential capacity, alongside more dynamic sources like auto-provisioned pools (NAP).
Design Decisions
* Decision: Decoupling Discovery from Actuation
   * Design: We separate the API for finding capacity (Potential Capacity API) from the API for buying capacity (Capacity Provisioning API).
   * Rationale:
      * Scalability: Schedulers run thousands of "fit check" simulations. Discovery is a high-volume, read-heavy operation. Provisioning is a slow, low-volume, write-heavy operation. Bundling them creates bottlenecks.
      * Safety: Schedulers need to perform "what-if" analysis. Querying this API is safe and side-effect-free. A unified API would risk accidental provisioning during simulation phases.
* Decision: Instance Types over Exhaustive SKU Lists
   * Design: The API uses ClusterPotentialInstanceType to encapsulate exact compute properties and groups them logically in ClusterPotentialCapacityPools, providing clear options without overwhelming the scheduler with flat SKU lists.
   * Rationale:
      * Cardinality: Cloud providers have hundreds or thousands of SKUs. A flat list is unwieldy and slow to process.
      * Fungibility: Workloads often don't care about the exact CPU generation (e.g., Broadwell vs. Skylake) as long as they get 4 vCPUs. Grouping these references allows the Cluster Capacity Provider to optimize the specific choice at the moment of provisioning.[ao]
* Decision: Positioning as Infrastructure API
   * Design: This API is positioned as a low-level integration point for controllers (schedulers, optimizers), not an interface for end-users (developers).
   * Rationale: Application developers should express intent via high-level APIs like Workload or Pod specs. They should not be selecting specific CapacityPools manually. This API allows the system to fulfill that intent automatically.
* Decision: Placeholder Constraints for Topology
   * Design: The API supports constraints with placeholder: true. These define keys (e.g., rack-id) where the value is unknown during discovery but can be arbitrarily assigned during provisioning to satisfy topology requirements.
   * Rationale:
      * Topology Awareness: Workloads often require "All pods on the same rack" or "Pods spread across distinct switches."
      * Unknown Values: The Cloud Provider cannot predict that it will give you "Rack #54" until you actually place the order. However, it can promise that it has the capability to put some number of[ap][aq][ar][as][at] nodes in a single rack if there is a capacity for this. Placeholders allow the scheduler to simulate this "grouping" capability without knowing the physical ID.
Scheduler Integration Logic
To utilize the information from the Potential Capacity API, the Workload Scheduler must bridge the gap between abstract API responses and the concrete Node objects that the Kubernetes scheduling cycle expects. This is achieved through the generation of "Potential Node Instances."
Generating Potential Node Instances
The scheduler periodically polls or watches the ClusterPotentialCapacityPool and associated ClusterPotentialInstanceType resources. To maintain performance and minimize memory overhead, the scheduler employs a Just-in-Time (JIT) Expansion strategy.
Instead of pre-generating a large number of nodes to represent total potential capacity, the scheduler injects one Potential Node Instance for each ClusterPotentialInstanceType referenced in spec.instanceTypeRefs for a given ClusterPotentialCapacityPool into the scheduling cycle.
1. Iterate Instance Types: The scheduler iterates over the instanceTypeRefs list and fetches the corresponding resource.
2. Instantiate Potential Node Instance: For each referenced instance type, it creates a JIT Potential Node Instance inheriting:
   * Capacity Pool Properties: Applied directly as labels and taints from the pool template.
   * Instance Properties: The constraints defined on the ClusterPotentialInstanceType.
   * Specific Capacity: capacity and allocatable from the ClusterPotentialInstanceType.
3. Cycle Injection: If a pod is tentatively assigned to one of these Instances, the scheduler immediately duplicates it to ensure the pool remains "available" for subsequent pods in the same cycle (up to the limit).
This dynamic expansion mechanism allows the scheduler to simulate the provisioning of an arbitrary number of nodes across different sizes without polluting the scheduler's cache with thousands of unused nodes during idle periods.
Transformation Logic
* Capacity & Allocatable: The capacity and allocatable from the specific ClusterPotentialInstanceType are mapped to the Instance's status.
* Dynamic Signal Injection: The scheduler looks up the corresponding status entry for the instance type in the pool's status.instanceTypeStatuses or the instance's status.instanceVariations. It injects availabilitySignal as annotations (e.g., capacity.k8s.io/availability-signal: "High") onto the Instance.
* Fungible Instance Properties: The constraints field (e.g., zone IN [us-west1-a, us-west1-b]) presents a unique challenge. A standard Kubernetes Node belongs to exactly one zone. However, a Potential Node Instance represents the potential to become a node in any of those zones. To handle this, the scheduler stores these constraints as internal metadata or special annotations on the Instance (e.g., capacity.k8s.io/potential-zones: "us-west1-a,us-west1-b").
* Placeholder Injection: For every constraint with placeholder: true, the scheduler generates synthetic values (e.g., virtual-rack-1, virtual-rack-2) on the Instances. This allows the simulation to test topology spread constraints (e.g., "MaxSkew=1" on topology.kubernetes.io/rack).
Required Changes to NodeAffinity Plugin
The standard NodeAffinity plugin in kube-scheduler performs exact matching between a Pod's requirements (e.g., requiredDuringSchedulingIgnoredDuringExecution) and a Node's labels. This logic breaks when applied to fungible Potential Node Instances.
The Mismatch:
* Pod Requirement: topology.kubernetes.io/zone In [us-west1-a]
* Potential Node Instance Label: The node is not yet in a zone. It has the potential to be in us-west1-a OR us-west1-b.
* Standard Behavior: If the Instance has no zone label, the match fails. If we arbitrarily label it us-west1-b, the match fails (false negative).
Proposed Change: "Satisfiability" Logic
The NodeAffinity plugin must be extended (or wrapped) to support Constraint Satisfiability Checking when evaluating Potential Node Instances.
Instead of checking Node.Labels == Pod.Selector, the logic becomes:
* Input: Pod Requirement set R, Potential Node Instance properties (consisting of P-pool for Capacity Pool Properties and P-instance for Instance Properties), and Allowed User Properties Regex L-regex (from spec.allowedAdditionalLabels).
* Logic: For each requirement (key, operator, values) in R:
   1. Check Capacity Pool Properties: Does the requirement match the static labels P-pool defined on the pool template? (Standard NodeAffinity matching).
   2. Check Instance Properties: If not found in P-pool, does the requirement overlap with the Instance's defined constraints P-instance? (Is there a non-empty intersection between R and P-instance?)
   3. Check User Properties: If the key is not present in P-pool or P-instance, does the key match L-regex?
* If any condition is met, the requirement is considered Satisfiable.
* Example 1 (Instance Property Overlap): If the Pod requires zone=us-west1-a and the Instance Type allows zone=[us-west1-a, us-west1-b], there is a valid intersection. The Instance is a match.
* Example 2 (User Property): If the Pod requires team=frontend and the Pool has no team label but sets allowedAdditionalLabels="^team$", the requirement is satisfiable. The scheduler knows it can attach team=frontend during the Capacity Request.
This change allows the scheduler to treat a ClusterPotentialCapacityPool as a valid placement target for any pod whose constraints overlap with the pool's capabilities or can be fulfilled dynamically via allowed custom labels. Once a Potential Node Instance is selected for a pod, the scheduler can then "lock in" the specific parameters (e.g., selecting us-west1-a specifically) when generating the subsequent Capacity Request.
Cost-Aware Scheduling on Potential Nodes
To enhance cost-effectiveness in cluster operations, the Workload Scheduler can be configured to consider cost implications when selecting among different ClusterPotentialInstanceType options for provisioning new nodes. This can be achieved through a dedicated scheduler plugin operating at the Score extension point, for instance named PotentialNodeCost.
Cost Information
The primary source of cost data is the cost field within the ClusterPotentialInstanceType.status.instanceVariations array. This field provides a numerical representation of the relative total cost for a particular instance type variation (e.g., for a specific zone and provisioning model like On-Demand vs. Spot).
Configuration
The PotentialNodeCost plugin requires configuration for relative resource weights and waste surcharge factors, provided via the scheduler configuration:
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
    plugins:
      score:
        enabled:
          - name: PotentialNodeCost
    pluginConfig:
      - name: PotentialNodeCost
        args:
          # Give PotentialNodeCost plugin a high weight so that cost will be
          # primary faactor when selecting potential nodes.
          weight: 10
          # Relative weights of resources. The scale doesn't matter, only the ratios.
          # Example: 1 CPU is weighted equivalent to 8 GiB of Memory in terms of cost.
          resourceWeights:
            cpu: "8.0"
            memory: "1.0"
            ephemeral-storage: "0.02"
            "nvidia.com/gpu": "350.0"
          # Factors to penalize unused resources for each resource type,
          # between 0.0 and 1.0.
          # 0.0 means unused resources of this type are not penalized.
          # 1.0 means unused resources of this type are fully counted as waste.
          wasteSurchargeFactors:
            cpu: "0.5"
            memory: "0.5"
            ephemeral-storage: "0.5"
            "nvidia.com/gpu": "0.0"
Scoring Logic
The plugin calculates a score (0-100) for each Potential Node Instance Variation, where higher scores indicate more cost-effective choices.
For existing nodes the plugin should return 0.
The core metric is the Cost Efficiency Metric (CEM), which represents the cost per unit of "effective weighted resource utilization". A lower CEM is better.
Let:
* R be the set of resource names (e.g., cpu, memory).
* weight[r] be the resourceWeights for resource r.
* wasteFactor[r] be the wasteSurchargeFactors for resource r.
* allocatable[r] be the node's allocatable amount for resource r.
* request[r] be the pod's requested amount for resource r.
* NodeBaseCost be the instanceVariation.cost.
Score Calculation Steps:
1. Calculate Weighted Pod Request (WPR): The total weighted resources the pod requests. WPR = sum(request[r] * weight[r] for r in R)
2. Calculate Weighted Unused Surcharge (WUS): The penalty for unused resources, weighted and summed across all resource types. Unused[r] = max(0, allocatable[r] - request[r]) WUS = sum(Unused[r] * weight[r] * wasteFactor[r] for r in R)
3. Calculate Cost Efficiency Metric (CEM): CEM = NodeBaseCost / (WPR + WUS) A lower CEM indicates better cost efficiency.
4. Normalization to Score (0-100): Since a lower CEM is preferable, the normalization process must invert this relationship to produce a score where higher is better, as expected by the Kubernetes scheduler framework. The plugin will calculate the CEM for all potential node variations for the given pod. Let CEM_i be the metric for variation i, and MinCEM be the minimum CEM observed across all variations. The score for variation i can be calculated as: Score_i = (MinCEM / CEM_i) * 100 This formula ensures the variation with the lowest CEM receives a score of 100, and others receive proportionally lower scores.
Example Scenario
Pod requests: 2 CPU, 8Gi Memory.
Configuration:
* resourceWeights: { cpu: "8.0", memory: "1.0" }
* wasteSurchargeFactors: { cpu: "0.5", memory: "0.5" }
Weighted Pod Request (WPR): WPR = (2 * 8.0) + (8 * 1.0) = 16 + 8 = 24
Option A: n2-standard-4
* Allocatable: 4 CPU, 16Gi
* NodeBaseCost = 100
* Unused: { CPU: 2, Memory: 8 }
* WUS_A = (2 * 8.0 * 0.5) + (8 * 1.0 * 0.5) = 8 + 4 = 12
* CEM_A = 100 / (24 + 12) = 100 / 36 = 2.778
Option B: n2-standard-8
* Allocatable: 8 CPU, 32Gi
* NodeBaseCost = 180
* Unused: { CPU: 6, Memory: 24 }
* WUS_B = (6 * 8.0 * 0.5) + (24 * 1.0 * 0.5) = 24 + 12 = 36
* CEM_B = 180 / (24 + 36) = 180 / 60 = 3.000
Comparison: CEM_A (2.778) is lower than CEM_B (3.000), indicating Option A is more cost-effective.
Scores: MinCEM = 2.778
* Score_A = (2.778 / 2.778) * 100 = 100
* Score_B = (2.778 / 3.000) * 100 = 92.6
The scheduler would favor Option A.
Integration with TopologyPlacementPlugin
The TopologyPlacementPlugin is a component of the Workload Scheduler responsible for generating candidate Placements based on topological domains (e.g., racks, blocks). To fully leverage potential capacity, this plugin integrates with the Potential Node Instance model described above.
Domain Discovery from Potential Node Instances
The plugin scans both existing nodes and the Potential Node Instances injected by the scheduler.
* Concrete Domains: For Instances with fixed topology labels (e.g., zone=us-west1-a), the plugin generates standard Placements.
* Synthetic Domains: For Instances with placeholder labels (e.g., rack=virtual-rack-1), the plugin generates "Virtual Placements." These represent a promise from the provider that a group of nodes can be provisioned within a single topological domain, without specifying the physical ID upfront.
Placement Generation & Selection
* Generation: The plugin produces Placements for these synthetic domains (e.g., Placement{ NodeSelector: rack=virtual-rack-1 }).
* Simulation: The scheduler simulates the workload on these Virtual Placements. The "Satisfiability" logic ensures that the Pod's topology constraints (e.g., topologyKey: rack) are considered satisfied by the virtual-rack-1 label.
* Actuation: If a Virtual Placement is selected, the scheduler translates the synthetic domain ID (virtual-rack-1) into a CapacityRequest with the corresponding placeholder configuration, ensuring the provisioned nodes land on a single physical rack.
Relation to Existing Ecosystems
GKE Compute Classes
Relationship: Input (Intent) vs. Output (Capability)
* GKE Compute Classes (CCC): Define the User Intent. A user creates a CCC named "AI-Training" that prioritizes "A3" machines, then falls back to "A2".
* Potential Capacity API: Defines the System Capability. The Cluster Capacity Provider reads the CCC. It checks cloud inventory. It then outputs a ClusterPotentialCapacityPool that represents the feasible intersection of the CCC's intent and the cloud's reality.
* Flow: User -> CCC -> Cluster Capacity Provider -> Potential Capacity API -> Workload Scheduler.
Karpenter
Relationship: Standardization and Interoperability
* Karpenter's NodePool: Karpenter currently uses its own NodePool CRD to define constraints. However, this logic is internal to the Karpenter controller. No other scheduler can "read" Karpenter's understanding of capacity.
* Potential Capacity API: This API effectively standardizes the "output" of a Karpenter-like system[au].
* Convergence: In a converged architecture, Karpenter could evolve to become the Cluster Capacity Provider. It would publish ClusterPotentialCapacityPools based on its configuration. This allows schedulers (like kube-scheduler) to read the capacity data generated by Karpenter without needing to replicate Karpenter's AWS/Azure integration logic.
Cluster Autoscaler
Relationship: Evolution from Implicit to Explicit
* Current State: The Cluster Autoscaler maintains an internal, in-memory model of "node groups" (cloud provider specific) to run simulations. This model is inaccessible to the outside world.
* Future State: The Potential Capacity API exposes this internal model as a first-class API resource. This allows the Workload Scheduler to run the same simulations the Autoscaler currently runs, but before pod placement, unifying the decision logic.
Conclusion
The Potential Capacity API is a necessary evolution for Kubernetes to support workload-aware scheduling. By elevating "potential nodes" to first-class API citizens, we bridge the gap between abstract workload requirements and concrete infrastructure constraints. This enables a new generation of schedulers that are cost-aware, availability-aware, and capable of handling the complex demands of AI/ML workloads without provider lock-in.
Open Problems
DaemonSet Simulation
Which component will simulate scheduling of DaemonSets on a Potential Node Instance? The same problem applies to Upcoming Node. Accurate simulation is crucial for determining the true allocatable capacity for user workloads, as DaemonSets consume resources on every node.
Solutions for DaemonSet Simulation:
1. Pre-calculation by the Cluster Capacity Provider: The provider watches all DaemonSets and subtracts their resource requirements from the node capacity when generating ClusterPotentialInstanceType resources. The daemonSetOverhead and systemOverhead fields then reflect the capacity used by DaemonSet pods. This keeps scheduler logic simple but duplicates some matching logic in the provider.
2. Scheduler-Side Simulation (Virtual DaemonSet Injection)[av][aw]: The Workload Scheduler dynamically scans for active DaemonSets when creating a "Potential Node Instance". It identifies matching DaemonSets and virtually "schedules" them on the node before attempting to place user workloads. This offers the highest accuracy and responsiveness to changes but adds computational overhead during simulation.
3. DaemonSet Controller Integration: The DaemonSet controller monitors ClusterPotentialCapacityPool resources. It identifies which DaemonSets are schedulable on a given pool's template and adds this information (e.g., resource overhead) directly to the ClusterPotentialInstanceType object. This decentralizes the logic but requires modifying the DaemonSet controller to interact with the new API.
Long-term Recommendation: Solution 3 (DaemonSet Controller Integration) is the preferred approach. It decentralizes the overhead calculation, ensuring that the component with the most knowledge about DaemonSet requirements (the controller itself) is responsible for declaring its impact on potential capacity. This avoids duplicating complex matching logic in the Cluster Capacity Provider and reduces simulation overhead in the Workload Scheduler.
Dynamic Resource Allocation (DRA) Integration
Kubernetes Dynamic Resource Allocation (DRA) moves resource management for devices (GPUs, FPGAs) out of the core node status capacity fields and into ResourceClaim and ResourceSlice objects managed by external drivers.
The Challenge: The Workload Scheduler relies on "Potential Node Instances" with static capacity and allocatable fields to perform simulation. With DRA, the resource availability logic is hidden behind the DRA driver.
Open Questions:
* How does a ClusterPotentialCapacityPool advertise its ability to satisfy specific ResourceClaim templates (e.g., "I can provide nodes compatible with intel.com/fpga driver")?
* Does the Cluster Capacity Provider need to generate "Virtual ResourceSlices" corresponding to the "Potential Node Instances"?[ax][ay]
* How do we prevent a "Potential Node Instance" from accepting a DRA claim if the underlying cloud infrastructure (which doesn't exist yet) cannot actually attach that specific device model?
Bin-Packing and Node Size Optimization
There is an open problem related to the logic of selecting between smaller and larger nodes for provisioning when scheduling multiple pods at once. In such cases, the system should ideally prefer using larger nodes, as those have lower per-node overheads (for instance, related to DaemonSets).
* Current Limitation: The current pod-by-pod scheduling logic will most likely prefer using smaller nodes or simply the first feasible option, failing to optimize for the total resource footprint of the workload batch.
* Path Forward: This limitation should be addressed in further improvements of the kube-scheduler workload scheduling logic, enabling it to consider aggregate efficiency and favor larger ClusterPotentialInstanceTypes when appropriate.
Representation of Node Slices
There is an open problem regarding the representation of "Slices" - groups of nodes that are provided and interconnected together, common in TPU or high-performance GPU environments.
* The Challenge: The current model assumes ClusterPotentialInstanceTypes represent individual nodes. It does not natively support the concept of a "Slice" (e.g., a TPU v4-32 Pod) where the unit of provisioning is a fixed topology of multiple nodes, rather than a count of individual machines.
* Proposed Solution: The solution should include an option to indicate "potential slices" as part of the ClusterPotentialInstanceType. This would allow a ClusterPotentialInstanceType to define not just the resources of a single node, but the structure and aggregate capacity of an entire slice, enabling the scheduler to reason about slice-level fits and atomic gang provisioning requirements.
Stockout Signal Granularity for Fungible Constraints
There is an open problem regarding how the availabilitySignal reflects stockouts when a ClusterPotentialCapacityPool defines a set of fungible constraints (e.g., multiple zones or machine types) but only a subset of those options is unavailable.
* The Challenge: The API is designed to group related capacities into a single pool to reduce cardinality. However, this abstraction creates an information bottleneck when reporting dynamic signals like stockouts:
   * Constraint-Specific Failures: A stockout often applies to a specific intersection of properties (e.g., "Spot nodes in Zone A") rather than the entire pool. Current status reporting per ClusterPotentialInstanceType or instanceVariation provides some granularity but does not inherently capture the cross-section of global constraints like zones and provisioning models.
* Potential Solutions:
   * Sub-resource Signaling: Expanding the status to report availability per unique constraint combination. This increases API size and complexity, potentially leading back to the "Exhaustive SKU List" problem the design intends to avoid.
   * Backoff-Aware Scheduling: Instead of relying solely on the static availabilitySignal, the scheduler could treat a failed CapacityRequest for a specific zone as a temporary local constraint, while continuing to treat the rest of the pool as available.
Changelog
* Mar 19, 2026:
   * Refactor the ClusterPotentialCapacityPool resource by decoupling it into distinct ClusterPotentialCapacityPool and ClusterPotentialInstanceType entities to normalize the representation and support more granular availability signaling.
   * Integrate explicit cost metadata within the ClusterPotentialInstanceType resource and propose a scoring framework designed to utilize this information for the cost-aware optimization of potential capacity.
* Mar 4, 2026:
   * Remove priority from ClusterPotentialCapacityPool.
* Feb 6, 2026:
   * Initial proposal.

Pod Scheduling Preferences API Design Proposal
[Public] Pod Scheduling Preferences API Design Proposal
Authors: Paweł Kępka
Status: Draft
Last Update: Feb 6, 2026
Summary
The Pod Scheduling Preferences API is a proposed policy-level integration API designed to centrally define scheduling preferences based on node capacity types (e.g., Spot, On-Demand, Reserved).
In the Workload Aware-Scheduler (WAS) architecture, this API serves as the "Optimization Layer," bridging the gap between the infrastructure possibilities exposed by the Potential Capacity API and the placement decisions made by the Workload Scheduler.
The proposal introduces PodSchedulingPreferences, a CRD that allows cluster administrators to inject strict scheduling priorities into pods. This ensures that the scheduler automatically enforces the most cost-effective or reliable infrastructure - whether existing or potential - using a deterministic waterfall logic without requiring changes to application manifests.
Problem Statement
Current Kubernetes scheduling and autoscaling architectures suffer from a disconnect between infrastructure provisioning logic and workload placement logic.
The "Preference Gap"
While the Potential Capacity API provides the scheduler with a "menu" of what can be provisioned (e.g., Spot vs. On-Demand, N2 vs. C3), it does not tell the scheduler which option it should choose for a specific workload.
Without a centralized preference mechanism, the scheduler lacks the business logic to differentiate between two valid but economically different placement options. This leads to the Disconnected Provisioning Lifecycle problem:
   * Suboptimal Placement: The scheduler might place a batch job on an expensive On-Demand node simply because it "fits," ignoring a cheaper Spot option available in the Potential Capacity pool.
   * Inefficient Autoscaling: Autoscalers rely on the scheduler to "pack" nodes efficiently. If the scheduler doesn't prioritize specific capacity types, it cannot effectively "starve" underutilized nodes to trigger scale-down events.
Limitations of Workload-Level Affinity
Currently, Kubernetes allows users to define these preferences manually via the spec.affinity.nodeAffinity field in the Pod specification. While functional for individual workloads, this decentralized approach fails at scale:
   * Decentralized & Error-Prone: Relying on every team to correctly configure affinity rules leads to inconsistency and compliance gaps.
   * Weak Guarantees: Standard preferredDuringScheduling rules are only "hints." The scheduler often ignores them in complex bin-packing scenarios, leading to unpredictable cost outcomes.
   * Blurred Responsibilities: Application developers are forced to understand infrastructure pricing models and node labels, distracting them from their core application logic.
Core Concepts
To resolve these issues, we introduce the concept of centralized preferences with strict prioritization.
PodSchedulingPreferences
The PodSchedulingPreferences is a new CRD that allows cluster administrators to define a prioritized list of node requirements for pods matching a podSelector.
   * Decoupled Policy: It separates the platform's cost/reliability strategy from the application's deployment manifest.
   * Hard Priorities (Waterfall): Unlike soft affinity weights, this API enforces a strict "Try Option A; if impossible, Try Option B" logic, ensuring deterministic behavior.
API Definition
The Pod Scheduling Preferences API defines the rules of engagement for the scheduler. It does not create resources; it influences how resources are selected.
Resource Model: PodSchedulingPreferences
The core resource is the PodSchedulingPreferences CRD. It acts as a policy object that matches specific workloads and applies strict scheduling priorities.
To handle overlapping policies (e.g., multiple preferences matching the same pod), the system enforces strict Conflict Resolution rules:
   1. Priority Wins: The system evaluates the priority field of all matching PodSchedulingPreferences.
   2. Highest Value Selected: The single policy with the highest integer priority is selected and applied.
   3. Tie-Breaking: If multiple matching policies have the same priority, a deterministic tie-breaker (e.g., alphanumeric ordering of the resource name) is applied.
API Structure
Example 1: Complex Cost Optimization (Batch Workloads)
This example demonstrates a high-priority specific policy for batch jobs that aggressively targets Spot capacity.
apiVersion: scheduling.k8s.io/v1alpha1 
kind: PodSchedulingPreferences 
metadata: 
  name: batch-processing-defaults 
spec: 
  # Priority defines the precedence of this policy.
  # A higher value (100) ensures this specific policy overrides 
  # lower-priority global defaults.
  priority: 100


  # Selector to identify target workloads.
  # If a pod matches this selector, the priorities below are applied.
  podSelector: 
    matchLabels: 
      workload-type: batch-processing [az]


  # Defines a strict, ordered list of scheduling tiers.
  # The scheduler must attempt to schedule the pod on nodes matching the first priority.
  # Only if no capacity can be found for Priority 1 does it proceed to Priority 2.
  #
  # This example demonstrates the "Reuse then Provision" pattern enabled by the 
  # Potential Capacity API labels (capacity.k8s.io/type).
  schedulingPriorities:
    
    # Tier 1: Prefer filling gaps in EXISTING Spot nodes (Fastest & Cheapest)[ba]
    - nodeSelector:
        matchLabels:
          capacity.k8s.io/type: existing
          karpenter.sh/capacity-type: spot


    # Tier 2: If no existing Spot fits, provision NEW Spot capacity
    # This targets "Potential Node Instances" exposed by the Potential Capacity API
    - nodeSelector:
        matchLabels:
          capacity.k8s.io/type: potential
          karpenter.sh/capacity-type: spot


    # Tier 3: Fallback to EXISTING On-Demand nodes (Reuse expensive resources)[bb]
    - nodeSelector:
        matchLabels:
          capacity.k8s.io/type: existing
          karpenter.sh/capacity-type: on-demand


    # Tier 4: Last Resort - Provision NEW On-Demand capacity
    - nodeSelector:
        matchLabels:
          capacity.k8s.io/type: potential
          karpenter.sh/capacity-type: on-demand
Example 2: Global "Bin-Pack First" Strategy (Backward Compatibility)
This configuration acts as a low-priority global default. It mimics the standard behavior of legacy Cluster Autoscalers and Karpenter by prioritizing existing capacity first.
apiVersion: scheduling.k8s.io/v1alpha1
kind: PodSchedulingPreferences
metadata:
  name: cluster-wide-binpacking
spec:
  # Low priority ensures this is only applied if no specific 
  # workload policy (like Example 1) matches.
  priority: 10


  # Matches all pods (use with caution, or target specific namespaces)
  podSelector: {} 


  schedulingPriorities:
    
    # Tier 1: Existing Capacity (Global)
    # Force the scheduler to exhaust all existing slots (Spot or On-Demand)
    # before considering expansion.
    - nodeSelector:
        matchLabels:
          capacity.k8s.io/type: existing


    # Tier 2: Provision New Capacity
    # If no existing node fits, allow provisioning from the Potential Capacity pools.
    # The specific choice of capacity (Spot vs. OD) will be determined by other
    # constraints or the provisioner's internal defaults.
    - nodeSelector:
        matchLabels:
          capacity.k8s.io/type: potential
Strategic Fit in the WAS Ecosystem
This API functions as the decision-making logic that sits between Discovery and Actuation.
   1. Waterfall Filtering of "Potential" Capacity
When the Workload Scheduler generates "Potential Node Instances" from the Potential Capacity API, it filters them using the schedulingPriorities list.
      * Tier 1 Evaluation: The scheduler filters existing and potential nodes against the first entry in schedulingPriorities (e.g., Spot). If valid candidates are found, it proceeds to binding/provisioning immediately. Lower priorities are ignored.
      * Fallback Trigger: Only if no nodes fit the Tier 1 selector does the scheduler "unlock" the Tier 2 selector (e.g., On-Demand).
      * Performance Optimization: To mitigate the performance burden imposed by this multi-pass filtering process, the implementation is designed to check all available options concurrently and then efficiently select the highest-ranking candidate set in a single operation.
      2. Driving the Provisioning Decision
This mechanism guarantees that the Capacity Provisioning API receives a request for the highest-priority infrastructure that is physically feasible. It eliminates the "randomness" of weight-based scoring.
      3. The Autoscaling Feedback Loop
By enforcing strict priorities, the scheduler ensures that preferred capacity (e.g., Spot) is utilized to its absolute maximum before leaking any workloads to expensive capacity.
      4. Independent Operation (Standalone Mode)
While integrated into the WAS ecosystem, the PodSchedulingPreferences API is fully functional in standalone Kubernetes clusters that do not implement the Potential Capacity or Capacity Provisioning APIs.
         * Mechanism: In a standard cluster, the schedulingPriorities will simply filter existing nodes. For example, a "Tier 1: Spot" rule will restrict the pod to existing Spot nodes. If none are found, it falls back to "Tier 2: On-Demand."
         * Benefit: This allows administrators to use this API immediately for centralized policy management (e.g., "Always prefer existing Spot nodes") on static clusters or with legacy autoscalers, providing value even before full WAS adoption.
Integration with Infrastructure Configs
Just as the Potential Capacity API translates cloud inventory into a Kubernetes format, the Pod Scheduling Preferences API can be used to translate user intent into scheduling rules.
Integration with GKE Custom Compute Classes (CCC)
A GKE Compute Class defines a provisioning hierarchy (e.g., Prefer Spot -> Fallback to On-Demand).
         * The Controller: A platform controller can watch ComputeClass resources.
         * The Translation: It automatically generates a PodSchedulingPreferences object that maps the CCC hierarchy directly into the ordered schedulingPriorities list.
         * The Result: The scheduler's behavior on existing nodes now perfectly matches GKE's behavior when provisioning new nodes.
Integration with Karpenter NodePools
Karpenter NodePools often contain weighted constraints or priority fields.
         * The Controller: A controller monitors NodePool resources.
         * The Translation: It generates PodSchedulingPreferences where higher-priority NodePools become the top entries in schedulingPriorities.
         * The Result: Workloads are strictly gated to high-priority capacity until it is exhausted, preventing "leakage" onto expensive fallback options.
Key Design Decisions
Decision: Extending Scheduler vs. New Plugin
Design: The proposal requires a dedicated plugin (or extension of the NodeAffinity plugin) that supports "Waterfall" logic (Multi-Pass Scheduling) rather than simple weighted scoring.
Rationale:
         * Determinism: Weighted scoring (standard nodeAffinity) is unpredictable when mixed with other scores (ImageLocality, TopologySpread). Hard priorities guarantee that cost/capacity preference always wins.
         * Feedback Clarity: It provides clear signals why a pod is pending (e.g., "Waiting for Priority 1 Spot availability" vs "Waiting for generic resources").
Decision: Hard Priorities vs. Soft Preferences
Design: The CRD uses schedulingPriorities (an ordered list of requirements) instead of preferredDuringSchedulingIgnoredDuringExecution.
Rationale:
         * Strict Cost Control: Administrators often want guarantees (e.g., "Never use On-Demand unless Spot is truly gone"). Soft preferences cannot guarantee this; hard waterfall logic can.
         * Safety for Critical Workloads: To prevent unintended scheduling (e.g., accidental placement of critical system components on Spot nodes via a broad selector), the system is designed to respect explicit nodeAffinity defined in the PodSpec. Workloads that define their own affinity override these defaults, ensuring critical components remain safe.
         * Alignment with Provisioners: Cloud provisioners (like Karpenter or GKE NAP) operate on ordered lists/priorities, not fuzzy weights. This aligns the scheduler's logic with the underlying infrastructure's logic.
Interaction with Pod-Level Constraints
A critical aspect of this design is how the platform-defined PodSchedulingPreferences interact with affinity rules explicitly defined by application developers on the Pod spec. This interaction is governed by the Kubernetes Scheduling Framework phases.
Hierarchy of Controls:
         1. Highest Priority: Pod.spec.affinity.nodeAffinity.requiredDuringScheduling… (Hard Constraints)
         2. Middle Priority: PodSchedulingPreferences.schedulingPriorities (Waterfall Defaults)
         3. Lowest Priority: Pod.spec.affinity.nodeAffinity.preferredDuringScheduling... (Soft Preferences)
Mechanism via Scheduling Phases
         1. Filter Phase (Pod Hard Constraints):
The scheduler first executes the Filter phase. Here, the Pod's explicit required constraints are evaluated.
            * Effect: If a Pod explicitly requires karpenter.sh/capacity-type: on-demand, all Spot nodes are removed from consideration immediately. The PodSchedulingPreferences can only act on the nodes that survive this phase. This ensures that safety requirements (e.g., "GPU workloads must run on GPU nodes") always trump cost optimization defaults.
            2. NodesPostFilter Phase (PodSchedulingPreferences):
The new NodesPostFilter (Prioritization) phase runs next. It receives the nodes that survived the Filter phase.
               * Effect: It first resolves conflicts by selecting the single highest-priority PodSchedulingPreferences object matching the pod. It then applies the schedulingPriorities waterfall logic to the remaining nodes. It prunes the candidate list down to the highest-priority tier (e.g., filtering out On-Demand nodes if Spot nodes are valid and preferred). This effectively overrides any soft preferences that might come later, enforcing platform policy strictly.
               3. Score Phase (Pod Soft Preferences):
Finally, the Score phase runs on the nodes surviving the NodesPostFilter phase. Here, the Pod's preferred weights are calculated.
                  * Effect: Because the candidate list has already been pruned to a single priority tier (e.g., only Spot nodes remain), the Pod's soft preferences can only influence the decision within that tier (e.g., picking "Spot Zone A" vs "Spot Zone B"). They cannot force the scheduler to "jump" back to a lower-priority tier (e.g., On-Demand) even if the score would have been higher.
This hierarchy ensures that Hard Constraints override everything (Safety), Preferences override soft weights (Policy), and Soft Weights are used for tie-breaking (Optimization).
Alternatives Considered
Weight-Based Affinity (Soft Preferences)
The initial design considered using standard preferredDuringSchedulingIgnoredDuringExecution.
                  * Pros: Native to existing Kubernetes NodeAffinity logic; no new plugin logic required.
                  * Cons: "Soft" weights are easily overridden by other scoring factors (e.g., packing efficiency), leading to pods landing on expensive nodes even when cheap ones are available. Lacks the determinism required for strict financial governance.
Node Preference Label
A simpler alternative is to configure the scheduler to read a single, standardized node label (e.g., preference-rank) and sort by it.
                  * Pros: Extremely simple; no CRD required.
                  * Cons: Decentralized; hard to govern. Cannot define workload-specific policies (e.g., "Batch prefers Spot" vs. "Db prefers Reserved") because node labels are global.
Direct Pod Reference (Explicit Binding)
An alternative model avoids podSelector entirely in favor of an explicit reference in the Pod API.
                  * Mechanism: The PodSpec would include a new field (e.g., spec.schedulingPriorities) that references the PodSchedulingPreferences object by name (similar to runtimeClassName or priorityClassName).
                  * Pros:
                  * Eliminates Ambiguity: Resolves the risk of overlapping selectors. There is never a question of which policy applies.
                  * Performance: Removes the need for the scheduler to match labels against every active policy in the critical path.
                  * CRD Simplification: The podSelector field becomes optional or unnecessary in the PodSchedulingPreferences CRD.
                  * Cons:
                  * Tight Coupling: Binds application manifests to platform-specific resource names, potentially reducing portability across clusters with different naming conventions.
Implementation
To support the deterministic "Waterfall" logic defined in PodSchedulingPreferences, this proposal introduces a new extension point to the Kubernetes Scheduling Framework, tentatively named NodesPostFilterPlugin (based on the location after Filter), located immediately after the Filter phase and before the Score phase.
Waterfall Filtering
Standard Filter plugins remove nodes that are absolutely strictly infeasible (e.g., insufficient RAM). The new NodesPostFilterPlugin phase receives this list of feasible nodes and performs an additional "preference-based pruning" step:
                  1. Group & Check: The plugin evaluates the feasible nodes against the ordered list of schedulingPriorities.
                  2. Select Tier: It identifies the highest-priority tier (e.g., "Tier 1: Spot") that contains at least one feasible node.
                  3. Prune: It discards all nodes that do not belong to this tier.
                  4. Proceed: Only the nodes from the winning tier are passed to the Score phase.
This ensures that the Score phase (which uses weighted averages) never mixes options from different priority tiers. For example, a "Tier 2" node will never be chosen over a "Tier 1" node, regardless of how high its score is in other categories.
Golang Interface Definitions
The NodesPostFilterPlugin interface is modeled after the existing PreFilterPlugin but adapted to receive the filtered list of nodes.
// NodesPostFilteringPlugin is an interface that must be implemented by plugins that
// want to perform additional pruning of nodes after the main Filter phase.
// These plugins are called sequentially.
type NodesPostFilteringPlugin interface {
        Plugin
        // NodesPostFilter is called after the Filter phase.
        // filteredNodes contains the names of nodes that passed the Filter phase.
        // The plugin returns a PostFilterResult containing the subset of nodes that should remain candidates.
        // If the plugin does not want to filter any nodes, it should return a result with NodeNames: nil.
        NodesPostFilter(ctx context.Context, state *CycleState, pod *v1.Pod, filteredNodes []*NodeInfo) (*PostFilterResult, *Status)
}


// NodesPostFilterResult wraps needed info for scheduler framework to act upon
// NodesPostFilter phase.
type NodesPostFilterResult struct {
        // The set of node names that passed the NodesPostFilter phase.
        // If nil, it implies no additional filtering occurred (all input nodes from filteredNodes pass).
        NodeNames sets.Set[string]
}
Chain Logic: Strict Plugin Ordering
To handle multiple plugins implementing NodesPostFilteringPlugin (e.g., one for Cost Preferences and another for Locality Preferences), the Scheduling Framework enforces Strict Ordering. The plugins are chained such that the output of one plugin becomes the input of the next.
                  1. Initial Input: The Scheduling Framework calls the first NodesPostFilterPlugin with the full list of nodes that passed the standard Filter phase.
                  2. Sequential Pruning:
                  * Plugin 1: Receives [NodeA, NodeB, NodeC]. It applies its logic (e.g., "Prefer Spot") and returns NodeNames: {NodeA} (discarding NodeB and NodeC).
                  * Plugin 2: Receives only [NodeA]. It applies its logic (e.g., "Prefer Zone X") to this reduced set. If NodeA matches Zone X, it returns NodeNames: {NodeA}.
                  3. Final Output: The set of nodes returned by the last plugin in the chain is passed to the Score phase.
This architecture ensures that higher-priority policies (configured earlier in the chain) have the first opportunity to prune the candidate list, effectively acting as a "firewall" for lower-priority policies.
Future Extensions
Granular Preemption Control
Currently, the Kubernetes scheduler's preemption logic operates globally based on PriorityClass. It does not consider the type of capacity being contended for.
A future extension to PodSchedulingPreferences would allow users to define whether preemption is allowed for nodes matching specific selectors within a priority tier.
Proposed Configuration:
schedulingPriorities:
  - nodeSelector:
      matchLabels:
        capacity-type: spot
    # New Field:
    allowPreemption: false 
Use Case:
                  * Tier 1 (Spot): allowPreemption: false. "Try to find empty Spot nodes. Do not kill existing pods to make room on Spot, because the node itself is unstable."
                  * Tier 2 (On-Demand): allowPreemption: true. "If no Spot is available, use On-Demand. If necessary, preempt lower-priority workloads here to ensure this critical job runs."
This provides fine-grained control over the "disruption budget" of the cluster, ensuring that preemption only occurs when utilizing specific, stable, or high-cost infrastructure.
Complex Logic Support (CEL)
Future versions could support Common Expression Language (CEL) for more expressive podSelector logic or conditional priorities, allowing administrators to define rules like "Prefer Spot only if the job duration estimate is < 1 hour."


Capacity Provisioning API Design Proposal
[Public] Capacity Provisioning API Design Proposal
Authors: Paweł Kępka
Status: Draft
Last Update: Feb 6, 2026 
Summary
The Capacity Provisioning API is a proposed infrastructure-level integration API designed to explicitly provision compute resources (nodes) in a Kubernetes cluster.
Unlike the traditional Cluster Autoscaler, which reacts implicitly to "Pending" pods, this API enables a proactive, declarative model for capacity creation. It allows higher-level components - such as a Workload Scheduler - to request specific, atomic groups of nodes based on prior "what-if" simulations against the Potential Capacity API.
This API acts as the actuation layer in the Workload Aware-Scheduler architecture.
API Definition
Core Concept
The Capacity Provisioning API exposes a mechanism to request the creation of a specified quantity of compute resources that match a specific profile defined in the discovery layer. It provides a handle to track the lifecycle of this provisioning request from acceptance to fulfillment (Node readiness) or failure.
Positioning: This is an Infrastructure API, intended for consumption by controllers (e.g., Workload Scheduler, Job Orchestrators), not by end-user application developers.
Resource Model: CapacityRequest
The core resource is the CapacityRequest. It represents a transactional request to acquire capacity.
Key Attributes:
                  * Source Pool (capacityPoolRef): References a ClusterPotentialCapacityPool. This is the "menu item" family chosen by the scheduler.
                  * Node Requests (nodeRequests): A list of requests specifying the shapes and counts required. This allows a single atomic transaction to provision heterogeneous resources (e.g., 2 "medium" nodes and 1 "large" node) from the same pool.
                  * Atomicity: A flag indicating if the group must be provisioned "all-or-nothing" (Gang Scheduling support).
                  * Custom Metadata: Allows specifying additionalLabels and additionalTaints to be applied to the provisioned nodes. This is useful for binding nodes to specific jobs or tenants (e.g., job-id=123).
                  * Constraints/Overrides: Specific labels, taints, or parameter overrides (e.g., "zone=us-central1-a") to lock in a specific configuration chosen during the simulation phase.
API Structure
apiVersion: capacity.k8s.io/v1alpha1
kind: CapacityRequest
metadata:
  name: batch-job-gpu-request-xyz
spec:
  # The "Menu Item" family selected from the Potential Capacity API
  capacityPoolRef: "gpu-pool-v1"
  
  # List of required shapes and their counts
  # Allows requesting mixed shapes in one atomic operation
  nodeRequests:
    - nodeShape: "medium"
      count: 2
      # [Post-Alpha / Future Feature]
      # Placeholder Resolution for Topology Awareness
      placeholders:
        - count: 2
          values:
            topology.kubernetes.io/rack: "rack-group-A"
    - nodeShape: "large"
      count: 1
  
  # Custom metadata to be applied to the provisioned nodes
  # These are added on top of labels/taints defined in the Potential Capacity API
  additionalLabels:
    batch.example.com/job-id: "job-12345"
    cost-center: "research"
    
  additionalTaints:
    - key: "example.com/dedicated"
      value: "job-12345"
      effect: "NoSchedule"
  
  # Gang Scheduling Requirement
  atomicity: true 
  
  # Configuration locked in by the scheduler after simulation
  constraints:
    topology.kubernetes.io/zone: "us-central1-a"
    provisioning-model: "spot"


  # [Post-Alpha / Future Feature]
  # Fallback strategy for robustness
  fallbackStrategies:
    - nodeRequests:
        - nodeShape: "large"
          count: 3
    - nodeRequests:
        - nodeShape: "medium"
          count: 2
        - nodeShape: "large"
          count: 1


status:
  # Current state of the request (e.g., Pending, Provisioning, Fulfilled, Ready, Failed)
  state: Provisioning
  
  # Names of the nodes being provisioned (populated once provisioning starts),
  # grouped by their requested shape.
  nodes:
    - nodeShape: "medium"
      instanceNames:
        - "gke-cluster-1-gpu-pool-8302-8a9d"
        - "gke-cluster-1-gpu-pool-8302-2b3c"
    - nodeShape: "large"
      instanceNames:
        - "gke-cluster-1-gpu-pool-8302-9d8e"
Lifecycle States
The API manages the request through a state machine:
                  * Pending: Request accepted, validation passed.
                  * Provisioning: Interfacing with the underlying Cloud Provider API (e.g., GCE, AWS EC2).
                  * Fulfilled: VMs created; UpcomingNode objects may be visible.
                  * Ready: Kubernetes Nodes are registered and Ready.
                  * Failed: Provisioning failed (e.g., stockout).
Problem Statement & Motivation
This API addresses critical limitations in the current Kubernetes autoscaling architecture:
The "Blind Actuation" Problem
Current autoscalers (CA) combine the "decision to buy" with the "act of buying." An external scheduler cannot explicitly instruct the CA to "buy X nodes of type Y" without creating pending pods and hoping the CA interprets them correctly. This API makes the provisioning intent explicit.
Lack of Atomic/Gang Provisioning
For AI/ML workloads requiring Gang Scheduling, 4 out of 5 GPUs are useless. Current APIs provision nodes incrementally. The Capacity Provisioning API introduces Atomic requests, ensuring that either all required nodes are provisioned together, or none are, preventing resource deadlocks and partial cost waste.
Key Design Decisions
Decoupling Discovery
                  * Decision: Separate the Potential Capacity API (Read/Discovery) from the Capacity Provisioning API (Write/Actuation).
                  * Rationale:
                  * Scalability: Discovery is high-volume/read-heavy (schedulers running thousands of simulations). Provisioning is low-volume/write-heavy. Separating them allows independent scaling.
                  * Safety: Schedulers can safely run "what-if" scenarios against the read-only API without risking accidental cost or infrastructure changes.
Explicit Atomic Semantics
                  * Decision: The API supports an atomicity: true flag.
                  * Rationale: Enables native support for Gang Scheduling. The underlying provider ensures that the entire set of nodes is allocated in a single transaction (or saga), preventing the "partial allocation" problem common in distributed training jobs.
Shape Selection within Pools
                  * Decision: Use nodeRequests to select specific capacity variants within a generic ClusterPotentialCapacityPool.
                  * Rationale: Allows a single Pool resource to represent a family of compatible hardware (e.g., "N2 Standard Family"), reducing the number of CRDs the system must watch. The CapacityRequest then becomes the specific order for a size within that family, or even a mixed set of sizes.
Relation to Existing APIs
vs. GKE Compute Classes (CCC)
                  * Compute Classes define the User Intent (e.g., "I prefer Spot, but will accept On-Demand"). They act as templates or inputs.
                  * Capacity Provisioning API is the Actuation Mechanism. The Workload Scheduler reads a Compute Class, simulates the best outcome, and then issues a CapacityRequest to lock in that specific outcome.
                  * Relationship: ComputeClass -> Potential Capacity -> Scheduler Decision -> CapacityRequest.
vs. Karpenter APIs (NodeClaim)
                  * Karpenter NodeClaim: Represents a request for a single node. Karpenter acts as both the scheduler (decision maker) and the provisioner.
                  * Capacity Provisioning API: Represents a request for a group of nodes (Cluster). It decouples the scheduler logic from the provisioner logic.
                  * Convergence: In the proposed architecture, Karpenter could evolve to become the Cluster Capacity Provider. The CapacityRequest would effectively be a higher-level abstraction that Karpenter fulfills by creating multiple NodeClaims or by calling cloud APIs directly. This allows other schedulers (like Kueue or specialized batch schedulers) to drive Karpenter's provisioning engine without replacing it.
Integration Workflow: Kube-Scheduler & Two-Phase Binding
The integration of kube-scheduler (or the Workload Aware-Scheduler) with the Capacity Provisioning API relies on a Two-Phase Binding process. This allows the scheduler to make binding decisions before the physical infrastructure exists, and then seamlessly transition those decisions to the concrete resources once they are identified.
Phase 1: Preallocation to Virtual Nodes
During the scheduling cycle, if the kube-scheduler cannot find fit on existing nodes, it consults the Potential Capacity API.
                  1. Virtual Node Creation: The scheduler instantiates in-memory "Virtual Nodes" derived from ClusterPotentialCapacityPool templates and nodeShapes.
                  2. Simulation & Preallocation: The scheduler runs its standard filtering and scoring logic against these Virtual Nodes. When it finds a fit for a pod (or a gang of pods), it "preallocates" the pod to the Virtual Node in its internal cache (scheduler_cache).
                  3. Actuation: Instead of marking the pod Scheduled, the scheduler triggers a CapacityRequest for the required number of nodes, specifying both the capacityPoolRef and the selected nodeShape and sets the state.nominatedNodeName on the Pod object to the id of the created CapacityRequest.
Phase 2: Transition to Actual Node Names
Once the CapacityRequest is submitted, the scheduler watches the request object.
                  1. Name Discovery: The backend provisioner (e.g., GKE or Karpenter) begins provisioning and updates the CapacityRequest.status.nodes list. It groups the names of the nodes being created by their nodeShape (e.g., [{nodeShape: "medium", instanceNames: [...]}]).
                  2. Binding Switch:
                  * The scheduler observes the update to the CapacityRequest.
                  * It retrieves the specific pods that were preallocated to the "Virtual Node" associated with this request.
                  * It updates its internal state to map these pods to the actual node names provided in the status list (VirtualNode-UUID -> node-pool-x-abc for the corresponding shape).
                  3. Final Binding: The scheduler can now safely set the state.nominatedNodeName on the Pod object to the actual node name. This effectively prebinds the pod to the UpcomingNode (a node that is currently booting), ensuring that as soon as the Kubelet registers, the workload is ready to run.
This mechanism decouples the decision to schedule from the latency of provisioning, allowing the scheduler to maintain a consistent view of the world without blocking on cloud API calls.
Handling Stockouts & Updating Potential Capacity
A critical aspect of the two-API architecture is maintaining consistency between the discovery layer (ClusterPotentialCapacityPool) and the actuation layer (CapacityRequest). When a CapacityRequest fails due to a "stockout" (infrastructure unavailability), it is not enough to simply fail the request; the system must learn from this event to prevent "thrashing".
The Feedback Mechanism:
                  1. Stockout Detection: The backend provisioner attempts to fulfill a CapacityRequest and encounters a stockout error.
                  2. Potential Capacity Update: The Cluster Capacity Provider observes this failure. It identifies the capacityPoolRef and nodeShape associated with the failed request.
                  3. Request Failure: The CapacityRequest.status transitions to Failed, with a specific reason code (e.g., Reason: Stockout).
                  4. Circuit Breaking: The provider updates the status of the corresponding ClusterPotentialCapacityPool to reflect this unavailability (e.g., setting availabilitySignal to Low for that specific shape or pool).
                  5. Scheduler Adaptation: The Workload Scheduler, observing the updated pool, temporarily removes these "Virtual Nodes" from consideration.
                  6. Recovery: After the retryAfter period expires, the circuit breaker resets.
Alternative Designs Considered
Status Quo: Reactive Scaling (Pending Pods)
                  * Design: Scheduler marks pods "Pending"; Autoscaler scans queue and buys nodes.
                  * Verdict: Rejected. It forces the scheduler to make decisions without knowing if capacity exists. It lacks atomicity guarantees and prevents complex "what-if" optimization before job submission.
Combined Discovery & Provisioning API
                  * Design: A single API endpoint that checks availability and provisions immediately.
                  * Verdict: Rejected. It prevents lightweight simulation. The Scheduler often needs to evaluate a large number of different placement options before choosing one. A "commit-on-check" API makes this prohibitively expensive and complex.
Open Problems
Failed Scheduling of Pods on Provisioned Capacity
A significant open problem in this architecture is the risk of scheduling failures caused by discrepancies between the "Virtual Nodes" simulated via the Potential Capacity API and the physical nodes delivered by the Capacity Provisioning API. While the scheduler makes placement decisions based on templates and available shapes, the actual provisioned nodes may occasionally lack specific characteristics - such as precise topology labels, local storage configurations, or DRA device attributes - that were expected during the simulation phase. If the backend provisioner fulfills a CapacityRequest with nodes that do not perfectly align with these anticipated constraints, the pods remain in a "Pending" or "Nominated" state despite the successful creation of new infrastructure. This misalignment not only leads to resource waste but can also trigger "thrashing" where the system repeatedly attempts to provision capacity that fails to satisfy the scheduler's final binding requirements.
Phased Implementation of Cluster Autoscaling APIs
[Public] Phased Implementation of Workload Aware-Scheduler Cluster Autoscaling APIs
Authors: Paweł Kępka
Status: Draft
Last Update: Feb 6, 2026 
Summary
This document outlines a phased approach for implementing the three core APIs supporting the Workload Aware-Scheduler (WAS) architecture:
                  1. Potential Capacity API: Enables discovery of available but not yet provisioned capacity.
                  2. Capacity Provisioning API Enables explicit requests for new capacity.
                  3. Pod Scheduling Preferences API: Allows users to define policies for capacity selection.
The primary goal is to incrementally shift autoscaling responsibilities from a reactive model within the cluster autoscalers to a proactive model driven by the Kubernetes Scheduler, making it fully aware of potential capacity and workload requirements. This plan prioritizes early integration with the scheduler and a gradual expansion of features.
Goals of the Phased Approach
                  * Deliver a functional MVP quickly, focusing on scheduler integration.
                  * Incrementally build out API features and capabilities.
                  * Allow for early testing and feedback on the core scheduler-driven autoscaling loop.
                  * De-risk the project by tackling core integrations first.
                  * Maintain flexibility to adapt based on learnings from each phase.
Phased Implementation Plan
The implementation is divided into three main phases:
Phase 1: MVP - Scheduler-Driven Basic Provisioning
                  * Objective: Establish the fundamental interaction between the Scheduler and the Cluster Capacity Provider. Enable the scheduler to recognize potential capacity and trigger node provisioning for basic scenarios.
                  * API Deliverables:
                  * Potential Capacity API (ClusterPotentialCapacityPool):
                  * Define the CRD with a minimal set of fields: e.g., node shapes (name, basic resources), high-level constraints, and a template for node metadata.
                  * Capacity Provisioning API (CapacityRequest):
                  * Define the CRD with a minimal set of fields: e.g., reference to a ClusterPotentialCapacityPool, desired shape name, node count.
                  * Pod Scheduling Preferences API (PodSchedulingPreferences):
                  * Deferred to Phase 2 to minimize MVP scope.
                  * Scheduler Changes:
                  * Integrate with watch mechanisms for ClusterPotentialCapacityPool resources.
                  * Generate in-memory representations of potential nodes based on discovered pools.
                  * When pods are unschedulable, evaluate if they can fit on any potential nodes.
                  * If a fit is found, create CapacityRequest objects to trigger provisioning.
                  * Cluster Capacity Provider (Adapter):
                  * Implement logic to expose a limited set of existing autoscalable node groups/pools as ClusterPotentialCapacityPool resources.
                  * Implement logic to watch and actuate CapacityRequest objects by interfacing with the underlying infrastructure. This actuation should reuse the existing node provisioning logic within the Cluster Capacity Provider whenever possible, minimizing new code paths and leveraging tested mechanisms. For example, a CapacityRequest might translate to an internal call to scale up an existing managed node group.
Phase 2: Expanding Capabilities & Introducing Preferences
                  * Objective: Enhance the expressiveness of the APIs, support a broader range of capacity types and constraints, and introduce the policy layer through PodSchedulingPreferences.
                  * API Deliverables:
                  * Potential Capacity API:
                  * Introduce support for more detailed constraints.
                  * Include more status fields like for fine grained availability signals.
                  * Capacity Provisioning API:
                  * Allow for more specific overrides during requests, consistent with pool definitions.
                  * Pod Scheduling Preferences API:
                  * Define and implement the PodSchedulingPreferences CRD.
                  * Enable users to specify an ordered preference of ClusterPotentialCapacityPools for their pods (e.g., based on labels or pool characteristics).
                  * Scheduler Changes:
                  * Integrate with PodSchedulingPreferences to guide pool selection when multiple potential options exist for a pod.
                  * Enhance filtering logic to account for more complex constraints.
                  * Cluster Capacity Provider:
                  * Expand the types of capacity exposed via ClusterPotentialCapacityPool.
                  * Provide more dynamic or granular availability signals if possible.
                  * Implement translation logic to automatically generate PodSchedulingPreferences resources from existing configurations. This could involve watching resources like GKE Custom Compute Classes or Karpenter NodePools and creating corresponding PodSchedulingPreferences to align the scheduler's behavior with the provisioning intent defined in those resources.
Phase 3: Advanced Features & Ecosystem Integration
                  * Objective: Introduce advanced provisioning features, improve signal fidelity, and ensure robust integration with other Kubernetes features like topology management.
                  * API Deliverables:
                  * Potential Capacity API:
                  * Add support for concepts like supportsAtomicProvisioning.
                  * Enable representation of placeholders and node slices.
                  * Refine dynamic signal reporting (e.g., more accurate availability).
                  * Capacity Provisioning API:
                  * Support requests for atomic provisioning (all-or-nothing for multiple nodes).
                  * Handle resolution of placeholders and node slices.
                  * Scheduler Changes:
                  * Implement support for creating atomic CapacityRequests.
                  * Integrate with topology-aware scheduling plugins to leverage abstract constraints from potential capacity.
                  * Cluster Capacity Provider:
                  * Implement the backend logic for atomic provisioning requests.
                  * Translate abstract topological requests into concrete infrastructure placements.


[a]Thanks for putting this together! This is something we have thought about in the Karpenter community for a long time. We do all of this work simulating only to throw it away and have the scheduler simulate again.
2 total reactions
Derek Frank reacted with ➕ at 2026-02-16 19:49 PM
Jason Deal reacted with ➕ at 2026-03-02 19:00 PM
[b]In Karpenter, we reason about this by modeling cost in everything we do. 


This works for provisioning new capacity with different purchase types or even variants of different operating systems. You can even do things like factor in per-node licensing fees from various software vendors.


Critically, the kube scheduler is not cost aware in making its decisions -- it's purely scheduling constraints. Making both the scheduler and autoscaler cost-aware will lead to lower overall costs.
[c]We've seen this be particularly be problematic for TSCs, where users want to spread across all available domains (e.g. zones), but kube-scheduler is only aware of those already provisioned in the cluster. You can work around this particular example with minDomains, but it's not ideal.
[d]Do we want to declare "backwards compatibility w/ implicit node autoscalers" a goal? Here's one example:


// PodPending means the pod has been accepted by the system, but one or more of the containers
// has not been started. This includes time before being bound to a node, as well as time spent
// pulling images onto the host.
PodPending PodPhase = "Pending"


In our new, explicit provisioning model, how would an implicit autoscaler (e.g., legacy Cluster Autoscaler or Karpenter) know that a "Pending" pod with an empty NodeName value in its PodSpec is actively being handled by an in-progress explicit provisioning transaction?


Unfortunately the "Pending" pod trigger for legacy autoscalers is a sort of trick, as the original "Pending" semantics were not meant to indicate scheduling lifecycle status, but rather to indicate pre-terminal running state conditions (pulling images, exec bootstrapping, etc).


Do we want to update the set of PodStatus conditions to include side-effects of this new orchestration?
[e]Usually the implicit autoscalers wait for the scheduler to set PodScheduled condition on the pods with status equal to false and reason equal to Unschedulable before they start acting on pods giving the scheduler the opportunity to schedule pods on the existing nodes first. I believe that this should also work with the proposed model with scheduler simply not setting Unschedulable condition on the pod.
[f]FWIW, even today Cluster Autoscaler has a list of scheduler names that it "bypasses" by triggering autoscaling even pod pods without unschedulable condition. This improves performance at scale since scheduler qps is rate limited, but may not be backwards compatible with this approach.
[g]If we have this, I don't think we need the other two. Looking at the proposed implementation for the preferences API, the scheduler doesn't actually need to know what capacity could exist, because the preferences API tells the scheduler exactly that. For a spot versus OD example, if the preferences API specifies that this workload prefers spot instances, the scheduler only needs to answer the question 'do any of the existing nodes meet that preference?' when first attempting scheduling. It also needs to know when provisioning fails and thus it should fall back, but could that be captured more simply by a status on the pod? The autoscaler would be the responsible for informing the scheduler when it can't meet a given preference. I think that simplifies the architecture greatly, but I might need to write up my thoughts more concretely.
[h]I don't know why this doesn't have my profile attached, this is my comment
[i]Yes, I think I see how this could work but I believe that there are at least two major problems with this approach. One is the very poor efficiency of this solution even with just one scheduling fallback - moving to the next rule would involve going once through scheduler and once though autoscaler with at least two status updates to the pod. The other problem is the fact that autoscaler still needs to handle pods scheduling simulation on its side.
[j]One of the key differences in Karpenter and CAS is the simulation. CAS explores a space of templates, Karpenter uses a constraint solving approach (i.e. set intersection) with a much higher dimensionality.


The scale of capacity diversity is a major concern to get right here.
1 total reaction
Jason Deal reacted with ➕ at 2026-03-02 19:25 PM
[k]When designing https://github.com/kubernetes-sigs/karpenter/blob/main/designs/node-overlay.md, we strongly considered adding an "InstanceType" CRD that would serve as this role. An installation could include a set of instance types that would be discovered by the autoscaler. 


We had a number of concerns about the number of instance types, and the complexity of getting these configurations right, so we went with a discovery + selector approach of NodeOverlay, but this API is intentionally left Alpha. 


I am super excited to revisit this question with the broader community.
[l]I wonder if this needs to be an in-cluster API. Would a plugin model simplify the contract? kube-scheduler could define an in-memory model for 'potential capacity', and then various cp\autoscalers would provide plugins that can translate nodepools\compute classes into potential capacity in memory. This makes the potential capacity more opaque to end users, but might simplify the implementation
[m]Theres going to be some very interesting discussions about how this fits with existing scheduler constructs!
[n]If you're not familiar, under the hood, Karpenter's NodeClaim (https://karpenter.sh/docs/concepts/nodeclaims/) concept fulfills exactly this purpose. Some customers choose to create these directly (i.e. batch workload schedulers creates X node claims and then schedules work)
[o]Is this better expressed as inefficient cost optimization? bin packing is one measure of cost, but only a proxy for it
[p]As suggested I have updated the description from bin-packing to cost optimization. 
I have also extended the proposal with a description of the cost optimization mechanism for selecting potential nodes in the scheduler - see "Cost-Aware Scheduling on Potential Nodes" section below.
[q]What would be an example of this? If the scheduler is able to provision the node to existing capacity in the cluster, current autoscalers wouldn't react to the pod in the first place since it wouldn't be marked as unschedulable. I see this causing issues for things like TSC skew or anti-affinity, but I'm not sure how it would result in inefficient bin-packing.
[r]Why is this separate from user properties? Do we need to differentiate between pool wide properties that are defined by the system and pool wide properties that are defined by users?
[s]I assume this API will be written mostly by autoscalers, in which case we don't need to separate out the two
[t]The main distinction between capacity pool properties and user properties is that the capacity pool properties would be normally defined either by system administrator or by the capacity provisioning component (Cluster Autoscaler/Karpenter) itself and those would have a fixed values allowing scheduler to identify the capacity pools which can/should be used by a given pod/workload. User properties are arbitrary node properties which can be defined by workload developer and which can be added to nodes within a given capacity pool when provisioning nodes.
[u]I understand, i'm wondering why the scheduler needs to know which are which
[v]We encode this in Karpenter by reusing the `Exists` - I think that's a pretty elegant solution to saying "I know there will be a label with this key, but I don't know the value".
[w]The idea here is to be able to distinguish between a case (a) where user or some component can request any specific value for a given label from the situation (b) that it is a responsibility of a scheduler to generate any value for a given label in a way which would fulfill the scheduling constraints for a given workload.
So basically the difference is between (a) workload having a node selector for topology.kubernetes.io/rack=rack-1 from the situation (b) where workload has a pod anti affinity with topology.kubernetes.io/rack label used a topology key.
In (a) scheduler simply copies the value provided via the node selector. In (b) scheduler may need to generate some new dummy unique value for the label for the scheduling constraints to be fulfilled. For me simply stating that any value is allowed for a given label using Exists does not provide a way to distinguish between (a) and (b).
[x]Wouldn't we be able to infer daemonset overhead at runtime? It's not clear how we could statically define this as part of the pool definition.
2 total reactions
Brandon Wagner reacted with ➕ at 2026-02-26 20:51 PM
Daniel Kłobuszewski reacted with ➕ at 2026-03-17 13:17 PM
[y]Yes, it is true that we should be able to  calculate this at runtime and scheduler should be able to this by its own but at the same time doing this would require running a separate pod scheduling simulation loop iterating through DaemonSets one by one which does not fit that well into the current pod/workload scheduling loop in  kube-scheduler. Because of this at least initially I would like to avoid adding this logic to kube-scheduler.
This is discussed in more details in the "DaemonSet Simulation" section below.
[z]The availability signal at this level isn't sufficient since it would be dependent on the zone and provisioning model (spot vs on-demand). I think limits would have the same problem.


One option is to expand the constraints and provide a node shape status for each. But every constraint may not have to do with capacity or limits.
[aa]I see this mentioned below in "Stockout Signal Granularity for Fungible Constraints"
[ab]Based on feedback on the original proposal I have updated it slightly and split the original ClusterPotentialCapacityPool resource into two separate ones -ClusterPotentialCapacityPool and ClusterPotentialInstanceType resources. Introduction of ClusterPotentialInstanceType should provide a way to give scheduler a clear and fine grained information on availability of nodes on the zone and provisioning model level.
[ac]In a world in which the scheduler is provisioning nodes, do autoscalers need to support this behavior or could it remain a scheduler construct?
[ad]The idea here is that the underlining compute platform may or may not support atomic/all-or-nothing provisioning of multiple nodes and this information can influence how the provisioning of nodes will be orchestrated on the scheduler side.
[ae]Shouldn't the scheduler just handle it regardless? Even if all compute platforms promise atomic provisioning, nodes can still fail to join the cluster after they've been provisioned. If we're making the scheduler the provisioner, shouldn't it handle provisioning failures?
[af]I think we can use the karpenter requirements construct here, its proven to be elegant: https://karpenter.sh/docs/concepts/nodepools/#spectemplatespecrequirements
[ag]The idea here is to allow administrator or some autoscaling component to specify what kind of  labels and taints are allowed on provisioned nodes apart from the ones which are directly listed as part of constraints section listed below. So for instance on a given cluster it may be allowed to request/add arbitrary node levels but only with with some prefix. The same concept could also be represented in the constraints section but it would require using something like a regex instead of a single value for key. I have decided to keep this separate for clarity but if this would be simpler we can add this option to constraints section below.
[ah]I have scale concerns about this api, but I expect we'll iterate on that
[ai]I agree and I had similar concerns. Because of this I was considering for those shapes to be represented as separate objects. Apart from addressing scalability concerns this would also allow sharing them between different Capacity Pools. I decided not to follow this path because I didn't want to make the proposal even more complicated and I see some opportunities to group similar shapes together for instance by using the minimum allocatable resources numbers for sharpes which are very similar. But I would like to hear your thoughts on this.
[aj]Just to note 1 Karpenter NodePool would map to multiple CPCPs as defined here.
[ak]Can you provide more details on your thinking here?
[al]This could be problematic given my previous comment on 1 Karpenter NodePool mapping to multiple CPCPs. Particularly in the case of Spot requests where the request to the cloud provider (at least in AWS's case) should be a flexible request of many instance types.  There may be a configuration that the scheduler could use to determine how many to look at to bound it. In Karpenter we use a minValues on the NodePool requirements so that you can try to require a minimum flexibility in the bin-packing solution (e.g. I want at least 3 instance families and 2 generations).  


https://karpenter.sh/preview/concepts/nodepools/#:~:text=%23%20minValues%20here%20enforces%20the%20scheduler%20to%20consider%20at%20least%20that%20number%20of%20unique%20instance%2Dcategory%20to%20schedule%20the%20pods.
[am]So if I understand it correctly instead of scheduler stopping at first valid priority and selecting only one best option in order to provide better integration and obtainability for AWS Spots we would like for the scheduler to generate at least a few different options for provisioning.


If this is correct then the Capacity Provisioning API proposal already has an option for this in the form of fallbackStrategies. What may be problematic in the current proposal is the fact that right now Capacity Provisioning API assumes that each CapacityRequest touches only a single Capacity Pool but for defining fallbacks we may be able to change this assumption.


This brings us to the question how those different options could be generated. On the scheduler side right now we are implementing a support for topology aware scheduling which will extend the scheduler to be able to consider multiple scheduling options for a group of pods at once. The standard implementation will be selecting a single best option from those but using this mechanism we could generate multiple options.
[an]I have removed the original part of the proposal to have priority on the level of ClusterPotentialCapacityPool but as this comment is still relevant I have moved it next to the other comment.
[ao]Will we only allow a provisioning request to consist of a single template? I'd be a bit concerned with that limitation, particularly when considering the spot market. In Karpenter (AWS provider), we defer spot instance type selection to the EC2 CreateFleet API which has a price-capacity optimized selection mechanism. To take advantage of this property, we send as many compatible instance types to CreateFleet as possible, and these instance types may not share the exact same constraints. Limiting the total instance types the cluster capacity provider can select from for a given node may result in suboptimal capacity pool placements and lead to higher interruption rates.
[ap]How would you specify this in the API? Seems like you'd need some constraint on a placeholder that only 10 can be bucketed.
[aq]I have tried to rephrase the sentence to make it clearer what is the actual promise here. In general the placeholders should provide an API to request some number of nodes in specific buckets like racks or blocks without pointing to actual buckets. More details on how those would be used for actual provisioning of nodes can be found in the "Capacity Provisioning API Design Proposal" tab.
[ar]I haven't gotten to the Capacity Provisioning API doc yet, but I was thinking of specific cases (in AWS) where a Placement Group resource can be made which guarantees some bucketing locality that has limits on how many nodes can be placed in a group (generally for high performance networking). I think it would be useful if you could tell the scheduler how big those buckets are.
[as]Another AWS example is capacity reservations, in Karpenter we distinguish reserved capacity from on-demand capacity since users want to prioritize pre-paid capacity. These capacity reservations have fixed sizes at a point in time, but those sizes can fluctuate (e.g. if the user edits the reservation or if it's utilized out-of-band). I don't think this is an AWS specific concept either, afaik all major cloudproviders have a similar concept.
[at]When it comes to the size limits for capacity reservations I believe that those could be addressed with limits for capacity pool which would support providing additional constraints per limit.
[au]For reference, this is the current representation of this type of API within Karpenter. The cloudprovider implementation returns a set of InstanceTypes on a per-NodePool basis. Each NodeClaim (the equivalent of the JIT Node) starts out as a superposition of those instance types and is constrained as pods are scheduled.


https://github.com/kubernetes-sigs/karpenter/blob/35cafa54792e1016fc292d0b942c346205754fb8/pkg/cloudprovider/types.go#L102-L121
[av]This is Karpenter's approach
[aw]Cluster Autoscaler has an existing logic for this as well using similar approach. Because of this when it comes to the initial implementation for the integration with scheduler I am proposing to expose the result of this logic and not to reimplement it in the scheduler.
[ax]This is how we're approaching it in Karpenter. This RFC describes a user-facing way to describe these virtual resource slices via the NodeOverlay CRD, but it mirrors the internal API we're going forward with. There are some additional challenges that I laid out in that doc though (e.g. how do your reason about inter-resource slice constraints when the values for attributes are unknown). 


https://github.com/kubernetes-sigs/karpenter/pull/2559
[ay]I believe that there is a similar plan on the Cluster Autoscaler side for this.
[az]This could be moved to a pod pointing to the preferences.
[ba]I've heard of some users wanting to try really hard to get cheap capacity within a time bound. A timeout may be interesting to add between the tiers.  Something like try Spot for 30 minutes before falling back to OD.
[bb]With this model, is there a way to conditionally fall back to on-demand? Specifically, I'm thinking of a scenario where a pod could be scheduled to both on-demand and spot nodes, but the compatible / available on-demand offerings are actually cheaper than the spot offerings. This does occur due to the volatility of spot markets. In that case, you would want to provision an on-demand instance over a spot instance. I don't think this model is expressive enough for that type of behavior.