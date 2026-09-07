---
name: argocd-kubectl-sync
description: Trigger and verify Argo CD Application sync operations through the Kubernetes API using kubectl, without the argocd CLI or Argo CD API server. Use for full Application syncs and carefully authorized prune or force operations; do not use a refresh annotation as a substitute for deployment.
---

# Sync Argo CD Applications with kubectl

Use only `kubectl` against the cluster containing the Argo CD `Application` CR. Establish the kubectl context, Application namespace, and exact Application name before changing it. A sync mutates the Application's destination cluster through Argo CD; Kubernetes API access to the management cluster is therefore authority to request a deployment, not permission to infer the target or destructive options.

## Understand the mechanism

Request a sync by setting the Application's top-level `.operation.sync`. The application-controller consumes `.operation` and reports the operation under `.status.operationState`.

Do not confuse sync with refresh:

- `argocd.argoproj.io/refresh: normal` asks Argo CD to recalculate target/live state.
- `argocd.argoproj.io/refresh: hard` also invalidates manifest and target-cluster caches.
- Neither refresh value deploys resources. Use `.operation.sync` when the requested outcome is synchronization.

Do not decide that a newly requested sync succeeded merely because `.status.operationState.phase` already says `Succeeded`; that may describe an older operation. Attach a unique item in `.operation.info`, wait until that exact value appears in `.status.operationState.operation.info`, and only then evaluate its phase.

## Safe preflight

Before submitting an operation:

1. Confirm the `applications.argoproj.io` CRD exists and read the selected Application.
2. Refuse if `metadata.deletionTimestamp` is set.
3. Report `argocd.argoproj.io/skip-reconcile: "true"`; do not remove it unless asked.
4. Refuse to overwrite a non-empty `.operation`. Wait for it to complete, or ask before terminating/replacing it.
5. Inspect `.spec.syncPolicy.syncOptions` and relevant resource annotations for behavior such as Replace, Force, ServerSideApply, pruning confirmation, or deletion propagation.
6. Keep `prune` and `force` false unless explicitly requested. Prune can delete resources. Force can delete/recreate resources after apply conflicts and may cause an outage.

## Self-contained helper

Copy this function into a Bash session. It uses Bash, `date`, and `kubectl`; it does not use the Argo CD CLI, server, `jq`, or an external script. It requests a full hook-aware sync, correlates the exact operation, waits for a terminal phase, and prints sync and health state.

```bash
argocd_kubectl_sync() {
  if [[ $# -lt 1 ]]; then
    echo "usage: argocd_kubectl_sync APP_NAME -n NAMESPACE [--timeout SECONDS] [--prune] [--force] [-- KUBECTL_ARGS...]" >&2
    return 2
  fi

  local app=$1 namespace="" timeout=300 prune=false force=false
  local current_operation deleting skip_reconcile rv token patch
  local seen phase message sync_status health_status revision deadline
  local -a kubectl_args=() k
  shift

  while [[ $# -gt 0 ]]; do
    case "$1" in
      -n|--namespace)
        [[ $# -ge 2 ]] || { echo "error: namespace value required" >&2; return 2; }
        namespace=$2; shift 2
        ;;
      --timeout)
        [[ $# -ge 2 && $2 =~ ^[1-9][0-9]*$ ]] || { echo "error: timeout must be positive seconds" >&2; return 2; }
        timeout=$2; shift 2
        ;;
      --prune) prune=true; shift ;;
      --force) force=true; shift ;;
      --) shift; kubectl_args=("$@"); break ;;
      *) echo "error: unknown option: $1" >&2; return 2 ;;
    esac
  done

  [[ -n "$namespace" ]] || {
    echo "error: specify the Application namespace with -n; it is not safe to guess" >&2
    return 2
  }
  [[ "$app" != */* ]] || app=${app#*/}

  k=(kubectl "${kubectl_args[@]}" --namespace "$namespace")
  "${k[@]}" get application.argoproj.io "$app" >/dev/null || {
    echo "error: cannot read Application $namespace/$app" >&2
    return 1
  }

  deleting=$("${k[@]}" get application.argoproj.io "$app" -o 'jsonpath={.metadata.deletionTimestamp}' 2>/dev/null || true)
  [[ -z "$deleting" ]] || {
    echo "error: Application $namespace/$app is being deleted" >&2
    return 1
  }
  skip_reconcile=$("${k[@]}" get application.argoproj.io "$app" -o go-template='{{index .metadata.annotations "argocd.argoproj.io/skip-reconcile"}}' 2>/dev/null || true)
  [[ "$skip_reconcile" != "true" ]] || {
    echo "error: Application has argocd.argoproj.io/skip-reconcile=true; not removing it" >&2
    return 1
  }
  current_operation=$("${k[@]}" get application.argoproj.io "$app" -o 'jsonpath={.operation}' 2>/dev/null || true)
  [[ -z "$current_operation" ]] || {
    echo "error: Application already has a pending/running .operation; not overwriting it" >&2
    return 1
  }

  rv=$("${k[@]}" get application.argoproj.io "$app" -o 'jsonpath={.metadata.resourceVersion}') || return 1
  token="kubectl-$(date -u +%Y-%m-%dT%H:%M:%SZ)-$$-$RANDOM"
  patch="{\"metadata\":{\"resourceVersion\":\"$rv\"},\"operation\":{\"initiatedBy\":{\"username\":\"kubectl\"},\"info\":[{\"name\":\"request-id\",\"value\":\"$token\"}],\"sync\":{\"prune\":$prune,\"syncStrategy\":{\"hook\":{\"force\":$force}}}}}"

  "${k[@]}" patch application.argoproj.io "$app" --type merge --patch "$patch" >/dev/null || {
    echo "error: sync request was not submitted; re-read the Application before retrying" >&2
    return 1
  }
  echo "requested Application $namespace/$app request-id=$token prune=$prune force=$force"

  deadline=$(( $(date +%s) + timeout ))
  while (( $(date +%s) < deadline )); do
    seen=$("${k[@]}" get application.argoproj.io "$app" \
      -o 'jsonpath={.status.operationState.operation.info[?(@.name=="request-id")].value}' 2>/dev/null) || {
        echo "error: Application disappeared or became unreadable" >&2
        return 1
      }
    if [[ "$seen" == "$token" ]]; then
      phase=$("${k[@]}" get application.argoproj.io "$app" -o 'jsonpath={.status.operationState.phase}') || return 1
      case "$phase" in
        Succeeded|Failed|Error|Terminated) break ;;
      esac
    fi
    sleep 2
  done

  seen=$("${k[@]}" get application.argoproj.io "$app" -o 'jsonpath={.status.operationState.operation.info[?(@.name=="request-id")].value}' 2>/dev/null || true)
  phase=$("${k[@]}" get application.argoproj.io "$app" -o 'jsonpath={.status.operationState.phase}' 2>/dev/null || true)
  if [[ "$seen" != "$token" || ! "$phase" =~ ^(Succeeded|Failed|Error|Terminated)$ ]]; then
    echo "error: timed out after ${timeout}s waiting for this operation; observed request-id=${seen:-<none>} phase=${phase:-<none>}" >&2
    return 1
  fi

  message=$("${k[@]}" get application.argoproj.io "$app" -o 'jsonpath={.status.operationState.message}' 2>/dev/null || true)
  sync_status=$("${k[@]}" get application.argoproj.io "$app" -o 'jsonpath={.status.sync.status}' 2>/dev/null || true)
  health_status=$("${k[@]}" get application.argoproj.io "$app" -o 'jsonpath={.status.health.status}' 2>/dev/null || true)
  revision=$("${k[@]}" get application.argoproj.io "$app" -o 'jsonpath={.status.sync.revision}' 2>/dev/null || true)
  echo "phase=${phase:-<none>} sync=${sync_status:-<none>} health=${health_status:-<none>} revision=${revision:-<none>}"
  echo "message=${message:-<none>}"

  [[ "$phase" == "Succeeded" ]]
}
```

