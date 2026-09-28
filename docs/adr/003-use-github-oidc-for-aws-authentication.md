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

## Trust Model

AWS IAM role trust policies will restrict which GitHub identities may assume each role.

Trust conditions should be scoped as narrowly as practical.

Relevant identity information may include:

- GitHub organization or user
- repository
- branch
- environment
- workflow context

For example, an infrastructure role should only trust approved workflows from:

```text
ronycal/pavedpath-infrastructure
```

An application CI role should only trust approved workflows from:

```text
ronycal/pavedpath-sample-api
```

A role intended for one repository must not automatically be assumable by unrelated repositories.

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

Where practical, planning and application permissions may be separated.

For example:

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

GitHub Actions workflows using OIDC will require permission to request an identity token.

Conceptually:

```yaml
permissions:
  id-token: write
  contents: read
```

The workflow will then assume an approved AWS IAM role.

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

### Negative

OIDC federation requires additional initial IAM configuration.

Trust policies must be designed carefully.

An overly broad trust policy could allow unintended GitHub workflows to assume an AWS role.

For this reason, trust relationships must be reviewed with the same care as IAM permission policies.

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
Infrastructure Actions      Application Actions
       |                           |
       | OIDC                      | OIDC
       v                           v
Infrastructure Role         Application CI Role
       |                           |
       v                           v
Terraform / AWS                   ECR
```

The two workflows authenticate through the same federation mechanism but receive different AWS permissions.

---

## Operational Implication

If a GitHub workflow cannot authenticate to AWS, engineers should investigate:

1. the GitHub workflow's OIDC permissions,
2. the IAM role ARN being requested,
3. the IAM trust policy,
4. repository and branch trust conditions,
5. AWS CloudTrail records where applicable.

Static AWS credentials should not be introduced as a workaround for an OIDC configuration problem.

---

## Review

This decision should be revisited if:

- GitHub's workload identity model materially changes,
- AWS introduces a more appropriate federation mechanism,
- workflows move to a different CI platform,
- self-hosted runner architecture changes the authentication requirements.