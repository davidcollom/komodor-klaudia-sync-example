# Blueprint: AWS GitOps Event Platform

## Overview

This blueprint describes a Kubernetes platform running on Amazon EKS with GitOps-driven deployment and an external AWS messaging backbone.

The platform uses Amazon SNS and Amazon SQS, or EventBridge where appropriate, to decouple services and drive asynchronous workflows outside the cluster.

## Core Components

- EKS cluster across multiple availability zones
- Argo CD or Flux for declarative reconciliation
- ECR for container images
- SNS topics for fan-out notifications
- SQS queues for durable service consumption
- optionally EventBridge for cross-account or rules-based routing
- AWS Secrets Manager or Parameter Store for configuration
- ALB or NLB ingress pattern depending on workload requirements

## Intended Operating Model

1. platform and application manifests are stored in Git
2. CI produces immutable container images and validated manifests
3. GitOps applies the desired state to EKS
4. services emit events to SNS or EventBridge
5. consumers process events from SQS with retry and dead-letter handling

## Example Logical Flow

```text
Commit merged to main
  -> CI publishes image to ECR
  -> GitOps applies updated manifests to EKS
  -> workload processes user request
  -> domain event published to SNS
  -> SQS subscriptions distribute work to consumers
  -> failed events route to DLQ for investigation
```

## Security and Identity Expectations

- IAM Roles for Service Accounts for AWS API access
- limited controller permissions for GitOps tooling
- encryption at rest for queues, topics, and secrets
- explicit queue and topic policies for producer and consumer identities
- environment isolation using namespaces, accounts, or both

## Messaging Pattern

Suggested split:

- SNS for broad notification fan-out
- SQS for reliable point-to-point consumption
- EventBridge for integration with wider AWS estates

Operationally important queues:

- deployment notifications
- reconciliation outcomes
- business event processing
- dead-letter queues for failed consumers

## Failure Domains to Consider

- IRSA misconfiguration
- EKS add-on or ingress-controller failure
- queue backlog or poisoned messages
- EventBridge rules routing to the wrong targets
- incorrect environment overlays in Git

## Operational Checks

- GitOps sync status and target revision
- queue depth, age, and DLQ growth
- ALB health and target group registration
- IAM policy changes affecting producers or consumers
- rollout progress and pod disruption impact

## When to Use This Blueprint

Use this pattern when the wider platform is AWS-native and teams need resilient asynchronous integration without placing the event bus inside the cluster.
