# Runbook: Ingress, DNS, and TLS Triage

## Purpose

Use this runbook when a Kubernetes service is deployed but users cannot reach it through the expected hostname.

## Symptoms

- 404 or 502 from ingress
- certificate not issued or expired
- DNS resolves to the wrong endpoint
- load balancer exists but traffic still fails

## Immediate Checks

### 1. Verify workload and service health

```bash
kubectl get deploy,po,svc -n <namespace>
kubectl describe svc <service-name> -n <namespace>
```

Make sure the service selects healthy pods and exposes the expected port.

### 2. Inspect ingress or gateway configuration

```bash
kubectl get ingress -n <namespace>
kubectl describe ingress <ingress-name> -n <namespace>
```

Check:

- hostnames
- backend service name and port
- ingress class
- TLS secret reference
- controller annotations

### 3. Validate DNS resolution

```bash
dig +short <hostname>
nslookup <hostname>
```

Compare the result with the external IP or hostname assigned to the ingress or load balancer.

### 4. Validate certificate state

```bash
kubectl get certificate,certificaterequest,challenge -A
kubectl describe certificate <name> -n <namespace>
```

## Common Causes

- wrong ingress class after controller migration
- service port mismatch
- DNS record not updated after load balancer recreation
- cert-manager issuer misconfiguration
- GitOps applied an outdated hostname or annotation

## Resolution Steps

1. fix the manifest in Git, not by editing the live ingress
2. ensure the service targetPort matches the application listener
3. confirm DNS points at the current ingress endpoint
4. validate cert-manager issuer and ACME challenge flow
5. reconcile the application through GitOps

## Escalate When

- certificates are failing cluster-wide
- shared ingress controller is unhealthy
- DNS is managed by another team and blocking recovery
- traffic failures affect production customer paths

## Evidence to Capture

- ingress manifest and events
- service endpoints
- DNS output
- certificate condition details
