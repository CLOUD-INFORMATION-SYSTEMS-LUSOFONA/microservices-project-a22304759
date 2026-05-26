# Microservices CI/CD Pipeline

A Java-based microservices project with a complete CI/CD pipeline built on GitHub Actions, Docker Hub, AWS, and Terraform.

---

## Repository Structure

```
project-root/
├── .github/
│   └── workflows/
│       ├── hello.yml           # Activity 1 – Hello Actions
│       ├── ci.yml              # Activity 2 – Java CI
│       ├── image.yml           # Activity 3 – Docker Build & Push
│       ├── aws-test.yml        # Activity 4 – AWS OIDC Test
│       ├── terraform.yml       # Activity 5 – Terraform Plan/Apply
│       ├── reusable-image.yml  # Activity 8 – Reusable Workflow
│       └── release.yml         # Activity 8 – Release Caller
├── services/
│   ├── user-service/
│   ├── product-service/
│   └── order-service/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── modules/
└── README.md
```

---

## Workflows

### 1. Hello Actions (`hello.yml`)
Runs on every push and manually via `workflow_dispatch`. Prints repository name, branch, commit SHA, and actor. Used to verify Actions is working.

**Trigger:** Any push, or manually from the Actions tab.

### 2. Java CI (`ci.yml`)
Runs on every Pull Request and on pushes to `main`. Validates, compiles, and tests the Java microservice using Maven. Uploads Surefire test reports as artifacts.

**Trigger:** Pull requests, push to `main`.  
**Blocks merge if:** Unit tests fail.

### 3. Docker Build & Push (`image.yml`)
Builds a Docker image for `product-service` and pushes it to Docker Hub on every push to `main`. Tags the image with both `latest` and the Git SHA.

**Trigger:** Push to `main`, or manually.

### 4. AWS OIDC Test (`aws-test.yml`)
Authenticates with AWS using OIDC (no stored keys) and runs `aws sts get-caller-identity` to confirm the assumed role ARN.

**Trigger:** Manual (`workflow_dispatch` only).

### 5. Terraform Plan & Apply (`terraform.yml`)
Runs the full Terraform workflow on any change to the `terraform/` directory. On Pull Requests, posts the plan output as a PR comment. On merge to `main`, runs `terraform apply` automatically.

**Trigger:** Pull requests or push to `main` when `terraform/**` files change.

### 6. Matrix Builds (`build-all.yml`)
Builds and pushes Docker images for all three services (`user-service`, `product-service`, `order-service`) in parallel using a matrix strategy.

**Trigger:** Push to `main`.

### 7. Environment-Gated Deploy
A `deploy-prod` job referencing the `production` environment. Pauses for manual approval from a required reviewer before running.

**Trigger:** After a successful build job.

### 8. Reusable Workflow (`reusable-image.yml` + `release.yml`)
Extracts Docker build/push logic into a reusable workflow. Called by `release.yml` on version tags, passing the service name as an input.

**Trigger:** Push of a tag matching `v*`.

---

## Required GitHub Secrets

| Secret | Description |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub personal access token |
| `AWS_ROLE_TO_ASSUME` | ARN of the IAM role for OIDC authentication |

Add these under **Settings → Secrets and variables → Actions**.

---

## How to Trigger Workflows

| Workflow | How to trigger |
|---|---|
| Hello Actions | Push any file, or click **Run workflow** in Actions tab |
| Java CI | Open or update a Pull Request |
| Docker Build | Push a commit to `main` |
| AWS OIDC Test | Click **Run workflow** in Actions tab |
| Terraform Plan | Open a PR that changes a file in `terraform/` |
| Terraform Apply | Merge a PR that changes a file in `terraform/` into `main` |
| Release | Push a tag: `git tag v1.0.0 && git push origin v1.0.0` |

---

## Docker Hub

Images are published at:

```
<your-dockerhub-username>/product-service:latest
<your-dockerhub-username>/product-service:<git-sha>
```

Pull the latest image:

```bash
docker pull <your-dockerhub-username>/product-service:latest
```

---

## Terraform

The `terraform/` directory manages AWS infrastructure. The workflow runs the standard Terraform steps automatically:

```
terraform fmt -check   → formatting check
terraform init         → initialise providers and backend
terraform validate     → syntax and configuration check
terraform plan         → preview changes (posted as PR comment)
terraform apply        → apply changes (main branch only, automatic)
```

To run locally:

```bash
cd terraform
terraform init
terraform plan
terraform apply
```

---

## AWS OIDC Setup

GitHub Actions authenticates with AWS without storing any credentials, using OpenID Connect (OIDC).

**Steps to configure:**
1. Create an OIDC Identity Provider in AWS IAM with URL `https://token.actions.githubusercontent.com` and audience `sts.amazonaws.com`.
2. Create an IAM role with a trust policy that allows your repository to assume it.
3. Attach the necessary permissions policy to the role.
4. Copy the role ARN into the `AWS_ROLE_TO_ASSUME` repository secret.

The `aws-test.yml` workflow can be used to verify the setup by running `aws sts get-caller-identity`.
