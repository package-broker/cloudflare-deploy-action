# package-broker/deploy-action

A reusable GitHub Action for deploying [PACKAGE.broker](https://package.broker) to Cloudflare Workers with automated resource creation and configuration.

## Features

- **Zero Setup**: Works with completely empty repositories (creates `package.json` if missing)
- **Automated Resource Management**: Discovers or creates D1 databases, KV namespaces, R2 buckets, and Queues
- **Idempotent Operations**: Safe to run multiple times, handles existing resources gracefully
- **Custom Domain Support**: Configure custom domains with CNAME setup instructions
- **Hybrid Approach**: Generates `wrangler.toml` at runtime (never committed to repository)
- **Secret Management**: Idempotent secret handling (only sets if missing)

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
        uses: package-broker/deploy-action@v1
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
  uses: package-broker/deploy-action@v1
  with:
    cloudflare_api_token: ${{ secrets.CLOUDFLARE_API_TOKEN }}
    cloudflare_account_id: ${{ vars.CLOUDFLARE_ACCOUNT_ID }}
    encryption_key: ${{ secrets.ENCRYPTION_KEY }}
    worker_name: ${{ vars.WORKER_NAME }}
    tier: ${{ vars.CLOUDFLARE_TIER || 'free' }}
    domain: ${{ vars.CUSTOM_DOMAIN }}  # e.g., app.example.com
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `cloudflare_api_token` | ✅ | - | Cloudflare API token with Workers, D1, KV, R2, and Queues permissions |
| `cloudflare_account_id` | ✅ | - | Cloudflare account ID |
| `encryption_key` | ✅ | - | Base64-encoded encryption key for PACKAGE.broker |
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

- `CLOUDFLARE_API_TOKEN`: Cloudflare API token with the following permissions:
  - Account: Workers Scripts:Edit
  - Account: Workers KV Storage:Edit
  - Account: Account Settings:Read
  - Zone: Zone Settings:Read (if using custom domain)
  - Zone: DNS:Read (if using custom domain)
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

Check that your API token has D1 Database permissions. Verify account ID is correct.

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

### CI/CD

This action repository includes CI workflows that:

- **Validate action syntax**: Ensures `action.yml` is properly formatted
- **Validate inputs/outputs**: Checks that required inputs are marked correctly
- **Lint documentation**: Verifies README structure
- **Integration tests**: Tests the action with real Cloudflare resources (requires organization secrets)

See [`.github/BRANCH_PROTECTION.md`](.github/BRANCH_PROTECTION.md) for branch protection setup and organization-level secrets configuration.

### Branch Protection

To ensure code quality, set up branch protection rules:

1. Go to **Settings** → **Branches**
2. Add protection rule for `main` branch
3. Require status checks: `validate`, `lint-docs`
4. Optionally require PR reviews

See [`.github/BRANCH_PROTECTION.md`](.github/BRANCH_PROTECTION.md) for detailed instructions.

### Organization-Level Secrets for CI

To enable integration testing in CI, configure organization-level secrets:

1. Go to **Organization Settings** → **Secrets and variables** → **Actions**
2. Add secrets:
   - `CLOUDFLARE_API_TOKEN` (test account token)
   - `TEST_ENCRYPTION_KEY` (test encryption key)
3. Add variables:
   - `CLOUDFLARE_ACCOUNT_ID` (test account ID)
4. Grant access to `package-broker/deploy-action` repository

See [`.github/BRANCH_PROTECTION.md`](.github/BRANCH_PROTECTION.md) for complete setup instructions.

## Support

- Documentation: https://package.broker/docs
- Issues: https://github.com/package-broker/deploy-action/issues