Examples:

```bash
argocd_kubectl_sync guestbook -n argocd
argocd_kubectl_sync guestbook -n argocd --timeout 600
argocd_kubectl_sync guestbook -n argocd --prune
argocd_kubectl_sync guestbook -n argocd -- --context production
```

The function returns success when the correlated Argo CD operation phase is `Succeeded`. Report application health separately: an operation can finish while health is still `Progressing`, and `Degraded` requires investigation even if the operation itself succeeded.

## Advanced operation fields

For selective sync, a pinned revision, or non-default sync options, construct and review an `operation` manifest using the installed `applications.argoproj.io` CRD schema before applying it. Relevant fields can include:

```yaml
operation:
  initiatedBy:
    username: kubectl
  info:
    - name: request-id
      value: UNIQUE_TOKEN
  sync:
    revision: OPTIONAL_REVISION
    prune: false
    resources:
      - group: apps
        kind: Deployment
        name: example
        namespace: example
    syncOptions:
      - Validate=true
    syncStrategy:
      hook:
        force: false
```

Do not invent resource selectors or revisions. With multi-source Applications, revision selection is more complex; inspect the installed CRD and Application sources instead of assuming the single-source shape. The hook strategy is the normal hook-aware synchronization path. An apply strategy can bypass hook behavior and should be chosen only when the user specifically needs it.

## Refresh-only workflow

When the user asks only to refresh Argo CD's comparison state, annotate the Application and wait for the controller to remove the annotation:

```bash
kubectl annotate application.argoproj.io/APP -n NAMESPACE --overwrite \
  argocd.argoproj.io/refresh=normal
kubectl wait application.argoproj.io/APP -n NAMESPACE \
  --for=jsonpath='{.metadata.annotations.argocd\.argoproj\.io/refresh}'='' \
  --timeout=2m
```

Use `hard` instead of `normal` only when cache invalidation is actually required. Do not manually set `argocd.argoproj.io/refresh-timestamp`; current Argo CD documents it as controller-managed.

## Failure evidence

On failure or timeout, do not repeatedly replace `.operation`. Capture:

```bash
kubectl get application.argoproj.io/APP -n NAMESPACE -o yaml
kubectl describe application.argoproj.io/APP -n NAMESPACE
kubectl get events -n NAMESPACE --field-selector involvedObject.name=APP --sort-by=.lastTimestamp
```

Report the context, Application namespace/name, correlated request ID, operation phase/message, sync status/revision, health status, and failed entries in `.status.operationState.syncResult.resources`. `Failed`, `Error`, and `Terminated` are unsuccessful. A timeout is unknown/incomplete, not success.

## Primary documentation

- Sync with kubectl: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-kubectl/
- Sync options and destructive behavior: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/
- Refresh annotations: https://argo-cd.readthedocs.io/en/latest/user-guide/annotations-and-labels/
- Application API source: https://github.com/argoproj/argo-cd/blob/master/pkg/apis/application/v1alpha1/types.go
