# PavedPath Platform Contract

## Purpose

The PavedPath Platform Contract defines the interface between application teams and the PavedPath platform.

Application teams should not need to understand or manage the underlying AWS and Kubernetes infrastructure in order to deploy a supported service.

The contract defines:

- what application teams provide,
- what the platform provides,
- what application CI is allowed to do,
- what the GitOps deployment layer owns,
- what security and operational requirements applications must satisfy,
- where responsibility boundaries exist.

The contract is intended to make the supported path predictable for both application engineers and platform engineers.

---

## Core Principle

PavedPath follows a separation-of-responsibilities model:

```text
Application Team
      |
      | provides
      v
Source + Tests + Container Definition + Runtime Requirements
      |
      v
Application CI
      |
      | validates and produces
      v
Verified Container Artifact
      |
      v
Platform Delivery Path
      |
      +---- Amazon ECR
      +---- GitOps
      +---- Argo CD
      +---- Amazon EKS
      |
      v
Running Application
```

The application team owns the application.

Application CI owns artifact validation and production.

GitOps owns deployment intent.

The platform team owns and operates the paved path used to run supported applications.

---

## Application Team Responsibilities

An application using PavedPath must provide the following capabilities.

### Source Code

The application repository contains the service source code and required dependencies.

Application teams own application behavior and application-specific dependencies.

### Tests

The application must provide automated tests that can run in CI.

At minimum, the reference paved path expects:

- unit tests,
- application startup validation,
- health endpoint validation.

Additional integration, contract, or end-to-end tests may be added where appropriate.

### Container Definition

The application must provide a reproducible container build.

For the PavedPath sample application this will be implemented using a `Dockerfile`.

The container must:

- start without interactive input,
- listen on a documented port,
- terminate correctly,
- return meaningful process exit codes,
- avoid requiring privileged execution unless explicitly approved.

### Health Endpoints

Services must expose health signals that Kubernetes can consume.

The reference application will provide:

```text
/health
/ready
```

`/health` represents application liveness.

`/ready` represents whether the application is ready to receive traffic.

These endpoints will be used to support Kubernetes health probes and operational visibility.

### Runtime Requirements

The application must declare runtime requirements such as:

- container port,
- CPU request,
- memory request,
- CPU limit,
- memory limit,
- environment-specific configuration,
- required secrets or external dependencies where applicable.

These values will be expressed through the supported PavedPath deployment interface as the platform is implemented.

---

## Platform Responsibilities

PavedPath provides the infrastructure and delivery capabilities required to run supported applications.

### AWS Infrastructure

The platform provides:

- networking,
- Amazon EKS infrastructure,
- IAM foundations,
- Amazon ECR repositories,
- encryption foundations,
- supporting AWS services.

These resources are provisioned through Terraform and owned by the infrastructure control plane.

### Container Registry

PavedPath provides approved Amazon ECR repositories for application container images.

Terraform owns the lifecycle and configuration of the repository.

Application CI may publish verified application images into approved repositories.

Publishing an artifact into an ECR repository does not transfer ownership of the repository to application CI.

### Kubernetes Runtime

PavedPath provides Amazon EKS as the application runtime.

Application teams do not provision or administer the EKS control plane.

Terraform owns the EKS infrastructure.

GitOps and Argo CD consume that infrastructure to run supported Kubernetes resources.

### GitOps Deployment

PavedPath provides Argo CD for Kubernetes deployment reconciliation.

Git defines the authoritative desired state for resources owned by the GitOps delivery plane.

Argo CD reconciles that desired state into Kubernetes according to the configured synchronization policy.

### Platform Services

PavedPath will provide shared capabilities such as:

- ingress,
- secrets integration,
- observability,
- policy enforcement,
- deployment health visibility.

Application teams consume these capabilities rather than independently rebuilding shared platform services.

---

## CI Contract

Application CI is responsible for artifact production and verification.

The expected CI workflow is:

```text
Application Commit
       |
       v
GitHub Actions
       |
       +---- Lint
       |
       +---- Test
       |
       +---- Security Validation
       |
       +---- Build Container
       |
       +---- Scan Container
       |
       v
Verified Container Artifact
       |
       v
Amazon ECR
```

CI ends when a verified artifact has been produced and published.

Application CI must not use direct Kubernetes deployment commands as the normal delivery mechanism.

Examples include:

```text
kubectl apply
kubectl create
helm install
helm upgrade
```

Deployment is owned by the GitOps delivery plane.

---

## CI Authentication Contract

Application CI will use approved temporary credentials when AWS access is required.

The normal authentication model is:

