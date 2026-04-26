# Agent Reconciliation Loop

This document describes the reconciliation logic for the `Agent` custom resource in kagent.

## Overview

The agent controller watches `Agent` CRs and reconciles them against the desired state. The reconciler is responsible for:

1. Creating/updating the underlying `AutoGen` agent configuration
2. Syncing tool bindings from `Tool` CRs
3. Managing model provider references
4. Updating status conditions

## Reconciliation Flow

```
Agent CR created/updated
        │
        ▼
  Fetch Agent CR
        │
        ├─── Not found → clean up finalizer, return
        │
        ▼
  Add finalizer if missing
        │
        ▼
  Resolve ModelConfig reference
        │
        ├─── Not found → set Degraded condition, requeue
        │
        ▼
  Resolve Tool references
        │
        ├─── Tool missing → set Degraded condition, requeue
        │
        ▼
  Build AutoGen agent spec
        │
        ▼
  Upsert agent via kagent API
        │
        ├─── Error → set Degraded condition, requeue with backoff
        │
        ▼
  Update Agent status (Ready condition)
        │
        ▼
        Done
```

## Status Conditions

The reconciler manages the following conditions on the `Agent` CR:

| Condition | Reason | Description |
|-----------|--------|-------------|
| `Ready` | `ReconcileSuccess` | Agent is fully reconciled and operational |
| `Ready` | `ReconcileError` | Reconciliation failed; see message for details |
| `Degraded` | `ModelConfigNotFound` | Referenced ModelConfig CR does not exist |
| `Degraded` | `ToolNotFound` | One or more referenced Tool CRs are missing |
| `Degraded` | `APIError` | kagent API returned an error during upsert |

## Finalizer

The finalizer `kagent.io/agent-finalizer` is added on first reconcile. During deletion:

1. The agent is removed from the kagent API
2. The finalizer is removed, allowing the CR to be garbage collected

If the API call to delete fails, the finalizer is **not** removed and the object requeues.

## Tool Binding Resolution

Tools referenced in `spec.tools` are resolved by name within the same namespace:

```go
// Example: resolving tool refs
for _, toolRef := range agent.Spec.Tools {
    tool := &kagentv1.Tool{}
    if err := r.Get(ctx, types.NamespacedName{
        Name:      toolRef.Name,
        Namespace: req.Namespace,
    }, tool); err != nil {
        if apierrors.IsNotFound(err) {
            meta.SetStatusCondition(&agent.Status.Conditions, metav1.Condition{
                Type:    "Degraded",
                Status:  metav1.ConditionTrue,
                Reason:  "ToolNotFound",
                Message: fmt.Sprintf("Tool %q not found", toolRef.Name),
            })
            return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
        }
        return ctrl.Result{}, err
    }
    resolvedTools = append(resolvedTools, tool)
}
```

## ModelConfig Resolution

The `spec.modelConfig` field is a reference to a `ModelConfig` CR in the **same namespace**. Cross-namespace references are not supported.

If the referenced `ModelConfig` is deleted while an `Agent` is using it, the `Agent` will enter `Degraded` state and requeue every 30 seconds until the `ModelConfig` is restored or the `Agent` spec is updated.

## Requeue Behavior

| Scenario | Requeue Delay |
|----------|---------------|
| Transient API error | Exponential backoff (max 5 min) |
| Missing dependency (Tool/ModelConfig) | 30 seconds |
| Successful reconcile | No requeue (watch-driven) |
| Deletion in progress | 10 seconds if API delete fails |

## Common Failure Patterns

### Agent stuck in Degraded

```bash
# Check conditions
kubectl get agent <name> -n <namespace> -o jsonpath='{.status.conditions}' | jq .

# Common causes:
# 1. ModelConfig deleted or renamed
kubectl get modelconfig -n <namespace>

# 2. Tool CR missing
kubectl get tool -n <namespace>

# 3. kagent API unreachable
kubectl logs -n kagent-system deploy/kagent-controller-manager | grep "APIError"
```

### Finalizer blocking deletion

If the kagent API is permanently unavailable and you need to force-delete:

```bash
# Remove finalizer manually (use with caution)
kubectl patch agent <name> -n <namespace> \
  -p '{"metadata":{"finalizers":[]}}' \
  --type=merge
```

## Related

- [CRD Workflow Detailed](./crd-workflow-detailed.md)
- [E2E Debugging](./e2e-debugging.md)
- [Database Migrations](./database-migrations.md)
