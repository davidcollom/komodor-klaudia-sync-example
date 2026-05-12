# Runbook: Flux Reconciliation Failure

## Purpose

Use this runbook when Flux resources fail to reconcile, report not-ready conditions, or stop applying new commits.

## Immediate Checks

### 1. Inspect Flux objects

```bash
flux get sources git -A
flux get kustomizations -A
flux get helmreleases -A
```

Look for failed readiness, stale revisions, or long reconcile delays.

### 2. Describe the failing object

```bash
kubectl describe kustomization <name> -n <namespace>
kubectl describe helmrelease <name> -n <namespace>
```

Pay attention to conditions, last applied revision, and dependency status.

### 3. Inspect controller logs

```bash
kubectl logs deploy/source-controller -n flux-system
kubectl logs deploy/kustomize-controller -n flux-system
kubectl logs deploy/helm-controller -n flux-system
```

## Typical Root Causes

- Git source authentication failure
- invalid Kustomize path or overlay
- Helm chart fetch or values rendering failure
- dependency not ready
- decryption failure for sealed or encrypted secrets
- API rate limiting or webhook timeout

## Resolution Steps

### 1. Reconcile source first

```bash
flux reconcile source git <name> -n <namespace>
```

### 2. Reconcile workload after the source is healthy

```bash
flux reconcile kustomization <name> -n <namespace> --with-source
```

or:

```bash
flux reconcile helmrelease <name> -n <namespace>
```

### 3. Verify referenced paths and dependencies

Confirm:

- branch or tag exists
- manifest path is correct
- secret references are present
- dependent Kustomizations are healthy

## Escalate When

- the source-controller cannot reach Git providers
- multiple tenants are blocked by one shared controller issue
- secret decryption failures imply key rotation or data loss
- reconciliation failures persist after source refresh

## Evidence to Capture

- object conditions
- controller logs
- failing revision
- namespace and cluster details
