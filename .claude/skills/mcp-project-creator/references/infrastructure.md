# Infrastructure Integration

How to register a new MCP server with the CDK infrastructure.

Replace `{{SERVER_NAME}}` with the kebab-case server name (e.g., `weather`).

## Contents

- [Adding to infrastructure.ts](#adding-to-infrastructurets)
- [Custom configuration](#custom-configuration)
- [Adding IAM permissions](#adding-iam-permissions)
- [Custom stacks (complex cases)](#custom-stacks-complex-cases)
- [Stack naming](#stack-naming)

## Adding to infrastructure.ts

Edit `infrastructure/bin/infrastructure.ts`:

```typescript
#!/usr/bin/env node
import 'source-map-support/register';
import * as cdk from 'aws-cdk-lib';
import { SharedStack } from '../lib/shared-stack';
import { McpDockerStack } from '../lib/mcp-docker-stack';

const app = new cdk.App();

const stage = app.node.tryGetContext('stage') || 'dev';

// Image tag: git SHA via the deploy workflow; falls back to 'latest' locally.
const imageTag = process.env.IMAGE_TAG || 'latest';

const env: cdk.Environment = {
  account: process.env.CDK_DEFAULT_ACCOUNT,
  region: process.env.CDK_DEFAULT_REGION || 'us-west-2',
};

const sharedStack = new SharedStack(app, `McpShared-${stage}`, {
  env,
  stage,
  description: 'Shared infrastructure for MCP servers',
});

// Register each server here:
new McpDockerStack(app, `McpServer-{{SERVER_NAME}}-${stage}`, {
  env,
  stage,
  serverName: '{{SERVER_NAME}}',
  imageTag,
  sharedResources: sharedStack.sharedResources,
  environment: {
    // EXAMPLE_API_BASE_URL: process.env.EXAMPLE_API_BASE_URL!,
  },
});

app.synth();
```

**Always pass `imageTag`.** Without it, CloudFormation will not detect new container images and the Lambda will not pick up code changes.

## Custom Configuration

Override defaults via construct props:

```typescript
new McpDockerStack(app, `McpServer-{{SERVER_NAME}}-${stage}`, {
  env,
  stage,
  serverName: '{{SERVER_NAME}}',
  imageTag,
  sharedResources: sharedStack.sharedResources,

  memorySize: 1024,   // default: 512
  timeout: 60,        // default: 30 (seconds)

  environment: {
    EXAMPLE_API_BASE_URL: process.env.EXAMPLE_API_BASE_URL!,
  },
});
```

## Adding IAM Permissions

The `McpDockerConstruct` exposes helpers for common permissions; for anything else, attach a policy to the execution role directly.

### Secrets Manager

```typescript
const serverStack = new McpDockerStack(app, /* ... */);

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

See [shared-env-vars.md](shared-env-vars.md) for the full Secrets Manager pattern.

### DynamoDB

```typescript
serverStack.server.grantDynamoDbRead(
  'arn:aws:dynamodb:us-west-2:123456789012:table/my-table'
);
```

### OpenSearch (cross-account)

```typescript
serverStack.server.grantOpenSearchAccess(
  '123456789012',         // account ID
  'us-west-2',            // region
  'my-opensearch-domain'  // domain name
);
```

### Custom policy

```typescript
serverStack.server.executionRole.addToPolicy(
  new cdk.aws_iam.PolicyStatement({
    effect: cdk.aws_iam.Effect.ALLOW,
    actions: ['s3:GetObject'],
    resources: ['arn:aws:s3:::my-bucket/*'],
  })
);
```

## Custom Stacks (complex cases)

For servers with non-trivial infrastructure (e.g., owning a DynamoDB table, OpenSearch domain), create a dedicated stack in `infrastructure/lib/stacks/{{SERVER_NAME}}.ts`:

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import { McpDockerConstruct } from '../mcp-docker-construct';
import type { SharedResources } from '../shared-stack';

export interface {{SERVER_NAME_PASCAL}}StackProps extends cdk.StackProps {
  stage: string;
  imageTag: string;
  sharedResources: SharedResources;
}

export class {{SERVER_NAME_PASCAL}}Stack extends cdk.Stack {
  public readonly server: McpDockerConstruct;

  constructor(scope: Construct, id: string, props: {{SERVER_NAME_PASCAL}}StackProps) {
    super(scope, id, props);

    this.server = new McpDockerConstruct(this, 'Server', {
      serverName: '{{SERVER_NAME}}',
      imageTag: props.imageTag,
      sharedResources: props.sharedResources,
      environment: { /* ... */ },
    });

    // Additional resources + permissions here
  }
}
```

Then register it in `infrastructure.ts` in place of the generic `McpDockerStack`.

## Stack Naming

| Resource | Pattern | Example |
|----------|---------|---------|
| CDK stack | `McpServer-{name}-{stage}` | `McpServer-weather-dev` |
| Shared stack | `McpShared-{stage}` | `McpShared-prod` |
| ECR repo | `mcp-{name}` | `mcp-weather` |
| Lambda function | `mcp-server-{name}-{stage}` | `mcp-server-weather-dev` |
