# ADR-003: Use GitHub OIDC for AWS Authentication

## Status

Accepted

## Context

PavedPath uses GitHub Actions to automate infrastructure and application workflows that require access to AWS.

Examples include:

- Terraform planning
- Terraform application
- publishing container images to Amazon ECR
- querying approved AWS resources

GitHub Actions therefore requires a secure mechanism for obtaining AWS credentials.

One approach would be to create IAM users and store long-lived credentials in GitHub repository secrets.

For example:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

Long-lived credentials increase operational and security risk.

They require:

- secret storage,
- credential rotation,
- revocation procedures,
- protection against accidental disclosure.

PavedPath requires a mechanism that provides temporary AWS credentials and allows access to be restricted according to GitHub repository and workflow identity.

---

## Decision

PavedPath will use GitHub Actions OpenID Connect (OIDC) federation with AWS.

GitHub Actions will request an OIDC identity token.

AWS Security Token Service (STS) will validate the federated identity and issue temporary credentials by allowing an approved workflow to assume an IAM role.

The authentication flow is:

```text
GitHub Actions
       |
       | requests OIDC token
       v
GitHub OIDC Provider
       |
       | identity assertion
       v
AWS IAM
       |
       v
AWS STS
       |
       | AssumeRoleWithWebIdentity
       v
Temporary AWS Credentials
```

Long-lived AWS access keys will not be the normal authentication mechanism for GitHub Actions.

---

## Federation Configuration

The AWS IAM OIDC provider will use:

Provider URL:

```text
https://token.actions.githubusercontent.com
```

Audience:

```text
sts.amazonaws.com
```

GitHub Actions will exchange its OIDC token for temporary AWS credentials through AWS STS using `AssumeRoleWithWebIdentity`.

---

## Trust Model

AWS IAM role trust policies must restrict which GitHub identities may assume each role.

Trust relationships will be scoped as narrowly as practical and will follow the principle of least privilege.

At minimum, trust policies will validate:

- the expected OIDC audience,
- the GitHub repository identity,
- and the appropriate branch or GitHub environment where applicable.

The expected audience for the standard AWS authentication flow is:

```text
sts.amazonaws.com
```

The trust policy must evaluate the GitHub OIDC subject (`sub`) and must not grant unrestricted GitHub repository access.

Relevant GitHub identity context may include:

- GitHub organization or user,
- repository identity,
- branch,
- environment,
- workflow context.

For example, an infrastructure role should only trust approved identities associated with:

```text
ronycal/pavedpath-infrastructure
```

An application CI role should only trust approved identities associated with:

```text
ronycal/pavedpath-sample-api
```

A role intended for one repository must not automatically be assumable by unrelated repositories.

### Subject Claim Validation

The exact GitHub OIDC subject format must be verified during implementation.

PavedPath will prefer immutable repository identity claims where supported rather than relying solely on mutable repository names.

Trust policies must be tested against the actual OIDC claims emitted by the PavedPath repositories before infrastructure or application workflows are enabled.

The subject may vary depending on workflow context. For example, workflows using a GitHub environment may produce a different subject from workflows scoped directly to a branch.

For this reason, PavedPath will not assume a subject format until the corresponding workflow and trust relationship are implemented and validated.

---

## Trust Policy and Permission Policy Separation

IAM role trust policies and IAM role permission policies serve different purposes.

The trust policy answers:

> Who may assume this role?

The permission policy answers:

> What may the assumed role do?

For example, GitHub OIDC trust conditions determine whether an approved `pavedpath-sample-api` workflow may assume the application CI role.

The policies attached to that role determine whether the resulting temporary credentials may perform approved operations against the PavedPath Amazon ECR repository.

Both layers must follow the principle of least privilege.

Conceptually:

```text
GitHub Workflow Identity
          |
          v
     Trust Policy
          |
    "May this identity
     assume the role?"
          |
          v
       IAM Role
          |
          v
   Permission Policy
          |
     "What may this
       role do?"
          |
          v
    AWS Resources
```

Successful authentication through GitHub OIDC does not automatically grant broad access to AWS. Authorization remains constrained by the policies attached to the assumed IAM role.

---

## Permission Separation

PavedPath will use separate IAM roles for workflows with different responsibilities.

Conceptually:

```text
pavedpath-infrastructure
        |
        v
Infrastructure IAM Role
        |
        v
Terraform / AWS Infrastructure
```

and:

```text
pavedpath-sample-api
        |
        v
Application CI IAM Role
        |
        v
Approved Amazon ECR Operations
```

The application CI role must not inherit infrastructure-administration permissions merely because both workflows run in GitHub Actions.

---

## Terraform Permissions

Terraform workflows require permissions appropriate to the infrastructure they manage.

The exact permissions will be developed as the infrastructure implementation evolves.

The objective is least privilege rather than permanent broad administrative access.

