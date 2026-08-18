# CI Pipeline

This repository uses `.github/workflows/golden-path-ci.yml` as a reusable
GitHub Actions workflow. A service-specific workflow calls it and supplies the
small amount of configuration that differs between services.

## What the Reusable Workflow Does

`golden-path-ci.yml` is triggered with `workflow_call`, so it does not run by
itself. It accepts these inputs:

| Input                 | Default | Purpose                                            |
| --------------------- | ------- | -------------------------------------------------- |
| `node_version`        | `20`    | Node.js version used by lint and test jobs         |
| `terraform_version`   | `1.7.0` | Terraform version used by infrastructure jobs      |
| `run_terraform_plan`  | `false` | Enables `security-scan` and `terraform-plan`       |
| `run_terraform_apply` | `false` | Enables the OIDC-authenticated deployment job      |
| `build_and_push`      | `false` | Enables Docker image publishing and ECS deployment |

The workflow declares an optional `aws_role_arn` secret. The secret is passed
by the caller and is used only by jobs that authenticate to AWS.

### Jobs

| Job               | When it runs                                                 | Why it exists                                                                                                                                                                                                                                                                                          |
| ----------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `lint`            | Every invocation                                             | Installs the workspace dependencies and runs ESLint for both the backend and frontend. It catches style, syntax, and static-analysis problems early.                                                                                                                                                   |
| `test`            | Every invocation                                             | Runs the backend Jest suite with coverage. The job fails below 80% global lines or branch coverage and writes the coverage figures to the GitHub Actions summary.                                                                                                                                      |
| `security-scan`   | When `run_terraform_plan` is `true`                          | Runs Checkov against `infra/` and fails on HIGH-severity findings. This prevents infrastructure changes with serious security issues from passing CI.                                                                                                                                                  |
| `terraform-plan`  | When `run_terraform_plan` is `true`                          | Installs the requested Terraform version, validates the dev stack during initialization, creates a plan with mock VPC and subnet values, writes a summary, and uploads the plan artifact. It uses `terraform init -backend=false`, so pull requests do not need access to the remote S3 state backend. |
| `docker-build`    | Pull requests only, after `lint` and `test`                  | Builds the backend and frontend Dockerfiles without pushing images. This verifies that the images can be built before merge.                                                                                                                                                                           |
| `terraform-apply` | When `run_terraform_apply` is `true`, after `terraform-plan` | Assumes the configured AWS IAM role through GitHub OIDC, initializes the S3 backend, downloads the plan artifact, and applies it to the dev stack. It also publishes the deployed load balancer URL in the job summary.                                                                                |
| `build-and-push`  | When `build_and_push` is `true`, after `terraform-apply`     | Authenticates to ECR through OIDC, builds and pushes both images with `latest` and commit-SHA tags, then triggers a new ECS deployment.                                                                                                                                                                |

The reusable workflow requests `contents: read` and
`pull-requests: write` at workflow scope. AWS jobs narrow their job-level
permissions to `contents: read` and `id-token: write`. The caller must also
request `id-token: write`; otherwise GitHub will not make an OIDC token
available to the reusable workflow.

## Adopting the Golden Path

A new service team adds a caller workflow under `.github/workflows/`. This is
the minimum caller for lint, tests, Terraform security scanning, and a
credential-free Terraform plan on pushes to `main` and pull requests:

```yaml
name: Todo Service CI

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read
  pull-requests: write

jobs:
  call-golden-path:
    uses: ./.github/workflows/golden-path-ci.yml
    with:
      node_version: "20"
      run_terraform_plan: true
```

For this repository's deployment flow, the caller additionally enables apply
and image publishing on pushes to `main`, requests the OIDC permission, and
passes the repository secret:

```yaml
permissions:
  contents: read
  pull-requests: write
  id-token: write

jobs:
  call-golden-path:
    uses: ./.github/workflows/golden-path-ci.yml
    with:
      node_version: "20"
      run_terraform_plan: true
      run_terraform_apply: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
      build_and_push: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
    secrets:
      aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
```

The `if` expressions ensure that pull requests can validate the proposed
change but cannot apply infrastructure or publish images. `terraform-apply`
also depends on the plan job, and `build-and-push` depends on apply, so those
steps run in order on the main branch.

## Required Checks

Configure these job names as required status checks in the repository branch
protection rules:

- **`lint`** validates both JavaScript workspaces with ESLint. Required because
  code that cannot pass the repository's static checks should not merge.
- **`test`** runs the backend Jest tests and enforces 80% global lines and
  branches coverage. Required to protect API behavior and prevent coverage
  from silently declining.
- **`security-scan`** runs Checkov on the infrastructure and blocks HIGH
  severity findings. Required because infrastructure mistakes can expose
  services or data even when application tests pass.
- **`terraform-plan`** confirms that the dev stack can initialize and produce a
  Terraform plan with the expected inputs. Required to catch invalid Terraform
  before apply and to provide a reviewable plan artifact.
- **`docker-build`** builds both service images on pull requests. Required to
  catch broken Dockerfiles, missing build context files, and image build
  regressions before merge.

`terraform-apply` and `build-and-push` are deployment jobs rather than pull
request gates. They should be protected by the main-branch condition and
deployment permissions instead of being required for every pull request.

## Configuring AWS OIDC

1. Create or obtain an AWS IAM role whose trust policy allows the repository's
   GitHub Actions OIDC subject to assume it. Follow GitHub's AWS OIDC guidance
   and restrict the trust policy to the intended organization, repository, and
   branch or environment.
2. In the repository, open **Settings > Secrets and variables > Actions**.
3. Create a repository secret named `AWS_ROLE_ARN`.
4. Set its value to the complete role ARN, for example
   `arn:aws:iam::<account-id>:role/<role-name>`.
5. Keep the caller mapping exactly as follows:

   ```yaml
   secrets:
     aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
   ```

The `terraform-apply` and `build-and-push` jobs pass this value to
`aws-actions/configure-aws-credentials@v4` as `role-to-assume` and request
`id-token: write`. No long-lived AWS access key is stored in GitHub.

Despite the reusable workflow's shared secret interface, the current
`terraform-plan` job does not use AWS credentials: it uses
`terraform init -backend=false`, mock network variables, and the provider's
local validation settings. Therefore `AWS_ROLE_ARN` is not required for a
plan-only pull request. It must be configured before enabling
`run_terraform_apply` or `build_and_push`; those jobs need it to access AWS.