```text
GitHub Actions
       |
       | OIDC
       v
AWS IAM / STS
       |
       | AssumeRoleWithWebIdentity
       v
Temporary AWS Credentials
       |
       v
Approved Artifact Operations
```

Application CI must not depend on long-lived AWS access keys stored in GitHub as its normal authentication mechanism.

The application CI IAM role will be scoped to the operations required by the application delivery workflow, such as publishing container artifacts to approved Amazon ECR repositories.

Application CI must not receive broad permissions to administer shared PavedPath infrastructure.

---

## CD Contract

Continuous delivery begins with GitOps.

The expected flow is:

```text
Verified Container Image
          |
          v
GitOps Change
          |
          v
Pull Request
          |
          v
Review / Validation
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

The GitOps repository records the application version that should be running.

Argo CD is responsible for reconciling Kubernetes with that approved desired state.

Application CI does not become the deployment controller.

---

## Image Version Contract

Application deployments must use immutable image references as the deployment source of truth.

Mutable tags such as:

```text
latest
```

must not be used as the authoritative deployment reference.

The preferred deployment reference is an immutable image digest:

```text
repository@sha256:<digest>
```

Human-readable tags may also exist for artifact discovery or traceability, but the GitOps desired state should identify the exact artifact intended for deployment.

This provides a deterministic relationship between:

```text
Git Deployment Intent
        |
        v
Exact Container Artifact
```

---

## Promotion Contract

Application promotion occurs through changes to GitOps desired state.

Conceptually:

```text
Verified Artifact
       |
       v
Development Desired State
       |
       v
Validation
       |
       v
GitOps Promotion Change
       |
       v
Production Desired State
```

The exact environment-promotion mechanism will be implemented as the platform evolves.

Regardless of implementation, promotion must preserve the principle that the intended deployed artifact is represented in Git.

Application CI must not bypass GitOps to promote an application directly into a Kubernetes environment.

---

## Infrastructure Boundary

Application repositories must not provision shared AWS infrastructure.

Application CI therefore must not run infrastructure operations such as:

```text
terraform apply
```

against the shared PavedPath environment.

Shared infrastructure belongs to:

```text
pavedpath-infrastructure
```

and is managed independently of application delivery.

Application teams consume platform capabilities without taking ownership of the infrastructure that implements those capabilities.

---

## GitOps Boundary

Application deployment state belongs to:

```text
pavedpath-gitops
```

Application repositories produce artifacts.

The GitOps repository determines which artifact version should run.

This creates the following separation:

```text
Application Repository
        |
        | produces
        v
Container Artifact
        |
        | referenced by
        v
GitOps Repository
        |
        | reconciled by
        v
Argo CD
        |
        v
Kubernetes
```

Terraform must not simultaneously manage normal application Kubernetes workloads owned by GitOps.

Application CI must not bypass the GitOps repository as the normal deployment path.

---

## Argo CD Bootstrap Contract

Argo CD requires an initial bootstrap because it cannot reconcile GitOps state before it exists.

The platform bootstrap sequence is conceptually:

```text
Terraform / Infrastructure Bootstrap
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

The bootstrap process may create the minimum resources required to establish the GitOps control plane.

After Argo CD becomes operational, normal application and supported platform-service delivery must use the GitOps path.

Bootstrap must not become an alternate ongoing application deployment mechanism.

---

## Secrets Contract

Plaintext secrets must not be committed to application or GitOps repositories.

Applications consume secrets through a platform-supported secrets mechanism.

The exact implementation will be selected during the platform-services phase.

The design must preserve the following boundary:

```text
Secret Value
     |
     X
     |
Plaintext Git
```

Git may contain references or declarations describing how a secret should be consumed, but not the plaintext secret value itself.

The selected mechanism should support:

- least-privilege access,
- auditable secret access where supported,
- controlled secret lifecycle,
- separation between secret references and secret values.

---

## Security Contract

Application workloads are expected to:

- run with least privilege,
- avoid privileged containers unless explicitly approved,
- define resource requests and limits,
- expose health probes,
- use approved container registries,
- avoid plaintext credentials,
- use platform-provided identity mechanisms where available,
- satisfy supported security and policy validation.

The platform will progressively enforce these requirements through automated CI validation, GitOps validation, and Kubernetes policy controls.

Security controls should be automated where practical so that the supported path is also the easiest compliant path.

---

## Observability Contract

Applications must expose enough information for operators to determine whether the service is healthy.

At minimum, PavedPath applications will support:

- application logs,
- liveness status,
- readiness status,
- deployment status.

The platform will provide shared mechanisms for collecting and viewing operational telemetry.

As the platform evolves, supported telemetry may also include:

- application metrics,
- platform metrics,
- Kubernetes events,
- Argo CD synchronization and health state,
- distributed tracing,
- alerting.

