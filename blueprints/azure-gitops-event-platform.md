# Blueprint: Azure GitOps Event Platform

## Overview

This blueprint describes a Kubernetes platform running on Azure Kubernetes Service with GitOps-based delivery and an Azure messaging layer external to the cluster.

The platform uses Azure Service Bus for queues and topics, and may combine it with Event Grid for broader event distribution.

## Core Components

- AKS cluster with separate system and user node pools
- Flux or Argo CD for GitOps reconciliation
- Azure Container Registry for image storage
- Azure Service Bus for queue and topic messaging
- optionally Event Grid for platform event routing
- Key Vault integration for secrets
- Azure Monitor, Log Analytics, and Prometheus-compatible metrics
- Application Gateway or nginx ingress depending on the environment

## Intended Operating Model

1. changes are proposed and reviewed through Git
2. CI validates code, images, policies, and manifests
3. GitOps reconciles approved changes into AKS
4. workloads publish commands or events to Service Bus
5. consumers process messages with retry and dead-letter support

## Example Logical Flow

```text
Pull request approved
  -> CI builds and signs image
  -> manifest change merged
  -> GitOps reconciles AKS
  -> application publishes event to Service Bus topic
  -> subscribers process the event
  -> failures move to dead-letter queue for analysis
```

## Security and Platform Expectations

- managed identity for workload access to Azure resources
- Key Vault-backed secret management
- network policies for namespace isolation
- GitOps repository and path restrictions
- controlled outbound access for workloads that call Azure APIs

## Messaging Pattern

Use Service Bus when applications need ordered, durable, enterprise-style messaging semantics.

Typical uses:

- integration with external business systems
- long-running asynchronous workflows
- release notifications and deployment approvals
- command processing with retry and dead-letter analysis

## Failure Domains to Consider

- managed identity permission drift
- Service Bus queue or subscription backlog
- Key Vault access failures during rollout
- ingress or WAF policy regressions
- GitOps reconciliation failures after overlay changes

## Operational Checks

- GitOps status and recent commits applied
- Service Bus backlog and dead-letter counts
- Key Vault access and secret version alignment
- ingress health and backend pool status
- node pool capacity and scaling events

## When to Use This Blueprint

Use this pattern when AKS is the main application platform and messaging requirements are better served by a managed external bus than by an in-cluster broker.
