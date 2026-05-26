# README Template

Replace `{{SERVER_NAME}}` with the kebab-case server name.
Replace `{{SERVER_DESCRIPTION}}` with a one-sentence description.

## README.md template

````markdown
# {{SERVER_NAME}} MCP Server

{{SERVER_DESCRIPTION}}. Deployed as a Docker container on AWS Lambda.

## Features

- **Feature 1**: Brief description
- **Feature 2**: Brief description
- **Feature 3**: Brief description

## Available Tools

| Tool | Purpose |
|------|---------|
| `tool_name` | What it does |
| `another_tool` | What it does |

## Use Cases

### Basic Queries

- "Example query that uses the tool"
- "Another example"

### Advanced Queries

- "Complex query combining multiple filters"
- "Query that requires multiple tool calls"

### Clarification-Required Queries

- "Vague query that needs clarification"
  - *Agent should ask: "What specific criteria are you looking for?"*

## Quick Start

### Local Development

Prerequisite: [uv](https://docs.astral.sh/uv/getting-started/installation/) installed.

```bash
cd packages/{{SERVER_NAME}}
uv sync                          # installs deps; downloads Python 3.12 if missing
PORT=8000 uv run python app.py
# Server: http://localhost:8000
# MCP endpoint: POST http://localhost:8000/mcp
```

`.env` (gitignored) is loaded automatically by `app.py` via `python-dotenv`.

### Test with curl

```bash
curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'
```

## Configuration

### Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `EXAMPLE_VAR` | What it configures | Yes/No |

## Deployment

Push to `main` to trigger automatic deployment via GitHub Actions.
````

## Why include use cases?

- Helps AI assistants understand when and how to use each tool
- Shows users the kinds of queries the server can handle
- Documents expected behavior for ambiguous queries
- Serves as integration test scenarios

## Tips for good use cases

1. **Be specific** — use realistic example queries users would actually type
2. **Show variety** — simple, complex, and edge cases
3. **Document clarifications** — show when the AI should ask follow-up questions
4. **Group logically** — by query type or user intent

## Tool documentation format

Each tool entry should include:
- **Name**: the exact function name as defined under `@mcp.tool`
- **Purpose**: one sentence on what it does
- **When to use**: brief guidance

The detailed API (parameter types, return shape) is inspected by clients directly via the MCP protocol — no need to duplicate it in the README.
