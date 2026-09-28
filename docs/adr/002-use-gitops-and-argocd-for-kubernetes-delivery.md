# ADR-002: Use GitOps and Argo CD for Kubernetes Delivery

## Status

Accepted

## Context

PavedPath requires a consistent, auditable, and reproducible mechanism for deploying applications and shared platform services to Kubernetes.

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
- detection and correction of configuration drift,
- clear ownership of Kubernetes resources.

---

## Decision

PavedPath will use GitOps for Kubernetes application and platform-service delivery.

The repository:

```text
pavedpath-gitops
```

will contain the authoritative desired state for Kubernetes resources managed through the GitOps delivery plane.

Argo CD will compare that desired state with the actual state running in Amazon EKS and reconcile approved differences according to the configured synchronization policy.

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
Review / Merge
        |
        v
Argo CD
        |
        v
Amazon EKS
```

Application CI will produce deployable artifacts but will not directly deploy normal application workloads to Kubernetes.

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

CI produces a verified artifact.

CI does not directly deploy that artifact to Kubernetes.

### Continuous Delivery

Continuous Delivery begins when the desired application version is proposed in the GitOps repository.

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
Git Desired State
       |
       v
Argo CD
       |
       v
Amazon EKS
```

This preserves Git as the authoritative record of deployment intent.

---

## Source of Truth

Git is the authoritative desired-state source for Kubernetes resources owned by the GitOps delivery plane.

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

The Kubernetes cluster itself is not the authoritative configuration store for these resources.

Manual changes to GitOps-managed cluster resources create drift and may be overwritten by reconciliation.

The desired state stored in Git must ultimately represent the intended configuration.

---

## Reconciliation

Argo CD compares:

```text
Desired State in Git
        |
        | compare
        v
Actual Cluster State
```

When the states differ, Argo CD can report the application as:

```text
OutOfSync
```

Depending on the configured synchronization policy, reconciliation may require approval or may occur automatically.

PavedPath will explicitly configure synchronization behavior rather than relying on accidental or implicit defaults.

For workloads where automated synchronization is enabled, PavedPath may use capabilities such as:

- automatic synchronization,
- pruning of resources removed from Git,
- self-healing of supported configuration drift.

These capabilities will be enabled deliberately based on the lifecycle and risk of the resources being managed.

---

## Deployment State

Application deployment versions must be represented in Git.

PavedPath will prefer immutable container image references for deployed application versions.

For example:

```text
repository@sha256:<digest>
```

A mutable tag may be useful for human readability or artifact discovery, but the deployed desired state should prefer an immutable digest where practical.

Changing the deployed application version therefore becomes a Git change rather than an imperative cluster operation.

This provides a version-controlled relationship between deployment intent and the exact application artifact.

---

## Promotion Model

Application promotion will occur by changing desired state in Git rather than by directly modifying the target cluster.

Conceptually:

```text
Verified Image
      |
      v
Development Desired State
      |
      v
Validation
      |
      v
GitOps Change
      |
      v
Production Desired State
```

The exact environment-promotion strategy will be implemented as the platform evolves.

Promotion must preserve the principle that the intended deployed version is represented in Git.

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
        Review
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

This keeps Git and the cluster aligned and preserves an auditable rollback history.

Rollback does not require rebuilding a previously verified immutable artifact if that artifact remains available and approved for deployment.

---

## Direct Cluster Changes

Direct modification of GitOps-managed application workloads is not part of the normal deployment process.

Commands such as:

```text
kubectl apply
kubectl edit
kubectl patch
helm upgrade
```

may be useful for investigation or explicitly approved break-glass recovery, but they do not replace the GitOps workflow.

Emergency manual changes create a difference between Git and the actual cluster.

If an emergency manual change is required:

1. the change must be treated as temporary,
2. the incident or operational reason should be recorded,
3. the authoritative configuration must subsequently be updated or restored through Git,
4. Argo CD reconciliation must return the managed resource to an approved desired state.

Break-glass access must not become an alternative deployment path.

---

## Argo CD Ownership

Argo CD will manage Kubernetes resources that belong to the GitOps delivery plane.

Examples may include:

- application and platform-service Namespaces,
- Deployments,
- Services,
- ConfigMaps,
- Ingress resources,
- platform services,
- supported Kubernetes policies,
- application deployment configuration,
- application image references.

Terraform will not simultaneously manage these same Kubernetes objects.

This prevents Terraform and Argo CD from competing for ownership of identical resources.

Resource ownership must be clear before a resource is introduced into either control plane.

---

## Bootstrap Boundary

Argo CD creates a bootstrap consideration because the GitOps controller must exist before it can reconcile GitOps-managed resources.

