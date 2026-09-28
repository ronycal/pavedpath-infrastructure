# PavedPath Platform Contract

## Purpose

The PavedPath Platform Contract defines the interface between application teams and the PavedPath platform.

Application teams should not need to understand or manage the underlying AWS and Kubernetes infrastructure in order to deploy a supported service.

The contract defines:

- what application teams provide,
- what the platform provides,
- what application CI is allowed to do,
- what the GitOps deployment layer owns,
- and where responsibility boundaries exist.

---

## Core Principle

PavedPath follows a simple operating model:

```text
Application Team
      |
      | provides
      v
Code + Tests + Container Definition + Runtime Requirements
      |
      v
PavedPath
      |
      | provides
      v
Build + Registry + Deployment + Runtime + Observability
```

The application team owns the application.

The platform team owns the paved path used to run it.

---

## Application Team Responsibilities

An application using PavedPath must provide:

### Source Code

The application repository contains the service source code and required dependencies.

### Tests

The application must provide automated tests that can run in CI.

At minimum:

- unit tests,
- application startup validation,
- health endpoint validation.

Additional integration tests may be added where appropriate.

### Container Definition

The application must provide a reproducible container build.

For the PavedPath sample application this will be implemented using a `Dockerfile`.

The container must:

- start without interactive input,
- listen on a documented port,
- terminate correctly,
- return meaningful process exit codes.

### Health Endpoints

Services must expose health signals that Kubernetes can consume.

The reference application will provide:

```text
/health
/ready
```

`/health` represents application liveness.

`/ready` represents whether the application is ready to receive traffic.

### Runtime Requirements

The application must declare runtime requirements such as:

- container port,
- CPU request,
- memory request,
- CPU limit,
- memory limit,
- environment-specific configuration.

These values will eventually be expressed through the supported PavedPath deployment interface.

---

## Platform Responsibilities

PavedPath provides the infrastructure and deployment capabilities required to run supported applications.

These include:

### AWS Infrastructure

The platform provides:

- networking,
- Kubernetes compute,
- IAM foundations,
- container registry,
- encryption foundations,
- supporting cloud services.

These resources are provisioned through Terraform.

### Container Registry

PavedPath provides Amazon ECR repositories for application container images.

Application CI may publish application images to approved repositories.

### Kubernetes Runtime

PavedPath provides Amazon EKS as the application runtime.

Application teams do not provision or administer the EKS control plane.

### GitOps Deployment

PavedPath provides Argo CD for deployment reconciliation.

Git defines the desired deployment state.

Argo CD reconciles that desired state into Kubernetes.

### Platform Services

PavedPath will provide shared capabilities such as:

- ingress,
- secrets integration,
- observability,
- policy enforcement,
- deployment health visibility.

Application teams consume these capabilities rather than independently rebuilding them.

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
       +---- Security Scan
       |
       +---- Build Container
       |
       +---- Scan Container
       |
       v
Amazon ECR
```

CI ends when a verified artifact has been produced and published.

Application CI must not use direct Kubernetes deployment commands as the normal delivery mechanism.

Examples of prohibited normal deployment paths include:

```text
kubectl apply
kubectl create
helm install
helm upgrade
```

Deployment is owned by the GitOps delivery plane.

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
Argo CD
          |
          v
Amazon EKS
```

The GitOps repository records the application version that should be running.

Argo CD is responsible for reconciling Kubernetes with that state.

---

## Image Version Contract

Application deployments must use immutable image references.

Mutable tags such as:

```text
latest
```

must not be used as the deployment source of truth.

The preferred deployment reference will be an immutable image digest:

```text
repository@sha256:<digest>
```

This ensures the GitOps repository identifies the exact artifact intended for deployment.

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

---

## Secrets Contract

Plaintext secrets must not be committed to application or GitOps repositories.

Applications should consume secrets through a platform-supported secrets mechanism.

The exact implementation will be selected during the platform services phase.

The design must preserve the following rule:

```text
Secret value
    X
    |
    v
Plaintext Git
```

Git may contain references or declarations describing how a secret should be consumed, but not the plaintext secret value itself.

---

## Security Contract

Application workloads are expected to:

- run with least privilege,
- avoid privileged containers unless explicitly approved,
- define resource requests and limits,
- expose health probes,
- use approved container registries,
- avoid plaintext credentials,
- use platform-provided identity mechanisms where available.

The platform will progressively enforce these requirements through automated validation and Kubernetes policies.

---

## Observability Contract

Applications must expose enough information for operators to determine whether the service is healthy.

At minimum, PavedPath applications will support:

- application logs,
- liveness status,
- readiness status,
- deployment status.

The platform will provide shared mechanisms for collecting and viewing operational telemetry.

Additional metrics and tracing may be added as the platform evolves.

---

## Deployment Ownership Summary

| Capability | Application Team | Application CI | GitOps / Argo CD | Terraform |
|---|---:|---:|---:|---:|
| Source code | Owns | Reads | No | No |
| Tests | Owns | Runs | No | No |
| Container build | Defines | Executes | No | No |
| Container image | Defines | Publishes | References | Registry infrastructure only |
| AWS infrastructure | No | No | No | Owns |
| EKS cluster | No | No | Consumes | Owns |
| Kubernetes desired state | No | No | Owns | No |
| Application deployment | Requests through Git | No direct deployment | Owns | No |
| Secrets values | Consumes | Must not expose | References only | Supporting infrastructure |
| Runtime health | Implements | Tests | Observes desired state | Provides infrastructure |

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
Container Produced
    |
    v
GitOps Change
    |
    v
Argo CD Reconciles
    |
    v
Application Running
```

The developer should not need to manually provision infrastructure or manually deploy Kubernetes resources.

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