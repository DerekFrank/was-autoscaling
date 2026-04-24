# Project Gluon: Architecture Diagram — What Karpenter Becomes

Recreate this in draw.io. Two side-by-side diagrams: "Today" and "With Project Gluon."

## Today

```mermaid
graph TB
    subgraph "Today: Split-Brain Architecture"
        pods[Pending Pods] --> sched[kube-scheduler]
        sched -->|bind to existing node| nodes[Existing Nodes]
        sched -->|pod stays pending| autoscaler

        subgraph autoscaler["Karpenter (scheduler + cloud provider actuator + lifecycle manager)"]
            sim[Scheduling Simulation<br/><i>reimplements scheduler logic</i>]

            subgraph cp["AWS Cloud Provider"]
                sel[Instance Selection<br/><i>cost-aware, constraint-solving</i>]
                act[EC2 Actuation<br/><i>launch instance</i>]
            end

            subgraph lm["Lifecycle Management"]
                consolidation[Consolidation]
                drift[Drift Detection]
                disruption[Disruption Handling]
            end

            sim --> sel --> act
        end

        autoscaler -->|provision node| nodes
    end
```

## With Project Gluon

```mermaid
graph TB
    subgraph "With Project Gluon"
        pods[Pending Pods] --> sched

        subgraph sched["kube-scheduler (unified)"]
            filter[Filter & Score<br/><i>existing + potential capacity</i>]
            decide{Best option?}
            filter --> decide
            decide -->|existing node| bind[Bind to Node]
            decide -->|new node needed| nc[Create NodeClaim]
        end

        capacity[Potential Capacity API<br/><i>InstanceTypes + Offerings</i>] -->|watches| sched
        bind --> nodes[Existing Nodes]
        nc --> karpenter

        subgraph karpenter["Karpenter (cloud provider actuator + lifecycle manager)"]
            subgraph cp["AWS Cloud Provider"]
                sel[Instance Selection<br/><i>cost-aware, constraint-solving</i>]
                act[EC2 Actuation<br/><i>Spot, RIs, Placement Groups,<br/>EFA, Nested Virtualization,<br/>Pod Isolation</i>]
            end

            subgraph lm["Lifecycle Management"]
                consolidation[Consolidation]
                drift[Drift Detection]
                disruption[Disruption Handling]
            end

            sel --> act
        end

        karpenter -->|provision node| nodes
        karpenter -->|publish offerings| capacity
    end
```

## Key Differences to Highlight

- **Removed from Karpenter:** "Scheduling Simulation" box — this moves into kube-scheduler.
- **Added to kube-scheduler:** Awareness of potential capacity (not just existing nodes).
- **Karpenter core retains:** Lifecycle management (consolidation, drift detection, disruption handling). These are upstream Karpenter features, cloud-provider-agnostic.
- **AWS cloud provider layer retains:** Instance selection, EC2 actuation, and deep integration with EC2 features (Spot, Reserved Instances, Placement Groups, EFA, Nested Virtualization, Pod Isolation).
- **New connection:** Potential Capacity API sits between Karpenter (publishes offerings) and kube-scheduler (consumes offerings).
- **NodeClaim** is the handoff contract between scheduler and Karpenter.