Application teams are responsible for producing meaningful application-level signals.

The platform is responsible for providing supported mechanisms to collect and expose those signals.

---

## Deployment Validation Contract

PavedPath will progressively move validation earlier in the delivery lifecycle.

Application CI may validate:

- source formatting,
- tests,
- dependencies,
- container builds,
- container vulnerabilities.

GitOps validation may validate:

- manifest syntax,
- manifest rendering,
- Kubernetes schemas,
- platform policies,
- deployment configuration.

The purpose is to identify invalid or unsafe changes before they reach the runtime environment.

---

## Rollback Contract

Application rollback should normally occur through Git.

Conceptually:

```text
Problematic Deployment
        |
        v
GitOps Revert
        |
        v
Review / Merge
        |
        v
Argo CD
        |
        v
Previous Approved Desired State
```

Where the previous immutable artifact remains available, rollback should not require rebuilding that artifact.

The GitOps repository must continue to represent the desired deployment state after rollback.

---

## Break-Glass Contract

Direct changes to platform-managed resources may occasionally be required during incident recovery.

Break-glass changes are temporary exceptions, not normal delivery mechanisms.

If a GitOps-managed Kubernetes resource is changed directly, the authoritative desired state must subsequently be reconciled with Git.

If Terraform-managed infrastructure is changed manually, the authoritative Terraform configuration must subsequently be reconciled with the infrastructure.

Emergency access must not become an alternative application deployment path.

---

## Deployment Ownership Summary

| Capability | Application Team | Application CI | GitOps / Argo CD | Terraform |
|---|---|---|---|---|
| Source code | Owns | Reads | No | No |
| Tests | Owns | Runs | No | No |
| Container definition | Owns | Builds from | No | No |
| Container artifact | Defines source | Builds and publishes | References | Provides registry infrastructure |
| ECR repository | Consumes | Publishes to | No | Owns |
| AWS infrastructure | Consumes | No | No | Owns |
| EKS infrastructure | Consumes | No | Consumes | Owns |
| Kubernetes desired state | Requests/defines application requirements | No | Owns | No |
| Application deployment | Requests through GitOps | No direct deployment | Reconciles | No |
| Application image version | Produces artifact | Publishes artifact | Owns deployment reference | No |
| Secrets values | Consumes | Must not expose | References only | May provide supporting infrastructure |
| Application health behavior | Implements | Tests | Configures supported probes | Provides runtime infrastructure |
| Platform services | Consumes | No | Reconciles supported services | Provides underlying infrastructure |

The table describes normal ownership.

Break-glass procedures do not permanently change these boundaries.

---

## Paved Path Experience

The intended developer experience is:

```text
Write Code
    |
    v
Open Pull Request
    |
    v
CI Validates Change
    |
    v
Merge
    |
    v
Verified Container Produced
    |
    v
Amazon ECR
    |
    v
GitOps Change
    |
    v
Review / Validation
    |
    v
Merge
    |
    v
Argo CD Reconciles
    |
    v
Application Running
```

The developer should not need to manually:

- provision shared AWS infrastructure,
- configure the EKS control plane,
- run shared Terraform infrastructure deployments,
- execute Kubernetes application deployments,
- configure shared ingress infrastructure,
- configure shared observability infrastructure,
- manage long-lived AWS credentials.

The paved path should make secure application delivery routine rather than exceptional.

---

## Platform Consumer Expectations

An application team using the supported paved path should be able to expect:

- a documented application interface,
- repeatable CI behavior,
- an approved container registry,
- a defined GitOps deployment mechanism,
- predictable runtime requirements,
- standard health checks,
- shared platform capabilities,
- observable deployment state,
- documented operational procedures.

In return, the application must satisfy the requirements defined by this contract.

This creates a two-way agreement:

```text
Application Team
      |
      | satisfies application contract
      v
PavedPath
      |
      | provides supported platform capabilities
      v
Reliable Delivery Path
```

---

## Contract Evolution

This contract will evolve as PavedPath gains additional capabilities.

Changes to the contract should be:

- documented,
- version controlled,
- backward compatible where practical,
- communicated to application teams,
- validated through platform automation.

Significant architectural changes should be recorded using Architecture Decision Records.

---

## Related Architecture Decisions

This contract implements the boundaries established by:

- ADR-001 — Separate Infrastructure Provisioning from Application Delivery
- ADR-002 — Use GitOps and Argo CD for Kubernetes Delivery
- ADR-003 — Use GitHub OIDC for AWS Authentication

The architecture document provides the high-level system view, while this contract defines what platform consumers and platform operators can expect from that architecture.