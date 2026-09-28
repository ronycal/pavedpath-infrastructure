# PavedPath Architecture

## Overview

PavedPath is a GitOps-driven Internal Developer Platform designed to provide application teams with a secure, repeatable, observable, and auditable path from source code to production.

The platform separates:

- cloud infrastructure provisioning,
- application artifact production,
- Kubernetes application delivery.

The primary architectural principle is:

> Infrastructure provisioning, artifact production, and application deployment are separate concerns with explicit ownership, permissions, and delivery workflows.

PavedPath provides shared platform capabilities while allowing application teams to consume those capabilities through a supported paved path.

---

## Goals

PavedPath is designed to:

- reduce developer cognitive load,
- standardize application delivery,
- provide secure self-service workflows,
- create repeatable AWS infrastructure,
- make Git the source of truth for supported Kubernetes desired state,
- provide observable and auditable deployments,
- reduce direct developer interaction with Kubernetes and AWS,
- enforce clear infrastructure and application ownership boundaries,
- use temporary workload identities instead of long-lived cloud credentials,
- provide a foundation for platform automation and AI-assisted engineering workflows.

---

## Repository Architecture

PavedPath is divided into three repositories with explicit ownership boundaries.

```text
pavedpath-infrastructure
        |
        | provisions
        v
AWS Infrastructure
        |
        +---- VPC
        +---- IAM
        +---- ECR repositories
        +---- EKS
        +---- supporting AWS services


pavedpath-sample-api
        |
        | builds and validates
        v
Application Artifact
        |
        | publishes
        v
Amazon ECR


pavedpath-gitops
        |
        | declares Kubernetes desired state
        v
Argo CD
        |
        | reconciles
        v
Amazon EKS
```

Each repository owns a different part of the platform lifecycle.

### pavedpath-infrastructure

Owns foundational AWS infrastructure.

Responsibilities include:

- Terraform configuration,
- Terraform state configuration,
- AWS networking,
- Amazon EKS infrastructure,
- Amazon ECR repositories,
- IAM,
- encryption infrastructure,
- GitHub OIDC integration,
- supporting AWS infrastructure,
- infrastructure CI workflows.

This repository does not own normal application deployments.

### pavedpath-gitops

Owns supported Kubernetes desired state.

Responsibilities include:

- Argo CD configuration after bootstrap,
- Kubernetes platform services,
- supported Kubernetes policies,
- application and platform-service Namespaces,
- application deployment manifests,
- environment-specific deployment configuration,
- application image references.

This repository does not provision foundational AWS infrastructure.

### pavedpath-sample-api

Represents an application team consuming PavedPath.

Responsibilities include:

- application source code,
- application dependencies,
- tests,
- Dockerfile,
- application CI,
- application health endpoints,
- container image creation,
- application-specific validation.

This repository does not provision shared infrastructure and does not directly deploy normal workloads to Kubernetes.

---

## Infrastructure Control Plane

Infrastructure changes follow a reviewed infrastructure workflow.

```text
Platform Engineer
       |
       v
Infrastructure Branch
       |
       v
Pull Request
       |
       v
GitHub Actions
       |
       +---- Terraform Format
       |
       +---- Terraform Validate
       |
       +---- Security / Policy Checks
       |
       +---- Terraform Plan
       |
       v
Review / Approval
       |
       v
Merge
       |
       v
Approved Terraform Apply
       |
       v
AWS
       |
       +---- VPC
       +---- Networking
       +---- IAM
       +---- ECR repositories
       +---- EKS
       +---- Supporting Services
```

Terraform owns foundational AWS infrastructure.

Terraform does not own normal application workloads.

Infrastructure workflows authenticate to AWS through approved GitHub OIDC federation and IAM roles.

---

## Application CI Control Plane

Application CI validates source code and produces deployable artifacts.

```text
Developer
    |
    v
Application Branch
    |
    v
Application Pull Request
    |
    v
GitHub Actions
    |
    +---- Lint
    +---- Test
    +---- Security Validation
    +---- Build
    +---- Image Scan
    |
    v
Verified Container Image
    |
    v
Amazon ECR
```

Application CI produces artifacts.

Application CI does not directly deploy normal workloads to Kubernetes.

Application CI receives only the AWS permissions required for approved artifact operations.

---

## Application Delivery Control Plane

Application delivery follows a separate GitOps path.

```text
Verified Container Image
        |
        v
GitOps Change
        |
        v
pavedpath-gitops
        |
        v
Pull Request
        |
        +---- Manifest Validation
        +---- Policy Validation
        +---- Review
        |
        v
Merge
        |
        v
Git Desired State
        |
        v
Argo CD
        |
        v
Amazon EKS
```

Application deployment intent is represented in Git.

Argo CD acts as the Kubernetes delivery controller.

Application CI does not act as the Kubernetes deployment controller.

---

## End-to-End Application Flow

