# Repo-Root Configuration

Files at the monorepo root that apply to every package.

## IAM Policies (in `/iam`)

### iam/trust-policy.json

OIDC trust policy for GitHub Actions. Replace `ACCOUNT_ID` and `YOUR_ORG/YOUR_REPO`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:*"
        }
      }
    }
  ]
}
```

### iam/deploy-policy.json

IAM permissions for the GitHub Actions deploy role. Scoped to:
- CloudFormation: `McpServer-*`, `McpShared-*`
- Lambda: `mcp-server-*`
- IAM roles: `McpServer-*`, `McpShared-*`
- CloudWatch logs: `/aws/lambda/mcp-server-*`
- SSM parameters: `/mcp-servers/*`
- S3 CDK assets buckets
- ECR: `mcp-*` (push, describe, create-repository)

## GitHub Actions Secrets

Required in repo settings (Settings → Secrets and variables → Actions):

| Secret | Description |
|--------|-------------|
| `AWS_DEPLOY_ROLE_ARN` | ARN of the IAM role assumed via OIDC |

Server-specific vars/secrets are added per [shared-env-vars.md](shared-env-vars.md) and [SKILL.md § Step 4](../SKILL.md).

## AWS OIDC Setup

One-time setup to let GitHub Actions deploy to AWS:

1. Create an OIDC identity provider in AWS IAM:
   - Provider URL: `https://token.actions.githubusercontent.com`
   - Audience: `sts.amazonaws.com`
2. Create an IAM role:
   - Trust policy from `iam/trust-policy.json`
   - Permissions from `iam/deploy-policy.json`
   - Suggested name: `github-actions-mcp-deploy`
3. Add the role ARN to GitHub as `AWS_DEPLOY_ROLE_ARN`

## Existing Root Configs

These apply to all packages:

- `package.json` — npm workspaces
- `tsconfig.base.json` — shared TypeScript settings
- `.eslintrc.json` — shared linting
- `.gitignore` — repo-wide exclusions
- `.nvmrc` — Node version

New packages inherit them automatically.