PavedPath will treat initial GitOps control-plane bootstrap separately from normal application delivery.

Conceptually:

```text
Infrastructure / Bootstrap
          |
          v
        EKS
          |
          v
Initial Argo CD Installation
          |
          v
Argo CD Becomes Operational
          |
          v
pavedpath-gitops
          |
          v
Ongoing Kubernetes Reconciliation
```

The bootstrap mechanism may install the minimum resources required to establish Argo CD.

After the GitOps control plane is operational, normal application and supported platform-service delivery must flow through the GitOps repository.

Terraform must not become the ongoing owner of application workloads merely because it participates in initial platform bootstrap.

---

## Repository Structure

The GitOps repository will separate platform configuration, application configuration, and environment-specific desired state.

An initial structure may be:

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

The exact structure may evolve during implementation.

Changes to repository organization must preserve clear ownership and the ability to identify the desired state for each environment.

---

## GitOps Change Control

Changes to Kubernetes desired state should normally be introduced through pull requests.

The expected control flow is:

```text
Proposed Change
      |
      v
GitOps Branch
      |
      v
Pull Request
      |
      v
Validation
      |
      v
Review
      |
      v
Merge
      |
      v
Argo CD Reconciliation
```

As the platform evolves, automated validation may include:

- YAML validation,
- manifest rendering,
- policy checks,
- security checks,
- schema validation,
- configuration tests.

This allows deployment controls to move earlier in the delivery lifecycle while preserving Git as the deployment source of truth.

---

## Security Implications

Using Argo CD as the deployment controller reduces the need to provide application CI with Kubernetes deployment credentials.

Application CI requires permissions appropriate to artifact creation, such as publishing to an approved Amazon ECR repository.

Argo CD receives Kubernetes permissions appropriate to the resources it manages.

These are separate security boundaries.

Conceptually:

```text
Application CI
      |
      | artifact permissions
      v
Amazon ECR


Argo CD
      |
      | Kubernetes reconciliation permissions
      v
Amazon EKS
```

Application CI does not receive Kubernetes administrative permissions merely because it produces an application artifact.

Argo CD permissions should also follow least privilege and should not automatically imply unrestricted cluster administration.

---

## Observability and Auditability

PavedPath should make the GitOps delivery path observable.

Relevant operational evidence may include:

- Git commit history,
- pull request history,
- Argo CD synchronization status,
- Argo CD health status,
- Kubernetes events,
- application deployment status.

This allows engineers to correlate a running workload with the Git change that expressed the deployment intent.

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
- a consistent deployment workflow across applications,
- clearer Kubernetes resource ownership,
- a foundation for policy-based deployment controls.

### Negative

GitOps introduces additional components and concepts:

- Argo CD must be operated as a platform service,
- GitOps repository changes become part of application delivery,
- engineers must understand reconciliation behavior,
- synchronization policies require deliberate configuration,
- emergency manual changes require careful reconciliation afterward,
- the GitOps control plane requires an initial bootstrap mechanism.

The additional complexity is accepted because the platform gains stronger auditability, consistency, separation of concerns, and operational control.

---

## Alternatives Considered

### Direct Deployment from GitHub Actions

Application CI could authenticate to EKS and deploy workloads directly.

This was rejected because it would combine CI and deployment responsibilities and require application workflows to possess Kubernetes deployment credentials.

It would also weaken Git's role as the authoritative desired state.

### Manual Kubernetes Deployment

Engineers could deploy applications manually using `kubectl` or Helm.

This was rejected because it is difficult to reproduce, audit, standardize, and scale across engineering teams.

### Terraform Kubernetes Resources

Terraform could manage Kubernetes application resources.

This was rejected for normal application delivery because infrastructure and application deployment have different change frequencies and ownership models.

It could also create competing ownership between Terraform and Argo CD.

ADR-001 defines the separation between infrastructure provisioning and application delivery.

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
Verified Immutable Container
    |
    v
Amazon ECR
    |
    v
GitOps Pull Request
    |
    v
Review / Merge
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

Application CI produces the artifact but does not act as the Kubernetes deployment controller.

---

## Operational Implication

When a deployed application differs from its expected state, engineers should determine whether the issue exists in:

1. the application artifact,
2. the GitOps desired state,
3. Argo CD reconciliation,
4. Kubernetes runtime state,
5. an underlying platform or AWS dependency.

Changes should normally be corrected through the control plane that owns the affected resource.

For GitOps-managed Kubernetes resources, the lasting correction must be represented in Git.

---

## Review

This decision should be revisited if:

- Argo CD no longer meets platform requirements,
- another GitOps controller is adopted,
- the deployment model materially changes,
- platform scale requires a different reconciliation architecture,
- the Kubernetes resource ownership model materially changes.