The supported application delivery path is:

```text
Developer
    |
    v
pavedpath-sample-api
    |
    v
Application CI
    |
    +---- Lint
    +---- Test
    +---- Scan
    +---- Build
    |
    v
Verified Immutable Image
    |
    v
Amazon ECR
    |
    v
GitOps Change
    |
    v
pavedpath-gitops
    |
    v
Review / Validation
    |
    v
Merge
    |
    v
Argo CD
    |
    v
Amazon EKS
```

The three repositories participate in the delivery process without sharing ownership of the same lifecycle.

---

## GitOps Model

Git represents the authoritative desired state of Kubernetes resources owned by the GitOps delivery plane.

The deployment model is:

```text
Git Desired State
        |
        v
     Argo CD
        |
        | compare / reconcile
        v
Kubernetes Actual State
```

Argo CD compares Git desired state with Kubernetes actual state.

Depending on the synchronization policy, Argo CD may report drift, reconcile approved differences automatically, or require synchronization approval.

Where appropriate, automated synchronization may include:

- pruning,
- self-healing,
- drift correction.

Synchronization behavior will be configured deliberately based on the lifecycle and risk of the managed resources.

Direct application deployment using commands such as:

```text
kubectl apply
helm upgrade
```

is not part of the normal deployment path.

Administrative use of Kubernetes tooling may be required for troubleshooting or approved break-glass operations, but those actions do not replace Git as the source of truth.

---

## Immutable Deployment State

PavedPath will prefer immutable application image references.

For example:

```text
repository@sha256:<digest>
```

Application CI produces and publishes the artifact.

The GitOps repository records which artifact should run.

This separates:

```text
Artifact Creation
      |
      v
Application CI

from

Deployment Intent
      |
      v
GitOps
```

A deployment therefore references a specific verified artifact rather than relying solely on a mutable image tag.

---

## Infrastructure and Application Boundary

The high-level ownership model is:

```text
+-------------------------------------------------------+
|              INFRASTRUCTURE CONTROL PLANE             |
|                                                       |
| pavedpath-infrastructure                              |
|          |                                            |
|          v                                            |
|    GitHub Actions                                     |
|          |                                            |
|          v                                            |
|      Terraform                                        |
|          |                                            |
|          v                                            |
|         AWS                                           |
|                                                       |
| VPC | IAM | ECR Repositories | EKS | AWS Services    |
+--------------------------+----------------------------+
                           |
                           | provides platform
                           | capabilities
                           v
+-------------------------------------------------------+
|              APPLICATION DELIVERY PLANE               |
|                                                       |
| pavedpath-sample-api                                  |
|          |                                            |
|          v                                            |
|    Application CI                                     |
|          |                                            |
|          v                                            |
| Verified Container Artifact                           |
|          |                                            |
|          v                                            |
|         ECR                                           |
|          |                                            |
|          v                                            |
|   pavedpath-gitops                                    |
|          |                                            |
|          v                                            |
|       Argo CD                                         |
|          |                                            |
|          v                                            |
|         EKS                                           |
+-------------------------------------------------------+
```

The infrastructure layer provides capabilities.

The application layer consumes those capabilities.

Consumption does not transfer ownership of the underlying infrastructure.

For example:

- Terraform owns the ECR repository.
- Application CI publishes images into the repository.
- Terraform owns EKS infrastructure.
- Argo CD reconciles supported Kubernetes resources onto EKS.

---

## Argo CD Bootstrap Boundary

Argo CD creates a bootstrap dependency because the GitOps controller must exist before it can reconcile GitOps-managed resources.

The initial sequence is:

```text
AWS Infrastructure
       |
       v
Amazon EKS
       |
       v
Initial Argo CD Bootstrap
       |
       v
Argo CD Operational
       |
       v
pavedpath-gitops
       |
       v
Ongoing Kubernetes Reconciliation
```

The bootstrap mechanism may establish the minimum resources required to make Argo CD operational.

Bootstrap ownership does not imply ongoing ownership.

After Argo CD becomes operational, normal application and supported platform-service delivery flows through GitOps.

---

## Authentication Model

Long-lived AWS access keys must not be the normal authentication mechanism for GitHub Actions.

GitHub Actions will authenticate to AWS using OpenID Connect federation.

The authentication flow is:

```text
GitHub Actions
       |
       | request OIDC token
       v
GitHub OIDC Provider
       |
       | identity assertion
       v
AWS IAM / STS
       |
       | AssumeRoleWithWebIdentity
       v
Temporary AWS Credentials
```

The expected OIDC audience for the standard AWS authentication flow is:

```text
sts.amazonaws.com
```

IAM trust policies determine which GitHub workload identities may assume a role.

IAM permission policies determine what an assumed role may do.

These are separate authorization controls.

---

## Workflow Permission Separation

Different workflows receive different AWS permissions.

Conceptually:

