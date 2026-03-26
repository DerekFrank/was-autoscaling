# NodePool and NodeClaim API - Scratch Notes

**Date:** March 24, 2026
**Context:** User intent and actuation resources for WAS counter-proposal

---

## NodePool - User Intent Resource

This is the user-facing resource that expresses what kind of capacity they want. Similar to Karpenter's NodePool but focused on intent rather than actuation.

```yaml
apiVersion: capacity.k8s.io/v1
kind: NodePool
metadata:
  name: general-purpose-pool
spec:
  # Template for nodes created from this pool
  template:
    metadata:
      labels:
        team: platform
    spec:
      # Requirements filter which instance types are eligible
      requirements:
        - key: capacity.k8s.io/instance-category
          operator: In
          values: ["general-purpose"]
        - key: capacity.k8s.io/capacity-type
          operator: In
          values: ["on-demand", "spot"]
        - key: topology.kubernetes.io/zone
          operator: In
          values: ["us-west-2a", "us-west-2b"]
      
      # Taints applied to nodes from this pool
      taints: []
      
      # Reference to cloud-specific configuration
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1
        kind: EC2NodeClass
        name: default
  
  # Limits on capacity from this pool
  limits:
    cpu: "1000"
    memory: "1000Gi"
  
  # Disruption settings
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30s

status:
  # Current resource usage
  resources:
    cpu: "100"
    memory: "200Gi"
```

---

## NodeClaim (Actuation Request)

Created by kube-scheduler when it decides to provision a new node. The cloud provider controller watches these and actuates.

```yaml
apiVersion: capacity.k8s.io/v1
kind: NodeClaim
metadata:
  name: general-purpose-pool-abc123
  ownerReferences:
    - apiVersion: capacity.k8s.io/v1
      kind: NodePool
      name: general-purpose-pool
spec:
  # Requirements that the provisioned node must satisfy
  # Scheduler has already resolved these to a specific instance type + offering
  requirements:
    - key: node.kubernetes.io/instance-type
      operator: In
      values: ["m5.large"]
    - key: topology.kubernetes.io/zone
      operator: In
      values: ["us-west-2a"]
    - key: capacity.k8s.io/capacity-type
      operator: In
      values: ["spot"]
  
  # Resource requests this node needs to satisfy
  resources:
    requests:
      cpu: "1500m"
      memory: "3Gi"
  
  # Reference to cloud-specific configuration
  nodeClassRef:
    apiVersion: karpenter.k8s.aws/v1
    kind: EC2NodeClass
    name: default

status:
  # Populated by cloud provider controller after launch
  providerID: "aws:///us-west-2a/i-1234567890abcdef0"
  nodeName: "ip-10-0-1-123.us-west-2.compute.internal"
  
  conditions:
    - type: Launched
      status: "True"
    - type: Registered
      status: "True"
    - type: Ready
      status: "True"
```

---

## Scheduler Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        kube-scheduler                            │
│                                                                  │
│  1. Watch InstanceType, CapacityOverlay, NodePool               │
│                                                                  │
│  2. For pending pods:                                           │
│     a. Find eligible NodePools (by pod requirements)            │
│     b. For each NodePool:                                       │
│        - Filter InstanceTypes by pool requirements              │
│        - Apply matching CapacityOverlays                        │
│        - Compute allocatable = capacity - overhead              │
│        - Filter offerings by pool zone/capacity-type reqs       │
│        - Result: List of (InstanceType, Offering) tuples        │
│                                                                  │
│  3. Score and select best (InstanceType, Offering, NodePool)    │
│                                                                  │
│  4. Create NodeClaim with resolved requirements                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Cloud Provider Controller                      │
│                                                                  │
│  1. Watch NodeClaim                                             │
│                                                                  │
│  2. On new NodeClaim:                                           │
│     - Read NodeClassRef for cloud-specific config               │
│     - Launch instance matching requirements                      │
│     - Update NodeClaim status with providerID                   │
│                                                                  │
│  3. On launch failure (ICE):                                    │
│     - Update InstanceType.status.offerings[].available = false  │
│     - Increment offeringsGeneration                             │
│     - Scheduler will retry with different offering              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Scale Analysis

| Resource | Count | Avg Size | Total |
|----------|-------|----------|-------|
| NodePool | ~10-50 | ~1KB | ~50KB |
| NodeClaim | ~1000s | ~1KB | ~1MB |

---

## Open Questions

1. **Reserved Capacity:** How to track in-flight NodeClaims against reservation limits?

2. **NodeClaim Lifecycle:** How long do NodeClaims persist after node is ready? Do they get garbage collected?

3. **Relationship to Karpenter:** Is this a direct evolution of Karpenter's NodePool/NodeClaim or a parallel concept?

## Related Design Notes

- **Potential Capacity API:** See `potential-capacity-api-scratch.md` for InstanceType and CapacityOverlay
- **Gang Scheduling:** See `gang-scheduling-design-scratch.md` for NodeClaimGang design
- **Feedback Loop:** See `feedback-loop-design-scratch.md` for ICE handling