PavedPath will target separate authorization boundaries for Terraform planning and Terraform application.

Pull request workflows should not automatically receive the same infrastructure modification permissions as approved apply workflows.

Conceptually:

```text
Pull Request
     |
     v
Terraform Plan Role
     |
     +---- Read state
     +---- Inspect infrastructure
     +---- Produce plan
```

and:

```text
Approved Main Workflow
          |
          v
Terraform Apply Role
          |
          +---- Modify approved infrastructure
```

If implementation constraints require temporarily sharing a role, that exception must be documented and revisited.

The final implementation will balance least privilege with maintainability.

---

## Application CI Permissions

Application CI requires a much smaller AWS permission scope.

The expected application workflow is:

```text
GitHub Actions
      |
      v
Assume Application CI Role
      |
      v
Amazon ECR
      |
      v
Push Container Image
```

Application CI should not require permissions to:

- create or modify VPCs,
- administer IAM,
- modify the EKS control plane,
- run Terraform against shared infrastructure,
- directly administer unrelated AWS services.

---

## Credential Lifetime

Credentials issued through AWS STS are temporary.

This reduces the risk associated with credential leakage compared with long-lived IAM access keys.

The workflow obtains credentials when required and they expire automatically.

---

## GitHub Actions Requirements

GitHub Actions workflows using OIDC require permission to request an identity token.

Conceptually:

```yaml
permissions:
  id-token: write
  contents: read
```

The `id-token: write` permission allows the workflow to request an OIDC token.

It does not itself grant permission to modify AWS resources.

AWS authorization is determined by the IAM role successfully assumed by the workflow and the policies attached to that role.

The workflow will assume an approved AWS IAM role using its GitHub OIDC identity.

No AWS secret access key is required in the repository for this authentication flow.

---

## Security Benefits

OIDC federation provides several benefits:

- no long-lived AWS credentials stored in GitHub,
- temporary credentials,
- repository-aware trust policies,
- easier credential lifecycle management,
- improved auditability,
- reduced secret rotation requirements,
- separation of workflow permissions.

---

## Consequences

### Positive

Using OIDC reduces dependence on static secrets and allows AWS access to be tied to GitHub workload identity.

Separate roles allow infrastructure and application workflows to operate with different permissions.

Credential expiration limits the lifetime of issued AWS credentials.

Separating trust policies from permission policies allows PavedPath to independently control which workflows may authenticate and what those authenticated workflows may do.

### Negative

OIDC federation requires additional initial IAM configuration.

Trust policies must be designed carefully.

An overly broad trust policy could allow unintended GitHub workflows to assume an AWS role.

Repository, branch, environment, and subject-claim changes may require corresponding updates to IAM trust policies.

For these reasons, trust relationships must be reviewed with the same care as IAM permission policies.

---

## Alternatives Considered

### IAM User Access Keys Stored in GitHub Secrets

GitHub Actions could use an IAM user's access key and secret key.

This was rejected because the credentials are long-lived and require storage, rotation, and revocation.

### Manually Supplied Temporary Credentials

Engineers could manually generate temporary credentials and place them into workflows.

This was rejected because it is operationally inefficient, difficult to automate, and unsuitable for a paved path.

### Self-Hosted Runner IAM Credentials

A self-hosted runner running inside AWS could receive credentials through its runtime identity.

This may be appropriate for some future workloads, but it introduces runner infrastructure and operational responsibilities that are not required for the initial PavedPath architecture.

---

## Resulting Authentication Model

```text
                         GitHub
              +---------------------------+
              |                           |
              v                           v
    Infrastructure Actions        Application Actions
              |                           |
              | OIDC                      | OIDC
              v                           v
    Infrastructure Role           Application CI Role
              |                           |
              v                           v
       Terraform / AWS                   ECR
```

The two workflow types authenticate through the same federation mechanism but receive different AWS permissions.

Within the infrastructure workflow, PavedPath will target further separation between planning and application authorization where practical.

---

## Operational Implication

If a GitHub workflow cannot authenticate to AWS, engineers should investigate:

1. the GitHub workflow's OIDC permissions,
2. the IAM role ARN being requested,
3. the configured OIDC provider and audience,
4. the IAM role trust policy,
5. repository, branch, environment, and subject conditions,
6. the role's attached permission policies if authentication succeeds but authorization fails,
7. AWS CloudTrail records where applicable.

Static AWS credentials should not be introduced as a workaround for an OIDC configuration problem.

Break-glass procedures, if required in the future, must be explicitly documented and must not silently replace the normal OIDC authentication path.

---

## Review

This decision should be revisited if:

- GitHub's workload identity model materially changes,
- AWS introduces a more appropriate federation mechanism,
- workflows move to a different CI platform,
- self-hosted runner architecture changes the authentication requirements,
- the platform's authorization boundaries materially change.