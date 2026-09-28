# ADR-002: Use GitOps and Argo CD for Kubernetes Delivery

## Status

Accepted

## Context

PavedPath requires a consistent and auditable mechanism for deploying applications and shared platform services to Kubernetes.

Several deployment models are possible.

Application CI could authenticate directly to Amazon EKS and execute commands such as:

```text
kubectl apply
helm upgrade
```

Alternatively, Kubernetes desired state can be stored in Git and reconciled into the cluster by a dedicated GitOps controller.

PavedPath requires:

- auditable deployment history,
- clear separation between CI and CD,
- reproducible cluster state,
- controlled application promotion,
- straightforward rollback,
- reduced Kubernetes credentials in application CI,
- detection and correction of configuration drift.

---

## Decision

PavedPath will use GitOps for Kubernetes application and platform-service delivery.

The repository:

```text
pavedpath-gitops
```

will contain the desired Kubernetes state.

Argo CD will continuously compare that desired state with the actual state running in Amazon EKS and reconcile differences.

The deployment flow will be:

```text
Application Repository
        |
        v
GitHub Actions
        |
        v
Test / Scan / Build
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
Argo CD
        |
        v
Amazon EKS
```

---

## CI and CD Separation

PavedPath separates Continuous Integration from Continuous Delivery.

### Continuous Integration

Application CI is responsible for:

- linting,
- testing,
- security validation,
- building container images,
- scanning container images,
- publishing verified images to Amazon ECR.

CI produces an artifact.

CI does not directly deploy the artifact to Kubernetes.

### Continuous Delivery

Continuous Delivery begins when the desired application version is changed in the GitOps repository.

```text
Verified Artifact
       |
       v
GitOps Pull Request
       |
       v
Review
       |
       v
Merge
       |
       v
Argo CD
       |
       v
EKS
```

This preserves Git as the authoritative record of deployment intent.

---

## Source of Truth

The authoritative desired state for Kubernetes workloads is Git.

The relationship is:

```text
Git
 |
 | desired state
 v
Argo CD
 |
 | reconciliation
 v
Kubernetes
```

The Kubernetes cluster itself is not the authoritative configuration store.

Manual changes to cluster resources may be overwritten by reconciliation.

---

## Reconciliation

Argo CD will continuously compare:

```text
Desired State
     |
     | compare
     v
Actual State
```

If the states differ, the application may be reported as:

```text
OutOfSync
```

Depending on the configured synchronization policy, Argo CD can reconcile the cluster back toward the desired state stored in Git.

This provides drift detection and correction.

---

## Deployment State

Application deployment versions must be represented in Git.

For example, the GitOps repository may reference an immutable container image:

```text
repository@sha256:<digest>
```

Changing the deployed application version therefore becomes a Git change rather than an imperative cluster operation.

This provides a version-controlled deployment history.

---

## Rollback Model

Rollback should normally be performed by reverting the Git change that introduced the unwanted deployment state.

The normal rollback model is:

```text
Problematic GitOps Change
          |
          v
      Git Revert
          |
          v
        Merge
          |
          v
       Argo CD
          |
          v
Previous Desired State
```

This keeps Git and the cluster aligned.

---

## Direct Cluster Changes

Direct modification of application workloads is not part of the normal deployment process.

Commands such as:

```text
kubectl apply
kubectl edit
kubectl patch
helm upgrade
```

may be useful for investigation or approved break-glass recovery, but they do not replace the GitOps workflow.

If an emergency manual change is required, the authoritative configuration must subsequently be reconciled with Git.

---

## Argo CD Ownership

Argo CD will manage Kubernetes resources that belong to the GitOps delivery plane.

Examples may include:

- Namespaces
- Deployments
- Services
- ConfigMaps
- Ingress resources
- platform services
- Kubernetes policies
- application deployment configuration

Terraform will not simultaneously manage these same resources.

This prevents competing controllers from attempting to own identical Kubernetes objects.

---

## Repository Structure

The exact structure will evolve as the platform is implemented, but the GitOps repository is expected to separate platform and application configuration.

A possible structure is:

```text
pavedpath-gitops/
|
├── platform/
│   ├── argocd/
│   ├── ingress/
│   ├── observability/
│   └── policies/
│
├── applications/
│   └── sample-api/
│
└── environments/
    ├── dev/
    └── prod/
```

The final structure will be implemented when the GitOps control plane is built.

---

## Security Implications

Using Argo CD as the deployment controller reduces the need to provide application CI with Kubernetes deployment credentials.

Application CI requires permissions appropriate to artifact creation, such as publishing to an approved Amazon ECR repository.

Argo CD receives Kubernetes permissions appropriate to the resources it manages.

These are separate security boundaries.

---

## Consequences

### Positive

GitOps provides:

- auditable deployment history,
- declarative desired state,
- drift detection,
- reproducible deployments,
- Git-based rollback,
- separation between CI and CD,
- reduced direct cluster access from application pipelines,
- a consistent deployment workflow across applications.

### Negative

GitOps introduces additional components and concepts:

- Argo CD must be operated as a platform service,
- GitOps repository changes become part of application delivery,
- engineers must understand reconciliation behavior,
- emergency manual changes require careful reconciliation afterward.

The additional complexity is accepted because the platform gains stronger auditability, consistency, and control.

---

## Alternatives Considered

### Direct Deployment from GitHub Actions

Application CI could authenticate to EKS and deploy workloads directly.

This was rejected because it would combine CI and deployment responsibilities and require application workflows to possess Kubernetes deployment credentials.

It would also weaken Git's role as the authoritative desired state.

### Manual Kubernetes Deployment

Engineers could deploy applications manually using kubectl or Helm.

This was rejected because it is difficult to reproduce, audit, standardize, and scale across engineering teams.

### Terraform Kubernetes Resources

Terraform could manage Kubernetes application resources.

This was rejected for normal application delivery because infrastructure and application deployment have different change frequencies and ownership models.

ADR-001 defines this separation.

---

## Resulting Control Model

```text
          APPLICATION DELIVERY

Developer
    |
    v
Application Git
    |
    v
Application CI
    |
    v
Immutable Container
    |
    v
Amazon ECR
    |
    v
GitOps Pull Request
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

Git records deployment intent.

Argo CD performs reconciliation.

Kubernetes runs the resulting workload.

---

## Review

This decision should be revisited if:

- Argo CD no longer meets platform requirements,
- another GitOps controller is adopted,
- the deployment model materially changes,
- platform scale requires a different reconciliation architecture.