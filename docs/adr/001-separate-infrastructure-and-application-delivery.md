# ADR-001: Separate Infrastructure Provisioning from Application Delivery

## Status

Accepted

## Context

PavedPath requires both cloud infrastructure provisioning and Kubernetes application delivery.

These concerns have different lifecycles, permissions, failure modes, change frequencies, and ownership boundaries.

Infrastructure includes resources such as:

- VPCs,
- subnets,
- route configuration,
- IAM roles,
- Amazon EKS,
- Amazon ECR repositories,
- encryption resources,
- supporting AWS services.

Application delivery includes resources such as:

- Kubernetes Deployments,
- Services,
- application configuration,
- application image versions,
- environment-specific workload configuration.

A platform could technically manage both concerns from a single repository or deployment pipeline.

However, doing so would couple infrastructure changes to application releases, blur ownership boundaries, and potentially require application delivery systems to receive broader permissions than necessary.

PavedPath therefore requires explicit separation between infrastructure provisioning, artifact production, and Kubernetes application delivery.

---

## Decision

PavedPath will maintain strict separation between infrastructure provisioning and application delivery.

Infrastructure will be managed through:

```text
pavedpath-infrastructure
        |
        v
GitHub Actions
        |
        v
Terraform
        |
        v
AWS
```

Application artifacts will be produced through:

```text
pavedpath-sample-api
        |
        v
Application CI
        |
        v
Test / Scan / Build
        |
        v
Amazon ECR
```

Application deployment will be managed through:

```text
Verified Application Artifact
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

Terraform will provision the foundational AWS infrastructure required by the platform.

Application CI will build, validate, and publish application artifacts but will not directly deploy normal workloads to Kubernetes.

Git will contain the authoritative desired state for Kubernetes resources owned by the GitOps delivery plane.

Argo CD will reconcile that desired state into Amazon EKS.

---

## Ownership Model

### Terraform Owns

Terraform is responsible for foundational AWS infrastructure, including:

- VPCs,
- subnets,
- route configuration,
- IAM foundations,
- EKS control plane and infrastructure required to operate the cluster,
- ECR repositories,
- encryption infrastructure,
- GitHub OIDC integration,
- supporting AWS services.

Terraform owns the lifecycle and configuration of these infrastructure resources.

### GitOps Owns

The GitOps layer is responsible for supported Kubernetes desired state, including:

- application Deployments,
- Services,
- application and platform-service Namespaces,
- application configuration,
- platform Kubernetes services,
- supported Kubernetes policy configuration,
- application image references.

Argo CD reconciles this desired state into Kubernetes.

### Application Repositories Own

Application repositories are responsible for:

- source code,
- dependencies,
- tests,
- container definitions,
- application CI,
- artifact creation,
- application-specific build and validation logic.

Application repositories produce deployable artifacts but do not become the Kubernetes deployment control plane.

---

## Shared Capability Boundary

Some platform resources are provisioned by one control plane and consumed by another.

Resource consumption does not transfer ownership of the underlying infrastructure.

### Amazon ECR Example

Terraform provisions and configures the ECR repository:

```text
Terraform
    |
    | provisions
    v
Amazon ECR Repository
```

Application CI publishes artifacts into that repository:

```text
Application CI
    |
    | pushes verified image
    v
Amazon ECR Repository
```

Terraform owns the lifecycle and configuration of the ECR repository.

Application CI owns the creation and publication of application container artifacts.

### Amazon EKS Example

Terraform provisions the EKS infrastructure:

```text
Terraform
    |
    | provisions
    v
Amazon EKS
```

Argo CD reconciles supported Kubernetes desired state onto that infrastructure:

```text
pavedpath-gitops
        |
        v
     Argo CD
        |
        | reconciles
        v
    Amazon EKS
```

Terraform owns the EKS infrastructure.

Argo CD owns supported Kubernetes desired state running on that infrastructure.

Consuming a platform capability does not transfer ownership of the underlying infrastructure.

---

## Bootstrap Boundary

Some platform components require a controlled bootstrap process.

Argo CD cannot reconcile GitOps state until both Amazon EKS and the initial Argo CD installation exist.

Conceptually:

```text
Terraform / Bootstrap
        |
        v
