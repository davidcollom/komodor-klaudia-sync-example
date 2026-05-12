# Runbook: CrashLoopBackOff After Config Change

## Purpose

Use this runbook when workloads begin crashing after a GitOps-applied configuration change.

## Typical Symptoms

- pods restart repeatedly after a merge
- deployment rollout stalls
- readiness probes fail after config or secret updates
- one environment fails while others remain healthy

## Immediate Checks

### 1. Identify the triggering revision

For Argo CD:

```bash
argocd app history <app-name>
```

For Flux:

```bash
flux get kustomizations -A
```

Correlate the failing rollout with the merged PR or Git revision.

### 2. Inspect pod state and logs

```bash
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
```

Look for:

- missing env vars
- invalid config format
- failed secret mount
- dependency connection failure
- probe misconfiguration

### 3. Compare current and previous config

Review the merged diff for:

- ConfigMap changes
- Secret reference changes
- image tag updates
- probe thresholds
- feature flags

## Resolution Steps

### 1. Decide rollback versus forward fix

Rollback if the change is clearly bad and customer impact is ongoing.

Forward-fix if the cause is understood and a small, safe correction is available.

### 2. Apply the fix through Git

Do not hot-edit workloads in the cluster unless there is an approved break-glass process.

### 3. Reconcile and observe rollout

```bash
kubectl rollout status deploy/<deployment> -n <namespace>
```

### 4. Confirm dependency health

Verify external services, secrets, and service discovery before closing the incident.

## Escalate When

- stateful workloads risk data corruption
- the bad config reached multiple clusters
- rollback is blocked by schema or migration changes
- the failure path affects shared platform services

## Evidence to Capture

- offending PR or commit
- previous and current config snippet
- container logs
- rollout events
