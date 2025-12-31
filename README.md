<div align="center">
  <img src="https://raw.githubusercontent.com/package-broker/docs/main/static/img/logo.svg" alt="PACKAGE.broker Logo" width="120" height="120">
  
  # package-broker/cloudflare-deploy-action
  
  **Reusable GitHub Action for deploying PACKAGE.broker to Cloudflare Workers**
  
  > Automated deployment with resource creation and configuration
</div>

**PACKAGE.broker** is a minimalistic, platform-agnostic Composer repository proxy and cache. This action automates its deployment to Cloudflare Workers.

Learn more: [package.broker](https://package.broker)

## Features

- **Zero Setup**: Works with completely empty repositories (creates `package.json` if missing)
- **Automated Resource Management**: Discovers or creates D1 databases, KV namespaces, R2 buckets, and Queues
- **Idempotent Operations**: Safe to run multiple times, handles existing resources gracefully
- **Custom Domain Support**: Configure custom domains with CNAME setup instructions
- **Hybrid Approach**: Generates `wrangler.toml` at runtime (never committed to repository)
- **Secret Management**: Idempotent secret handling (only sets if missing)

## ⚠️ Important: Local Initialization Recommended

**While this action works with completely empty repositories, it is strongly recommended to initialize your project locally first and commit the following files:**

- **`package.json`** - Ensures consistent dependency versions across environments
- **`package-lock.json`** - Locks dependency versions for reproducible builds
- **`wrangler.toml`** - Contains service IDs (database_id, kv_namespace_id, etc.) for faster deployments and local development

### Why Commit These Files?

1. **Reproducibility**: Locked dependency versions ensure consistent builds across CI/CD and local development
2. **Faster Deployments**: With `wrangler.toml` committed, the action can read existing resource IDs instead of discovering them every time
3. **Local Development**: Having `wrangler.toml` allows you to run `wrangler dev` locally for testing
4. **Version Control**: Track configuration changes and resource IDs over time
5. **Team Collaboration**: Other developers can clone and work with the project immediately

**Note**: The `wrangler.toml` should **not** contain secrets (like `ENCRYPTION_KEY`). Secrets are managed separately via Cloudflare secrets or GitHub Secrets.

### Recommended Workflow

1. **Initialize locally** using the CLI: `npx package-broker-cloudflare init`
2. **Commit** `package.json`, `package-lock.json`, and `wrangler.toml` (without secrets)
3. **Use this action** for automated deployments - it will use the committed `wrangler.toml` for faster execution

This approach gives you the best of both worlds: convenience for quick setups and proper version control for production deployments.

## Usage

### Basic Example

```yaml
name: Deploy to Cloudflare Workers

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Cloudflare Workers
        uses: package-broker/cloudflare-deploy-action@v1
        with:
          cloudflare_api_token: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          cloudflare_account_id: ${{ vars.CLOUDFLARE_ACCOUNT_ID }}
          encryption_key: ${{ secrets.ENCRYPTION_KEY }}
          worker_name: ${{ vars.WORKER_NAME }}
          tier: ${{ vars.CLOUDFLARE_TIER || 'free' }}
```

### With Custom Domain

```yaml
- name: Deploy to Cloudflare Workers
  uses: package-broker/cloudflare-deploy-action@v1
  with:
    cloudflare_api_token: ${{ secrets.CLOUDFLARE_API_TOKEN }}
    cloudflare_account_id: ${{ vars.CLOUDFLARE_ACCOUNT_ID }}
    encryption_key: ${{ secrets.ENCRYPTION_KEY }}
    worker_name: ${{ vars.WORKER_NAME }}
    tier: ${{ vars.CLOUDFLARE_TIER || 'free' }}
    domain: ${{ vars.CUSTOM_DOMAIN }}  # e.g., app.example.com
```

### Monorepo / Subdirectory Project

If your `package.json` lives in a subdirectory, set `working_directory` so the action does not generate a new `package.json` at the repository root:

```yaml
- name: Deploy to Cloudflare Workers
  uses: package-broker/cloudflare-deploy-action@v1
  with:
    cloudflare_api_token: ${{ secrets.CLOUDFLARE_API_TOKEN }}
    cloudflare_account_id: ${{ vars.CLOUDFLARE_ACCOUNT_ID }}
    encryption_key: ${{ secrets.ENCRYPTION_KEY }}
    working_directory: ./deploy
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `cloudflare_api_token` | ✅ | - | Cloudflare API token with Workers, D1, KV, R2, and Queues permissions |
| `cloudflare_account_id` | ✅ | - | Cloudflare account ID |
| `encryption_key` | ✅ | - | Base64-encoded encryption key for PACKAGE.broker |
| `working_directory` | ❌ | `.` | Directory containing `package.json` / `package-lock.json` / `wrangler.toml` (relative to repository root) |
| `package_broker_version` | ❌ | `latest` | Version to use for `@package-broker/*` when generating `package.json` (only used if `package.json` is missing) |
| `worker_name` | ❌ | Repository name | Worker name (alphanumeric, hyphens, underscores only) |
| `tier` | ❌ | `free` | Cloudflare Workers tier: `free` or `paid` |
| `node_version` | ❌ | `20` | Node.js version to use |
| `wrangler_version` | ❌ | `3` | Wrangler version to install (`3` or `4`) |
| `skip_ui_build` | ❌ | `false` | Skip UI build if pre-built assets are available |
| `skip_migrations` | ❌ | `false` | Skip migration application |
| `domain` | ❌ | - | Custom domain (e.g., `app.example.com`). Adds route configuration. User must manually create CNAME record. |

## Outputs

| Output | Description |
|--------|-------------|
| `worker_url` | Deployed Worker URL (e.g., `https://worker-name.subdomain.workers.dev`) |
| `database_id` | Created or found D1 database ID |
| `kv_namespace_id` | Created or found KV namespace ID |

## Required GitHub Configuration

### Secrets

- `CLOUDFLARE_API_TOKEN`: Cloudflare API token with required permissions
  - **See**: [Cloudflare API Token Permissions](https://package.broker/docs/deployment/cloudflare-api-token-permissions) for complete details
  - **Quick reference**: Workers Scripts:Edit, D1:Edit, KV:Edit, R2:Edit, Queues:Edit (paid tier), Account Settings:Read (optional)
- `ENCRYPTION_KEY`: Base64-encoded encryption key for PACKAGE.broker

### Variables

- `CLOUDFLARE_ACCOUNT_ID`: Your Cloudflare account ID
- `WORKER_NAME` (optional): Worker name (defaults to repository name)
- `CLOUDFLARE_TIER` (optional): `free` or `paid` (defaults to `free`)
- `CUSTOM_DOMAIN` (optional): Custom domain to use (e.g., `app.example.com`)

## How It Works

1. **Validation**: Validates all required inputs and configuration
2. **Package Management**: Creates `package.json` if missing, installs dependencies
3. **UI Build**: Builds UI assets if not already built
4. **Resource Discovery**: Discovers existing Cloudflare resources by name pattern:
   - D1 Database: `${worker_name}-db`
   - KV Namespace: `${worker_name}-kv`
   - R2 Bucket: `${worker_name}-artifacts`
   - Queue: `${worker_name}-queue` (paid tier only)
5. **Resource Creation**: Creates missing resources if not found
6. **Configuration**: Generates `wrangler.toml` at runtime with discovered/created resource IDs
7. **Secret Management**: Sets `ENCRYPTION_KEY` as Cloudflare secret (idempotent - only if missing)
8. **Migrations**: Applies database migrations to D1 database
9. **Deployment**: Deploys Worker to Cloudflare
10. **Post-Deployment**: Displays Worker URL and custom domain setup instructions (if applicable)

## Custom Domain Setup

When `domain` input is provided:

1. The action adds route configuration to `wrangler.toml`
2. After deployment, displays step-by-step CNAME setup instructions
3. You must manually create the CNAME record in Cloudflare DNS:
   - Go to Cloudflare Dashboard → Your Zone → DNS → Records
   - Add CNAME record:
     - **Name**: Subdomain (e.g., `app` for `app.example.com`)
     - **Target**: `${worker_name}.${account_subdomain}.workers.dev`
     - **Proxy**: Proxied (orange cloud) or DNS only (grey cloud)

**Note**: Wrangler API permissions don't include DNS management, so DNS records must be created manually.

## Idempotency

All operations are idempotent:

- **Resources**: Discovers existing resources by name before creating new ones
- **Secrets**: Only sets `ENCRYPTION_KEY` if it doesn't already exist
- **Migrations**: Handles duplicate column errors gracefully
- **Deployment**: Can be run multiple times safely

## Troubleshooting

### "Missing required CLOUDFLARE_API_TOKEN"

Ensure the secret is set in GitHub Settings → Secrets and variables → Actions → Secrets.

### "Invalid CLOUDFLARE_TIER"

Must be exactly `free` or `paid` (case-sensitive).

### "Failed to create or find D1 database"

Check that your API token has D1 Database permissions. See [Cloudflare API Token Permissions](https://package.broker/docs/deployment/cloudflare-api-token-permissions) for required permissions. Verify account ID is correct.

### "Migrations may have already been applied"

This is a warning, not an error. Migrations are idempotent and safe to run multiple times.

### Custom domain not working

1. Verify CNAME record is created in Cloudflare DNS
2. Check that the target points to `${worker_name}.${account_subdomain}.workers.dev`
3. Wait for DNS propagation (usually a few minutes)
4. Verify the route is configured in `wrangler.toml` (check deployment logs)

## License

AGPL-3.0

## Development

This action repository includes CI workflows that validate action syntax, inputs/outputs, and documentation. Integration tests are available when organization-level secrets are configured.

### Branch Protection

To ensure code quality, set up branch protection rules requiring status checks (`validate`, `lint-docs`) to pass before merging.

### Organization-Level Secrets for CI

To enable integration testing in CI, configure organization-level secrets:
- `CLOUDFLARE_API_TOKEN` (test account token)
- `TEST_ENCRYPTION_KEY` (test encryption key)
- `CLOUDFLARE_ACCOUNT_ID` (test account ID)

Grant access to `package-broker/cloudflare-deploy-action` repository in organization settings.

## Support

- Documentation: https://package.broker/docs
- Issues: https://github.com/package-broker/cloudflare-deploy-action/issues