```text
Infrastructure Pull Request
        |
        v
Terraform Plan Authorization
        |
        +---- Read approved state
        +---- Inspect approved infrastructure
        +---- Produce plan


Approved Infrastructure Apply
        |
        v
Terraform Apply Authorization
        |
        +---- Modify approved infrastructure


Application CI
        |
        v
Application CI Role
        |
        +---- Authenticate to approved ECR
        +---- Push approved container artifacts
```

PavedPath will target separate authorization boundaries for Terraform planning and application.

Application CI does not receive permissions to administer shared AWS infrastructure.

---

## Desired Ownership Model

| Resource | Normal Owner |
|---|---|
| VPC | Terraform |
| Subnets | Terraform |
| Route configuration | Terraform |
| EKS infrastructure | Terraform |
| IAM foundations | Terraform |
| ECR repositories | Terraform |
| GitHub AWS OIDC integration | Terraform |
| Argo CD initial bootstrap | Infrastructure / bootstrap process |
| Argo CD ongoing configuration | GitOps |
| Kubernetes platform services | Argo CD / GitOps |
| Supported Kubernetes policies | Argo CD / GitOps |
| Application manifests | Argo CD / GitOps |
| Application source | Application repository |
| Application tests | Application repository |
| Container artifact | Application CI |
| Application deployment version | GitOps repository |

A resource should have one normal control-plane owner.

Multiple automation systems must not independently manage the same resource lifecycle.

---

## Security Principles

PavedPath follows these security principles:

1. No long-lived AWS credentials in GitHub as the normal authentication path.
2. GitHub Actions uses OIDC federation and temporary AWS credentials.
3. IAM trust relationships are scoped to approved workload identities.
4. IAM permissions follow least privilege.
5. Infrastructure changes occur through reviewed Git workflows.
6. Application deployments occur through reviewed GitOps changes.
7. Application CI cannot modify shared infrastructure.
8. Application CI does not require normal Kubernetes deployment credentials.
9. Secrets must not be stored in plaintext in Git.
10. Container images are scanned before promotion.
11. Infrastructure and application permissions remain separated.
12. Terraform and Argo CD must not compete for ownership of the same managed resource.

---

## Secrets Boundary

Application and platform secrets require a mechanism that preserves GitOps without storing plaintext secret values in Git.

The exact secrets-management implementation will be selected during the platform-services phase.

Regardless of implementation, the architecture requires that:

- plaintext production secrets are not committed to Git,
- access follows least privilege,
- secret retrieval is auditable where supported,
- application teams consume secrets through the supported platform mechanism.

---

## Observability Model

PavedPath will provide visibility across both platform and application delivery.

Relevant signals include:

```text
Infrastructure
    |
    +---- Terraform workflow status
    +---- AWS service health
    +---- EKS health

Delivery
    |
    +---- GitOps change history
    +---- Argo CD synchronization status
    +---- Argo CD health status

Application
    |
    +---- health endpoints
    +---- metrics
    +---- logs
    +---- Kubernetes events
```

The platform should allow engineers to correlate deployment intent, reconciliation state, and runtime behavior.

---

## Operational Principles

PavedPath will be designed so that:

- infrastructure changes are auditable,
- application deployments are auditable,
- deployment state can be reconstructed from Git,
- application versions can be rolled back through Git,
- infrastructure and application ownership are identifiable during incidents,
- platform services expose useful operational signals,
- application workloads expose health signals,
- platform failures have documented runbooks,
- break-glass changes are reconciled back into the appropriate source of truth.

---

## Break-Glass Model

Manual changes may occasionally be required during incident recovery.

Break-glass actions do not transfer authoritative ownership away from the normal control plane.

Infrastructure emergency changes must subsequently be reconciled with Terraform.

GitOps-managed Kubernetes emergency changes must subsequently be reconciled with Git.

Break-glass access must not become an alternate application deployment workflow.

---

## Platform Success Criteria

The platform should eventually allow an application engineer to move from:

```text
Code Written
```

to:

```text
Application Running
```

without needing to manually:

- provision AWS infrastructure,
- configure an EKS cluster,
- run Terraform,
- execute Kubernetes deployments,
- configure shared ingress infrastructure,
- configure shared observability infrastructure,
- manage long-lived AWS credentials.

The supported path should provide:

- automated validation,
- secure artifact production,
- immutable application artifacts,
- reviewed deployment intent,
- GitOps reconciliation,
- observable runtime behavior,
- documented operational procedures.

The platform should make the secure and supported path the easiest path.

---

## Related Architecture Decisions

The detailed architectural decisions are documented in:

- ADR-001 — Separate Infrastructure Provisioning from Application Delivery
- ADR-002 — Use GitOps and Argo CD for Kubernetes Delivery
- ADR-003 — Use GitHub OIDC for AWS Authentication

These ADRs define the ownership, delivery, authentication, and authorization boundaries summarized by this document.