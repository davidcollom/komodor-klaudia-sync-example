# Runbook: GitOps Drift Investigation

## Purpose

Use this runbook when the live cluster state no longer matches the desired state stored in Git.

Typical symptoms:

- Argo CD or Flux reports drift
- resources are repeatedly reconciled
- a manual change appears to have been made in the cluster
- pods or services differ from what is defined in the repository

## First Questions

1. Which application, namespace, and cluster are affected?
2. Is the drift limited to one resource kind or many?
3. Did the drift start after a merge, manual hotfix, or controller upgrade?
4. Is the change benign metadata noise or a functional difference?

## Immediate Checks

### 1. Compare Git revision and deployed revision

For Argo CD:

```bash
argocd app get <app-name>
```

For Flux:

```bash
flux get kustomizations -A
flux get helmreleases -A
```

Confirm the revision currently applied by the controller and compare it to the expected branch or tag.

### 2. Inspect the drifted objects

```bash
kubectl get <kind> <name> -n <namespace> -o yaml
```

Look for:

- image tag differences
- replica count changes
- manually added annotations
- webhook-injected fields
- mutated security context or affinity rules

### 3. Check whether the difference is controller-managed noise

Common examples:

- defaulted fields added by the API server
- cert-manager annotations
- service mesh sidecar injection
- HPA-managed replica counts
- admission-controller mutations

If the difference is expected noise, update the GitOps ignore rules rather than repeatedly fighting reconciliation.

## Likely Causes

- a manual `kubectl edit` or `kubectl patch`
- a controller mutating fields not ignored by GitOps
- a Helm chart rendering differently due to values drift
- an environment-specific overlay applied incorrectly
- a stale branch or incorrect Git ref configured in the GitOps controller

## Resolution Path

### Option A: Reconcile back to Git

If Git is correct and the live change is not intended:

```bash
argocd app sync <app-name>
```

or:

```bash
flux reconcile kustomization <name> -n <namespace> --with-source
```

### Option B: Preserve the live fix in Git

If an emergency change fixed production behaviour, capture it properly:

1. identify the exact live delta
2. create a PR with the same change
3. get it reviewed and merged
4. allow GitOps to converge back to managed state

### Option C: Add ignore rules

If the drift is caused by expected mutations, define explicit ignore behaviour in Argo CD or Flux configuration rather than accepting permanent out-of-sync status.

## Escalate When

- drift affects cluster-wide components
- repeated reconciliation causes restarts or outages
- security controls were changed manually
- the source of truth is unclear across multiple repos

## Evidence to Capture

- affected application and namespace
- Git revision and live revision
- diff of the key fields
- whether the change was manual, automated, or unknown
- PR or incident link
