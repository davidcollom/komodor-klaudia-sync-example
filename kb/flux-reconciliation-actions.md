---
name: flux-kubectl-reconcile
description: Trigger and verify fresh Flux CD reconciliations using kubectl annotations, including Kustomizations, HelmReleases, HelmCharts, Sources, image automation, and other Flux resources that implement the reconcile-request status contract. Use when the Flux CLI is unavailable or must not be used.
---

# Reconcile Flux resources with kubectl

Use `kubectl` only. First establish the current context, namespace, exact resource kind/name, and whether the user wants merely the selected object or its upstream source refreshed too. Reconciliation changes cluster state; do not infer a cluster, namespace, target, `--force`, or retry reset.

## Core contract

Set `reconcile.fluxcd.io/requestedAt` to a value different from `.status.lastHandledReconcileAt`. The value is an opaque token; a timestamp is conventional. The controller queues the object and copies the handled value to `.status.lastHandledReconcileAt` when it acts on the request.

For a single object, use the self-contained helper below:

The helper annotates, waits for acknowledgement of that exact token, then evaluates the fresh `Ready` condition. It also supports HelmRelease-only `--force` and `--reset`, which are exceptional recovery actions and require explicit user intent.

### Self-contained helper

Copy this function into a Bash session and call `flux_kubectl_reconcile RESOURCE/NAME [options]`. It requires only Bash, `date`, and `kubectl`—never the Flux CLI.

```bash
flux_kubectl_reconcile() {
  if [[ $# -lt 1 ]]; then
    echo "usage: flux_kubectl_reconcile RESOURCE/NAME [-n NAMESPACE] [--timeout SECONDS] [--force] [--reset] [-- KUBECTL_ARGS...]" >&2
    return 2
  fi

  local target=$1 namespace="" timeout=120 force=false reset=false
  local resource token deadline handled suspended conditions ready
  local force_handled reset_handled
  local -a kubectl_args=() k annotations
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
      --force) force=true; shift ;;
      --reset) reset=true; shift ;;
      --) shift; kubectl_args=("$@"); break ;;
      *) echo "error: unknown option: $1" >&2; return 2 ;;
    esac
  done

  resource=${target%%/*}
  [[ "$target" == */* && -n "${target#*/}" ]] || {
    echo "error: target must be RESOURCE/NAME" >&2
    return 2
  }
  if { $force || $reset; } && [[ ! "$resource" =~ ^(helmrelease|helmreleases|hr)(\.helm\.toolkit\.fluxcd\.io)?$ ]]; then
    echo "error: --force and --reset apply only to HelmRelease" >&2
    return 2
  fi

  k=(kubectl)
  [[ -n "$namespace" ]] && k+=(--namespace "$namespace")
  k+=("${kubectl_args[@]}")

  "${k[@]}" get "$target" >/dev/null || {
    echo "error: cannot read $target" >&2
    return 1
  }
  suspended=$("${k[@]}" get "$target" -o 'jsonpath={.spec.suspend}' 2>/dev/null || true)
  [[ "$suspended" != "true" ]] || {
    echo "error: $target is suspended; not changing spec.suspend" >&2
    return 1
  }

  token="$(date -u +%Y-%m-%dT%H:%M:%SZ)-$$-$RANDOM"
  annotations=("reconcile.fluxcd.io/requestedAt=$token")
  $force && annotations+=("reconcile.fluxcd.io/forceAt=$token")
  $reset && annotations+=("reconcile.fluxcd.io/resetAt=$token")

  "${k[@]}" annotate --field-manager=flux-client-side-apply --overwrite \
    "$target" "${annotations[@]}" >/dev/null || return 1
  echo "requested $target token=$token"

  deadline=$(( $(date +%s) + timeout ))
  while (( $(date +%s) < deadline )); do
    handled=$("${k[@]}" get "$target" -o 'jsonpath={.status.lastHandledReconcileAt}' 2>/dev/null) || {
      echo "error: $target disappeared or became unreadable" >&2
      return 1
    }
    if [[ "$handled" == "$token" ]]; then
      if $force; then
        force_handled=$("${k[@]}" get "$target" -o 'jsonpath={.status.lastHandledForceAt}' 2>/dev/null || true)
        [[ "$force_handled" == "$token" ]] || { sleep 2; continue; }
      fi
      if $reset; then
        reset_handled=$("${k[@]}" get "$target" -o 'jsonpath={.status.lastHandledResetAt}' 2>/dev/null || true)
        [[ "$reset_handled" == "$token" ]] || { sleep 2; continue; }
      fi
      break
    fi
    sleep 2
  done

  handled=$("${k[@]}" get "$target" -o 'jsonpath={.status.lastHandledReconcileAt}' 2>/dev/null || true)
  [[ "$handled" == "$token" ]] || {
    echo "error: timed out after ${timeout}s waiting for acknowledgement; lastHandledReconcileAt=${handled:-<empty>}" >&2
    "${k[@]}" get "$target" -o wide >&2 || true
    return 1
  }

  conditions=$("${k[@]}" get "$target" -o go-template='{{range .status.conditions}}{{printf "%s=%s reason=%s message=%s\n" .type .status .reason .message}}{{end}}')
  printf '%s' "$conditions"
  ready=$("${k[@]}" get "$target" -o go-template='{{range .status.conditions}}{{if eq .type "Ready"}}{{.status}}{{end}}{{end}}')
  if [[ "$ready" == "True" ]]; then
    echo "reconciled $target token=$token"
    return 0
  fi
  if [[ -z "$ready" ]]; then
    echo "handled $target token=$token; no Ready condition is exposed" >&2
  else
    echo "error: $target handled token=$token but Ready=$ready" >&2
  fi
  return 1
}
```