Amazon EKS
        |
        v
Initial Argo CD Installation
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

The bootstrap mechanism may create the minimum resources required to establish the GitOps control plane.

Bootstrap ownership does not imply ongoing ownership.

After Argo CD becomes operational, normal application and supported platform-service delivery must follow the GitOps model defined in ADR-002.

Terraform must not become the ongoing owner of application workloads merely because infrastructure automation participates in platform bootstrap.

---

## Authentication and Authorization Boundary

GitHub workflows that require AWS access will use the federated authentication model defined in ADR-003.

Conceptually:

```text
GitHub Actions
      |
      | OIDC
      v
AWS IAM Role
      |
      | temporary credentials
      v
Approved AWS Resources
```

Infrastructure and application workflows will use separate authorization boundaries.

For example:

```text
pavedpath-infrastructure
        |
        v
Infrastructure IAM Role
        |
        v
Approved Infrastructure Operations
```

and:

```text
pavedpath-sample-api
        |
        v
Application CI IAM Role
        |
        v
Approved Artifact Operations
```

Using the same GitHub OIDC federation mechanism does not mean these workflows receive the same AWS permissions.

Authentication and authorization must preserve the infrastructure/application ownership boundary.

---

## Enforcement Rules

The separation will be reinforced through repository design, workflow design, IAM permissions, Kubernetes permissions, and GitOps reconciliation.

### Rule 1 — Application CI Does Not Deploy Directly

Application pipelines must not use direct deployment commands as the normal delivery path.

Examples include:

```text
kubectl apply
kubectl create
helm install
helm upgrade
```

Instead, application CI produces a verified artifact and proposes or enables a GitOps change.

---

### Rule 2 — Application CI Does Not Provision Shared Infrastructure

Application CI must not execute:

```text
terraform apply
```

against the shared PavedPath infrastructure.

Application workflows will not receive IAM permissions required to administer shared platform infrastructure.

---

### Rule 3 — Terraform Does Not Own Normal Application Deployments

Terraform must not manage normal application Kubernetes workloads.

For example, application Deployments and Services should not be modeled as Terraform-managed Kubernetes resources.

This prevents Terraform and Argo CD from competing for ownership of the same Kubernetes objects.

---

### Rule 4 — Git Is the Application Deployment Source of Truth

The desired application deployment state must be represented in:

```text
pavedpath-gitops
```

Argo CD reconciles the cluster toward the approved desired state stored in Git.

The cluster itself is not the authoritative configuration source for GitOps-managed workloads.

---

### Rule 5 — A Resource Has One Normal Control-Plane Owner

A managed resource must have a clearly identified normal control-plane owner.

For example:

```text
AWS VPC                 -> Terraform
EKS infrastructure      -> Terraform
ECR repository          -> Terraform
Application image       -> Application CI
Kubernetes Deployment   -> GitOps / Argo CD
Kubernetes Service      -> GitOps / Argo CD
```

Multiple automation systems must not independently attempt to control the same resource lifecycle.

---

### Rule 6 — Permissions Follow Ownership

A workflow should receive only the permissions required for its responsibilities.

Application CI must not receive broad infrastructure-administration permissions.

Infrastructure automation must not be used as an alternate application deployment mechanism.

Argo CD must receive Kubernetes permissions appropriate to the resources it is expected to reconcile.

---

## Change Flow

The normal end-to-end application delivery path is:

```text
Developer
    |
    v
pavedpath-sample-api
    |
    v
Application CI
    |
    +---- Test
    +---- Scan
    +---- Build
    |
    v
Verified Container Image
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
Pull Request / Validation
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

The normal infrastructure change path is:

```text
Engineer
    |
    v
pavedpath-infrastructure
    |
    v
Pull Request / Validation
    |
    v
Terraform Plan
    |
    v
Approved Merge / Apply
    |
    v
