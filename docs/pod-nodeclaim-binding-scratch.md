# Pod ↔ NodeClaim Binding Design

## Problem

When scheduler creates a NodeClaim for pending pods, there's a race:
- NodeClaim takes 1-3 minutes to become a Node
- During that time, new pods could claim the same capacity
- Scheduler needs to "reserve" capacity on the NodeClaim

## Proposed Solution: `pod.spec.nodeClaimName` + `Provisioning` Phase

### Pod Spec Change

```yaml
spec:
  nodeClaimName: pool-abc123  # New field, set by scheduler
  nodeName: ""                # Empty until Node registers
```

### New Pod Phase

```
Pending → Provisioning → Running
                      ↘ Failed/Succeeded
```

- `Pending`: No scheduling decision
- `Provisioning`: Bound to NodeClaim, waiting for Node to exist
- `Running`: Node exists, containers executing

### Pod Status

```yaml
status:
  phase: Provisioning
  conditions:
    - type: PodScheduled
      status: "True"
      reason: "ProvisioningOnNodeClaim"
      message: "Pod provisioning on NodeClaim pool-abc123"
```

## Lifecycle Flow

1. **Scheduler creates NodeClaim** for pending pods
2. **Scheduler sets `pod.spec.nodeClaimName`** → phase becomes `Provisioning`
3. **Cloud provider launches instance** → NodeClaim gets `status.nodeName`
4. **Scheduler/controller copies `nodeName` to pod** → normal binding
5. **Kubelet runs pod** → phase becomes `Running`

### Failure Handling

- **NodeClaim fails:** Scheduler clears `nodeClaimName` → phase returns to `Pending` → pod re-evaluated
- **NodeClaim deleted:** Same as failure
- **Pod deleted while Reserved:** Normal pod deletion, no special handling

## Kubelet Behavior

Kubelet ignores pods where:
- `nodeClaimName` is set AND
- `nodeName` is empty OR doesn't match this node

This is a minor kubelet change - just skip these pods in the sync loop.

## Benefits

- **Explicit reservation:** Capacity is reserved in the API, survives scheduler restart
- **Clear UX:** `kubectl get pods` shows `Provisioning` vs `Pending`
- **No races:** Other schedulers/pods can't claim provisioning capacity
- **Natural extension:** Fits existing pod lifecycle model

## Alternatives Considered

### Scheduling Gates (no core API change)
```yaml
spec:
  schedulingGates:
    - name: capacity.k8s.io/node-provisioning
```
- Works today, but less explicit
- Pod stays `Pending` which is confusing
- Gate removal is another operation to coordinate

### Annotation-based reservation
- Scheduler state can drift from API
- Doesn't survive scheduler restart cleanly

### NodeClaim.spec.reservedPods
- Requires updating NodeClaim on each pod decision
- Inverts the ownership model

## Open Questions

1. **Who promotes nodeClaimName → nodeName?** Scheduler or dedicated controller?
2. **Timeout:** How long can a pod stay `Provisioning` before giving up?
3. **Priority/Preemption:** Can a higher-priority pod preempt a `Provisioning` pod's spot on a NodeClaim?