Examples:

```bash
flux_kubectl_reconcile kustomization/app -n flux-system --timeout 180
flux_kubectl_reconcile helmrelease/app -n apps --timeout 300
flux_kubectl_reconcile helmrelease/app -n apps --force --timeout 300
flux_kubectl_reconcile gitrepository/flux-system -n flux-system -- --context production
```

Do not use `kubectl wait --for=condition=ready` immediately after annotation as the only verification. An already-present `Ready=True` may satisfy it before the new request is processed.

If the helper cannot be used, follow the same invariant manually:

1. Generate one unique token and retain it.
2. Annotate with `kubectl annotate --field-manager=flux-client-side-apply --overwrite ... "reconcile.fluxcd.io/requestedAt=$TOKEN"`.
3. Poll until `.status.lastHandledReconcileAt` exactly equals `$TOKEN`.
4. Read the current `Ready`, `Reconciling`, and `Stalled` conditions and relevant revision/artifact fields. Report failure when fresh `Ready=False`, `Stalled=True`, acknowledgement times out, or the object disappears.

## Resource and dependency behavior

This contract is controller/API dependent, not universal to every Flux object. Before using it for an unfamiliar CRD, inspect the object/CRD for `.status.lastHandledReconcileAt` or confirm it in current Flux documentation. Common direct targets include Kustomization, HelmRelease, HelmChart, GitRepository, OCIRepository, Bucket, HelmRepository (non-OCI mode), ImageRepository, ImageUpdateAutomation, Receiver, and ArtifactGenerator.

- `ImagePolicy` is derived from `ImageRepository`; reconcile the referenced ImageRepository when a fresh scan/evaluation is needed.
- An OCI-mode HelmRepository is a data container and is not itself fetched; reconcile the HelmChart or OCIRepository that produces the artifact.
- A HelmRelease reconcile does not guarantee its upstream source was freshly fetched. If fresh upstream content is required, identify and reconcile the source first, then HelmChart (when independently managed), then HelmRelease. For a HelmRelease-generated HelmChart, inspect `.status.helmChart` and avoid guessing its generated name.
- A Kustomization reconcile uses the current artifact of `.spec.sourceRef`. Reconcile that Source first when the user means “fetch remote changes and apply them,” wait for its fresh result, then reconcile the Kustomization.
- Respect `.spec.suspend: true`; report it rather than silently resuming. Annotation is not a substitute for changing suspension.
- `HelmChart.spec.reconcileStrategy` can prevent a new artifact from being produced for GitRepository/Bucket changes when it is `ChartVersion`; `Revision` is needed if source revisions should drive chart artifacts. Do not mutate this policy unless asked.

Read [references/resource-notes.md](references/resource-notes.md) when resolving source chains, generated HelmCharts, condition semantics, or special Helm actions.

## Diagnostics and reporting

On failure, show concise evidence from the target rather than repeatedly retriggering it:

```bash
kubectl get RESOURCE/NAME -n NAMESPACE -o wide
kubectl describe RESOURCE/NAME -n NAMESPACE
kubectl get events -n NAMESPACE --field-selector involvedObject.name=NAME --sort-by=.lastTimestamp
```

Report the context, namespace, exact target, acknowledgement token, final Ready/Reconciling/Stalled reason and message, and artifact/revision fields when present. A successful request means both that the exact token was acknowledged and that the fresh state is Ready; distinguish this from “request accepted but reconciliation failed.”
