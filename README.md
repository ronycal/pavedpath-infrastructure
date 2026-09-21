# PavedPath Infrastructure

Infrastructure-as-Code repository for the PavedPath Internal Developer Platform.

## Purpose

This repository owns the AWS infrastructure required to run PavedPath.

Infrastructure will be provisioned and managed using Terraform.

## Responsibilities

This repository will manage:

- AWS networking
- Amazon EKS
- Amazon ECR
- AWS IAM
- KMS
- GitHub Actions OIDC integration
- Terraform remote state
- Supporting AWS infrastructure

## Ownership Boundary

This repository owns **infrastructure provisioning only**.

It does not own application deployments.

Application workloads will be deployed through the separate PavedPath GitOps repository and reconciled by Argo CD.

## Architecture Principle

```text
GitHub
   |
   v
GitHub Actions
   |
   v
Terraform
   |
   v
AWS Infrastructure

Application Repository
        |
        v
       CI
        |
        v
Container Registry
        |
        v
GitOps Repository
        |
        v
Argo CD
        |
        v
Amazon EKS


Save it with **Ctrl+S**.

### Why we're documenting this immediately

We're establishing the ownership contract **before writing Terraform**.

If another engineer opens this repository six months from now, the README should immediately answer:

> What does this repository own?

and, equally importantly:

> What does this repository *not* own?

That's part of the Senior Platform Engineer mindset we're demonstrating.

---

# Step 1.3 — Create `.gitignore`

From the terminal:

```bash
code .gitignore

# Terraform
.terraform/
*.tfstate
*.tfstate.*
crash.log
crash.*.log

# Terraform variable files
*.tfvars
*.tfvars.json

# Terraform override files
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Terraform CLI
.terraformrc
terraform.rc

# Plan files
*.tfplan

# Environment files
.env
.env.*

# VS Code local settings
.vscode/

# OS files
.DS_Store
Thumbs.db