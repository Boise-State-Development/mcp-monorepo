# Shared Environment Variables & Secrets Manager

Two patterns this monorepo supports for getting config into Lambda:
1. **Shared variables** declared once in `infrastructure.ts`, reused across multiple server stacks (e.g., a common auth provider's JWKS URI).
2. **AWS Secrets Manager** for sensitive credentials, with a small Python helper that caches the value and falls back to env vars for local dev.

---

## Shared Variables Pattern

When two or more servers need the same value (an identity provider's config, a shared API endpoint), don't duplicate the GitHub variable per server. Declare it once.

### Step 1: Declare shared constants in `infrastructure.ts`

```typescript
// Top of infrastructure.ts, after `const env = { ... }`:

// Shared across multiple servers
const sharedAuthJwksUri = process.env.SHARED_AUTH_JWKS_URI;
const sharedAuthAudience = process.env.SHARED_AUTH_AUDIENCE;
const sharedAuthIssuer = process.env.SHARED_AUTH_ISSUER;
```

### Step 2: Pass them to each stack that needs them

```typescript
if (sharedAuthJwksUri) {
  new McpDockerStack(app, `McpServer-my-server-${stage}`, {
    env, stage, imageTag,
    serverName: 'my-server',
    sharedResources: sharedStack.sharedResources,
    environment: {
      // The Lambda sees them under shorter names
      JWKS_URI: sharedAuthJwksUri,
      ...(sharedAuthAudience && { JWT_AUDIENCE: sharedAuthAudience }),
      ...(sharedAuthIssuer && { JWT_ISSUER: sharedAuthIssuer }),
    },
  });
}
```

### Step 3: Add the shared vars to `.github/workflows/deploy.yml` *once*

Inside the "Deploy server stack" step's `env:` block:

```yaml
- name: Deploy server stack
  env:
    STAGE: ${{ inputs.environment || 'dev' }}
    IMAGE_TAG: ${{ github.sha }}
    # Shared across servers (do not duplicate per-server)
    SHARED_AUTH_JWKS_URI: ${{ vars.SHARED_AUTH_JWKS_URI }}
    SHARED_AUTH_AUDIENCE: ${{ vars.SHARED_AUTH_AUDIENCE }}
    SHARED_AUTH_ISSUER: ${{ vars.SHARED_AUTH_ISSUER }}
```

### When to use shared vs. server-specific

- **Shared** — the value is identical across servers (auth config, a common base URL).
- **Server-specific** — the value is unique to one server. Prefix with the server name: `{SERVER_NAME}_VAR`.

---

## AWS Secrets Manager Pattern

For sensitive credentials (API keys, passwords), pass the **secret name** as a Lambda env var rather than the secret value. The server reads from Secrets Manager at runtime, with caching, and falls back to env vars during local development.

### How it works

1. Secret stored in AWS Secrets Manager as JSON
2. Lambda receives the **secret name** in an env var
3. App fetches the secret on first use and caches it
4. Local dev checks env vars first, so a `.env` file can substitute

### Python helper (with caching + env fallback)

```python
import json
import os
from typing import Optional

import boto3
from botocore.exceptions import ClientError
from fastmcp.exceptions import ToolError

_credentials_cache: Optional[dict] = None


def get_credentials() -> dict:
    """Get credentials with caching. Env vars > Secrets Manager."""
    global _credentials_cache
    if _credentials_cache:
        return _credentials_cache

    # Local dev: env vars
    username = os.environ.get("MY_USERNAME")
    password = os.environ.get("MY_PASSWORD")
    if username and password:
        _credentials_cache = {"username": username, "password": password}
        return _credentials_cache

    # Production: Secrets Manager
    secret_name = os.environ.get("MY_CREDENTIALS_SECRET_NAME")
    if not secret_name:
        raise ToolError("Server misconfiguration: credentials not configured")

    try:
        region = os.environ.get("AWS_REGION", "us-west-2")
        client = boto3.client("secretsmanager", region_name=region)
        response = client.get_secret_value(SecretId=secret_name)
        _credentials_cache = json.loads(response["SecretString"])
        return _credentials_cache
    except ClientError as e:
        code = e.response.get("Error", {}).get("Code", "Unknown")
        raise ToolError(f"Failed to retrieve credentials: {code}")
```

The cache is module-scoped, so it persists across invocations within the same Lambda container — important because Secrets Manager calls cost money and add latency.

### CDK: grant the Lambda read access

```typescript
serverStack.server.executionRole.addToPolicy(
  new cdk.aws_iam.PolicyStatement({
    effect: cdk.aws_iam.Effect.ALLOW,
    actions: ['secretsmanager:GetSecretValue'],
    resources: [
      `arn:aws:secretsmanager:${env.region}:${env.account}:secret:${secretName}*`,
    ],
  })
);
```

The trailing `*` is intentional — Secrets Manager appends a random suffix to the ARN (e.g., `mysecret-AbCdEf`), so the ARN you specify in `resources` must allow the suffix.

### Workflow wiring

```yaml
env:
  MY_CREDENTIALS_SECRET_NAME: ${{ secrets.MY_CREDENTIALS_SECRET_NAME }}
```

Use `secrets.` (not `vars.`) for the secret-name reference — it isn't strictly sensitive on its own, but treating it as a secret keeps the naming consistent and avoids accidentally logging it.

### Local `.env` template

```
# Use plain env vars locally, no Secrets Manager call
MY_USERNAME=local-username
MY_PASSWORD=local-password
```
