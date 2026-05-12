# Blueprint: GCP GitOps Event Platform

## Overview

This blueprint describes a Kubernetes platform running on Google Kubernetes Engine with GitOps as the primary deployment model.

The platform integrates with Google Cloud Pub/Sub as an external event backbone for asynchronous processing, audit events, and cross-service workflows.

## Core Components

- GKE regional cluster for application workloads
- Argo CD or Flux for GitOps reconciliation
- Google Cloud Pub/Sub for event ingestion and fan-out
- Cloud SQL or managed data stores for application persistence
- External Secrets or Secret Manager integration for runtime secrets
- Managed ingress with Cloud Load Balancing and certificate automation
- Centralised logging and metrics via Cloud Logging and Managed Prometheus

## Intended Operating Model

1. application teams submit changes through Git
2. CI validates manifests, tests, and container images
3. GitOps reconciles approved changes into the cluster
4. workloads publish and consume events through Pub/Sub
5. platform observability captures deployment, runtime, and event health signals

## Example Logical Flow

```text
Developer PR
  -> CI build and test
  -> container image pushed to registry
  -> manifest update merged to main
  -> GitOps controller reconciles GKE
  -> service emits domain events to Pub/Sub
  -> downstream processors consume and update external systems
```

## Networking and Security Expectations

- private cluster nodes where possible
- workload identity for service authentication to GCP APIs
- restricted egress for workloads that do not need internet access
- namespace isolation for teams or environments
- GitOps controller access scoped to approved repositories and paths

## Pub/Sub Usage Pattern

Pub/Sub is used outside the cluster for durable decoupling between services.

Typical topics:

- deployment lifecycle notifications
- audit and compliance events
- application domain events
- operational automation triggers

Typical subscriptions:

- in-cluster event consumers
- analytics pipelines
- incident enrichment services
- external workflow processors

## Failure Domains to Consider

- GitOps reconciliation lag or failed sync
- service account or workload identity breakage
- Pub/Sub subscription backlog growth
- ingress misconfiguration or certificate failure
- environment drift between overlays

## Operational Checks

- GitOps health and last applied revision
- Pub/Sub subscription age and unacked message count
- node pool health and autoscaling behaviour
- Secret Manager access and rotation status
- service-to-topic IAM bindings

## When to Use This Blueprint

Use this pattern when teams need managed Kubernetes with strong Git-driven delivery and an external, cloud-native event backbone for loosely coupled services.
