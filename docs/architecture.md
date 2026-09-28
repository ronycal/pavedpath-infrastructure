# PavedPath Architecture

## Overview

PavedPath is a GitOps-driven Internal Developer Platform designed to provide application teams with a secure, repeatable, and observable path from source code to production.

The platform separates cloud infrastructure provisioning from application delivery.

The primary architectural principle is:

> Infrastructure provisioning and application deployment are separate systems with separate ownership, permissions, and deployment workflows.

---

## Goals

PavedPath is designed to:

- Reduce developer cognitive load.
- Standardize application delivery.
- Provide secure self-service deployment workflows.
- Create repeatable AWS infrastructure.
- Make Git the source of truth for Kubernetes desired state.
- Provide observable and auditable deployments.
- Reduce direct developer interaction with Kubernetes and AWS.
- Provide a foundation for platform automation and AI-assisted engineering workflows.

---

## Repository Architecture

PavedPath is divided into three repositories.

### pavedpath-infrastructure

Owns foundational AWS infrastructure.

Responsibilities include:

- Terraform
- Terraform state
- AWS networking
- Amazon EKS
- Amazon ECR
- IAM
- KMS
- GitHub OIDC integration
- Supporting AWS infrastructure

This repository does not own application deployments.

---

### pavedpath-gitops

Owns Kubernetes desired state.

Responsibilities include:

- Argo CD configuration
- Kubernetes platform services
- Kubernetes policies
- Namespaces
- Application deployment manifests
- Environment-specific deployment configuration
- Application image versions

This repository does not provision foundational AWS infrastructure.

---

### pavedpath-sample-api

Represents an application team consuming PavedPath.

Responsibilities include:

- Application source code
- Application dependencies
- Tests
- Dockerfile
- CI
- Application health endpoints
- Container image creation

This repository does not provision shared infrastructure or deploy directly to Kubernetes.

---

## Infrastructure Control Plane

Infrastructure changes follow this path:

```text
Platform Engineer
       |
       v
Infrastructure Pull Request
       |
       v
GitHub Actions
       |
       +---- Terraform Format
       |
       +---- Terraform Validate
       |
       +---- Terraform Plan
       |
       v
Review / Approval
       |
       v
Terraform Apply
       |
       v
AWS
       |
       +---- VPC
       +---- Networking
       +---- IAM
       +---- ECR
       +---- EKS
       +---- Supporting Services
```

Terraform owns AWS infrastructure.

Terraform does not own application workloads.

---

## Application Delivery Control Plane

Application delivery follows a separate path:

```text
Developer
    |
    v
Application Pull Request
    |
    v
GitHub Actions
    |
    +---- Lint
    +---- Test
    +---- Security Scan
    +---- Build
    |
    v
Container Image
    |
    v
Amazon ECR
    |
    v
GitOps Pull Request
    |
    v
pavedpath-gitops
    |
    v
Argo CD
    |
    v
Amazon EKS
```

Application CI creates artifacts.

Application CI does not directly deploy workloads to Kubernetes.

---

## GitOps Model

Git represents the desired state of workloads running on Kubernetes.

The deployment model is:

```text
Git
 |
 v
Desired State
 |
 v
Argo CD
 |
 v
Kubernetes
```

Argo CD continuously compares Git desired state with Kubernetes actual state.

When differences are detected, Argo CD reconciles the cluster toward the state defined in Git.

Direct application deployment using the following commands is not part of the normal deployment path:

```text
kubectl apply
helm upgrade
```

Administrative use of Kubernetes tooling may be required for troubleshooting or break-glass operations, but those actions do not replace Git as the source of truth.

---

## Infrastructure and Application Boundary

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
| VPC | IAM | ECR | EKS | Supporting Infrastructure    |
+--------------------------+----------------------------+
                           |
                           | provides runtime
                           v
+-------------------------------------------------------+
|               APPLICATION DELIVERY PLANE              |
|                                                       |
| pavedpath-sample-api                                  |
|          |                                            |
|          v                                            |
|    GitHub Actions                                     |
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

---

## Authentication Model

Long-lived AWS access keys must not be stored in GitHub.

GitHub Actions will authenticate to AWS using OpenID Connect (OIDC).

The intended authentication flow is:

```text
GitHub Actions
       |
       | OIDC identity token
       v
AWS IAM / STS
       |
       v
AssumeRole
       |
       v
Temporary AWS Credentials
```

Different workflows will receive different IAM permissions.

For example:

```text
Terraform Plan
      |
      +---- Read infrastructure state
      +---- Read AWS configuration

Terraform Apply
      |
      +---- Modify approved infrastructure

Application CI
      |
      +---- Push container images to ECR
```

Application CI does not receive permissions to modify shared AWS infrastructure.

---

## Desired Ownership Model

| Resource | Owner |
|---|---|
| VPC | Terraform |
| Subnets | Terraform |
| EKS cluster | Terraform |
| IAM foundations | Terraform |
| ECR repositories | Terraform |
| GitHub AWS OIDC | Terraform |
| Argo CD bootstrap | Infrastructure bootstrap |
| Kubernetes platform services | Argo CD / GitOps |
| Kubernetes policies | Argo CD / GitOps |
| Application manifests | Argo CD / GitOps |
| Application source | Application repository |
| Application tests | Application repository |
| Container image | Application CI |
| Application deployment version | GitOps repository |

---

## Security Principles

PavedPath follows these security principles:

1. No long-lived AWS credentials in GitHub.
2. Least-privilege IAM roles.
3. Infrastructure changes occur through reviewed Git workflows.
4. Application deployments occur through reviewed GitOps changes.
5. Application CI cannot modify shared infrastructure.
6. Secrets must not be stored in plaintext in Git.
7. Container images are scanned before promotion.
8. Infrastructure and application permissions remain separated.

---

## Operational Principles

PavedPath will be designed so that:

- Infrastructure changes are auditable.
- Application deployments are auditable.
- Deployment state can be reconstructed from Git.
- Application versions can be rolled back through Git.
- Platform services expose metrics and logs.
- Application workloads expose health signals.
- Platform failures have documented runbooks.

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
- execute kubectl deployments,
- configure shared ingress infrastructure,
- configure shared observability infrastructure.

The platform should make the secure and supported path the easiest path.