AWS
```

These workflows intentionally have different lifecycles and authorization boundaries.

---

## Break-Glass Changes

Manual changes may occasionally be necessary during incident recovery.

A break-glass change does not transfer authoritative ownership away from the normal control plane.

If an emergency infrastructure change is made manually, the corresponding Terraform configuration must subsequently be reconciled.

If an emergency GitOps-managed Kubernetes change is made directly in the cluster, the corresponding Git desired state must subsequently be updated or restored.

Break-glass procedures must not become an alternative deployment mechanism.

---

## Consequences

### Positive

This decision provides:

- clear ownership boundaries,
- smaller permission scopes,
- independent infrastructure and application lifecycles,
- improved auditability,
- safer application deployments,
- easier rollback through Git,
- reduced risk of Terraform and Argo CD managing the same resource,
- clearer responsibility during incidents,
- stronger least-privilege boundaries,
- clearer troubleshooting paths.

### Negative

This design introduces additional workflow components and operational concepts.

An application release may require:

1. application CI to create and verify an artifact,
2. the artifact to be published to Amazon ECR,
3. a GitOps change to reference the artifact,
4. review and merge of that desired-state change,
5. Argo CD to reconcile the change.

The platform also requires an explicit bootstrap process before the GitOps control plane becomes operational.

This is more complex than allowing CI to deploy directly to Kubernetes.

The additional complexity is accepted because it creates stronger separation of concerns, auditability, security boundaries, and operational safety.

---

## Alternatives Considered

### Terraform Manages Kubernetes Applications

Terraform could provision AWS infrastructure and manage Kubernetes application resources.

This was rejected because application deployments change much more frequently than foundational infrastructure and would couple unrelated lifecycles.

It could also create ownership conflicts between Terraform and GitOps tooling.

### Application CI Deploys Directly to Kubernetes

GitHub Actions could authenticate to EKS and execute direct deployment commands.

This was rejected because CI would become the deployment control plane and Git would no longer reliably represent the desired application state.

It would also require application CI to receive Kubernetes deployment permissions.

### Single Repository

Infrastructure, GitOps configuration, and application source could exist in one repository.

This was rejected for PavedPath because separate repositories make ownership, permissions, CI behavior, and change lifecycles explicit.

The repository model may be revisited if platform scale or organizational ownership changes materially.

---

## Resulting Architecture

```text
                       PAVEDPATH

                INFRASTRUCTURE PLANE

             pavedpath-infrastructure
                       |
                       v
                GitHub Actions
                       |
                       v
                   Terraform
                       |
                       v
                      AWS
                 /           \
                v             v
        ECR Repository    Amazon EKS
                ^             |
                |             |
       publish  |             | hosts
                |             v
          Application       Argo CD
              CI              ^
                ^             |
                |             |
      pavedpath-sample-api    |
                              |
                       pavedpath-gitops

                APPLICATION DELIVERY PLANE
```

The diagram represents interaction between the planes, not shared ownership.

Terraform owns the underlying AWS infrastructure.

Application CI produces artifacts.

Git records Kubernetes deployment intent.

Argo CD reconciles GitOps-managed Kubernetes state.

---

## Operational Implication

During incidents, engineers must identify which control plane owns the affected resource before making lasting changes.

Infrastructure problems should normally be corrected through Terraform.

Application desired-state problems should normally be corrected through GitOps.

Application artifact problems should normally be corrected through the application repository and CI pipeline.

Authentication and authorization problems should be investigated through the appropriate GitHub OIDC and IAM boundary.

Manual changes may be necessary during break-glass recovery, but authoritative configuration must subsequently be reconciled back into the appropriate Git repository.

---

## Related Decisions

This ADR establishes the high-level ownership boundary.

Related decisions provide additional detail:

- ADR-002 defines GitOps and Argo CD as the Kubernetes delivery model.
- ADR-003 defines GitHub OIDC federation as the AWS authentication model for GitHub Actions.

Together, these decisions establish the foundational PavedPath control-plane boundaries.

---

## Review

This decision should be revisited if:

- platform scale requires a different repository strategy,
- infrastructure and application ownership models materially change,
- another deployment control plane replaces Argo CD,
- the GitOps operating model no longer meets platform requirements,
- authentication or authorization boundaries materially change.