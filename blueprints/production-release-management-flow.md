# Blueprint: Production Release Management Flow

## Overview

This blueprint describes a GitOps-centred release process for promoting Kubernetes changes into production across cloud environments.

It is designed for teams operating on GCP, AWS, or Azure with cloud-native messaging services outside the cluster for release notifications, audit, and workflow orchestration.

## Release Objectives

- preserve Git as the single source of truth
- produce immutable build artefacts
- separate build, approval, and deployment concerns
- ensure production promotion is observable and reversible
- emit release events to an external messaging system for audit and automation

## High-Level Flow

```text
Developer PR
  -> CI validation and security checks
  -> image build and push
  -> manifest or chart update PR
  -> approval gate
  -> merge to production branch or environment path
  -> GitOps reconciliation into production
  -> post-deploy verification
  -> release event emitted to external bus
```

## Detailed Stages

### 1. Change Preparation

- code change merged after review
- image built once and tagged immutably
- SBOM, provenance, and vulnerability checks completed
- deployment manifests updated to reference the immutable artefact

### 2. Pre-Production Validation

- unit and integration tests pass
- policy checks pass
- staging environment reconciles successfully
- synthetic checks and smoke tests succeed

### 3. Production Approval

- production promotion PR created or environment gate opened
- change record linked to ticket or incident if required
- approvers verify blast radius, rollback plan, and release notes

### 4. Production Reconciliation

- GitOps controller detects the approved production change
- rollout happens progressively where supported
- health checks validate pod readiness, service reachability, and key dependencies

### 5. Post-Deploy Verification

- confirm deployment revision and image digest
- validate golden signals and business KPIs
- verify no unexpected message backlog on the external bus
- send release outcome event to audit and operations consumers

## External Messaging Role

The external message bus is not the delivery engine for manifests. It supports the operating model around releases.

Example use cases:

- notify downstream systems of successful production release
- trigger audit enrichment or CMDB updates
- fan out release metadata to analytics and incident tooling
- initiate controlled follow-on jobs outside Kubernetes

Example by cloud:

- GCP: Pub/Sub topic for release lifecycle events
- AWS: SNS topic with SQS subscribers for release consumers
- Azure: Service Bus topic for release notifications and workflow subscribers

## Rollback Model

Preferred rollback path:

1. revert the production Git change
2. merge the revert through the same controls
3. allow GitOps to reconcile back to the previous known-good state

Emergency rollback path:

- only use break-glass controls when business impact justifies bypassing normal GitOps flow
- record the manual action and follow immediately with a Git correction

## Evidence to Capture for Every Release

- commit SHA and image digest
- approver and approval timestamp
- GitOps sync result
- rollout duration
- post-deploy verification result
- emitted release event ID or correlation ID

## When to Use This Blueprint

Use this flow when production delivery must remain Git-centred, cloud-portable in principle, and integrated with external messaging for operational workflow and auditability.
