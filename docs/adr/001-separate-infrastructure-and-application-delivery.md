# ADR-001: Separate Infrastructure Provisioning from Application Delivery

## Status

Accepted

## Context

PavedPath requires both cloud infrastructure provisioning and Kubernetes application delivery.

These concerns have different lifecycles, permissions, failure modes, and ownership boundaries.

Infrastructure includes resources such as:

- VPCs
- subnets
- IAM roles
- Amazon EKS
- Amazon ECR
- encryption resources
- supporting AWS services

Application delivery includes resources such as:

- Kubernetes Deployments
- Services
- application configuration
- image versions
- environment-specific workload configuration

A platform could technically manage both concerns from a single repository or deployment pipeline.

However, doing so would couple infrastructure changes to application releases and would require application delivery systems to receive broader permissions than necessary.

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

Application deployment will be managed through:

```text
pavedpath-sample-api
        |
        v
Application CI
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

Terraform will provision the infrastructure required by the platform.

Argo CD will reconcile Kubernetes application desired state.

Application CI will produce artifacts but will not directly deploy workloads to Kubernetes.

---

## Ownership Model

### Terraform Owns

Terraform is responsible for foundational AWS infrastructure, including:

- VPC
- subnets
- route configuration
- IAM foundations
- EKS cluster infrastructure
- ECR repositories
- encryption infrastructure
- GitHub OIDC integration
- supporting AWS services

### GitOps Owns

The GitOps layer is responsible for Kubernetes desired state, including:

- application Deployments
- Services
- Namespaces
- application configuration
- platform Kubernetes services
- policy configuration
- application image references

### Application Repositories Own

Application repositories are responsible for:

- source code
- dependencies
- tests
- container definitions
- application CI
- artifact creation

---

## Enforcement Rules

The separation will be reinforced through both repository design and permissions.

### Rule 1 — Application CI Does Not Deploy Directly

Application pipelines must not use direct deployment commands as the normal delivery path.

Examples include:

```text
kubectl apply
kubectl create
helm install
helm upgrade
```

Instead, application CI produces an immutable artifact and proposes a GitOps change.

---

### Rule 2 — Application CI Does Not Provision Shared Infrastructure

Application CI must not execute:

```text
terraform apply
```

against the shared PavedPath infrastructure.

Application workflows will not receive IAM permissions required to modify shared infrastructure.

---

### Rule 3 — Terraform Does Not Own Application Deployments

Terraform must not manage normal application Kubernetes workloads.

For example, application Deployments and Services should not be modeled as Terraform-managed Kubernetes resources.

This prevents Terraform and Argo CD from competing for ownership of the same Kubernetes objects.

---

### Rule 4 — Git Is the Application Deployment Source of Truth

The desired application deployment state must be represented in:

```text
pavedpath-gitops
```

Argo CD reconciles the cluster toward this desired state.

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
- clearer responsibility during incidents.

### Negative

This design introduces additional workflow components.

An application release may require:

1. application CI to create an artifact,
2. a GitOps change to reference that artifact,
3. Argo CD to reconcile the change.

This is more complex than allowing CI to deploy directly to Kubernetes.

The additional complexity is accepted because it creates stronger separation of concerns, auditability, and operational safety.

---

## Alternatives Considered

### Terraform Manages Kubernetes Applications

Terraform could provision AWS infrastructure and manage Kubernetes application resources.

This was rejected because application deployments change much more frequently than foundational infrastructure and would couple unrelated lifecycles.

It could also create ownership conflicts between Terraform and GitOps tooling.

### Application CI Deploys Directly to Kubernetes

GitHub Actions could authenticate to EKS and execute direct deployment commands.

This was rejected because CI would become the deployment control plane and Git would no longer reliably represent the desired application state.

It would also require application CI to receive Kubernetes deployment credentials.

### Single Repository

Infrastructure, GitOps configuration, and application source could exist in one repository.

This was rejected for PavedPath because separate repositories make ownership, permissions, CI behavior, and change lifecycles explicit.

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
                |
                v
               EKS
                ^
                |
        -------------------
         OWNERSHIP BOUNDARY
        -------------------
                |
                |
             Argo CD
                ^
                |
         pavedpath-gitops
                ^
                |
          GitOps Change
                ^
                |
           Amazon ECR
                ^
                |
          Application CI
                ^
                |
      pavedpath-sample-api

         APPLICATION PLANE
```

---

## Operational Implication

During incidents, engineers must identify which control plane owns the affected resource before making changes.

Infrastructure problems should normally be corrected through Terraform.

Application desired-state problems should normally be corrected through GitOps.

Manual changes may be necessary during break-glass recovery, but the authoritative configuration must subsequently be reconciled back into the appropriate Git repository.

---

## Review

This decision should be revisited if:

- platform scale requires a different repository strategy,
- infrastructure and application ownership models materially change,
- another deployment control plane replaces Argo CD,
- the GitOps operating model no longer meets platform requirements.