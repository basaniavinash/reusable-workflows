# Reusable Workflows

Centralized GitHub Actions workflow library for the Mercury platform. Provides callable workflows for ECR image builds and Terraform operations — all using AWS OIDC federation, no stored credentials.

---

## Workflows

### `ecr-build-push.yml` — Build and push to ECR

```yaml
uses: bar-private-org/reusable-workflows/.github/workflows/ecr-build-push.yml@main
with:
  image-name: my-service
  dockerfile: ./Dockerfile
secrets: inherit
```

| Input | Description |
|-------|-------------|
| `image-name` | ECR repository name |
| `dockerfile` | Path to Dockerfile |
| `platforms` | Target platforms (default: `linux/amd64,linux/arm64`) |

**What it does:**
1. Assumes AWS role via OIDC (no stored credentials)
2. Authenticates Docker to ECR
3. Builds multi-platform image with Docker Buildx
4. Tags with git SHA, branch name, and semver tag (if applicable)
5. Pushes to ECR

---

### `terraform.yml` — Plan and apply

```yaml
uses: bar-private-org/reusable-workflows/.github/workflows/terraform.yml@main
with:
  working-directory: ./infra
  apply: ${{ github.ref == 'refs/heads/main' }}
secrets: inherit
```

| Input | Description |
|-------|-------------|
| `working-directory` | Path to Terraform root module |
| `apply` | Whether to apply (true) or plan-only (false) |

**What it does:**
1. Assumes AWS role via OIDC
2. Runs `terraform init` with remote backend
3. Runs `terraform plan`, posts diff as PR comment
4. Runs `terraform apply` if `apply: true` (typically only on main branch)

---

## Design decisions

### OIDC — zero stored credentials

Both workflows use `aws-actions/configure-aws-credentials` with `role-to-assume` from `secrets.AWS_ROLE_ARN`. GitHub exchanges its OIDC token for temporary AWS credentials — no access keys exist to rotate, leak, or accidentally commit.

The IAM role's trust policy restricts which repos and branches can assume it via the `sub` claim, preventing a compromised fork from deploying.

### Callable workflows as the single source of truth

Rather than copy-pasting build steps across 9 service repos, each repo calls these workflows with a few inputs. When a step needs updating (e.g., upgrading the Docker Buildx version), one PR here propagates to all consumers automatically.

### Multi-platform builds by default

Images are built for both `linux/amd64` and `linux/arm64` using Docker Buildx. This means the same image runs on x86 EC2 instances and Apple Silicon developer machines without separate tags.

### Conditional apply

The Terraform workflow accepts an `apply` boolean. Callers typically set `apply: ${{ github.ref == 'refs/heads/main' }}` — PRs get a plan diff as a comment, merges to main apply automatically. This enforces the review-before-deploy workflow without baking it into the reusable workflow itself.
