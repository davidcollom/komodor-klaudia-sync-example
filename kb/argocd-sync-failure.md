# Runbook: Argo CD Sync Failure

## Purpose

Use this runbook when Argo CD reports `Sync Failed`, `OutOfSync`, or repeated reconciliation errors.

## Common Symptoms

- application stuck in `Progressing`
- sync retries without convergence
- hooks failing during deployment
- manifests render locally but fail in cluster

## Immediate Checks

### 1. Inspect application state

```bash
argocd app get <app-name>
argocd app history <app-name>
argocd app logs <app-name>
```

Review:

- target revision
- health status
- operation state
- failing resource
- hook or wave order

### 2. Check rendered manifests

```bash
argocd app manifests <app-name>
```

Validate that the rendered output matches expectations for the target cluster.

### 3. Check cluster-side errors

```bash
kubectl get events -A --sort-by=.lastTimestamp
kubectl describe <kind> <name> -n <namespace>
```

Look for:

- immutable field errors
- CRD missing or outdated
- RBAC denied
- admission webhook rejection
- missing namespace or secret

## Frequent Root Causes

- CRDs not installed before custom resources
- hook jobs failing or timing out
- Helm values mismatch between environments
- repo-server plugin or Kustomize build failure
- sync wave ordering issues
- invalid image pull secret or workload identity change

## Resolution Steps

### 1. Fix ordering issues

Ensure CRDs, namespaces, secrets, and controllers are applied before dependent resources.

### 2. Resolve immutable changes

If a resource field cannot be updated in place, plan a controlled recreation through Git rather than forcing ad hoc cluster changes.

### 3. Repair hook failures

Inspect hook pod logs:

```bash
kubectl logs job/<hook-job> -n <namespace>
```

### 4. Re-run a targeted sync

```bash
argocd app sync <app-name>
```

Avoid blanket force-sync unless the failure mode is understood.

## Escalate When

- the failure affects shared platform controllers
- the sync would recreate stateful resources
- the repo-server cannot render manifests consistently
- the same application fails across multiple clusters

## Evidence to Capture

- Argo CD application status output
- failing resource and error text
- rendered manifest snippet
- relevant Kubernetes events
