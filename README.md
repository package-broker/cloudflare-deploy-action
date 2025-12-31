<div align="center">
  <img src="https://raw.githubusercontent.com/package-broker/docs/main/static/img/logo.svg" alt="PACKAGE.broker Logo" width="120" height="120">
  
  # package-broker/cloudflare-deploy-action
  
  **Reusable GitHub Action for deploying PACKAGE.broker to Cloudflare Workers**
  
  > Automated deployment with resource creation and configuration
</div>

**PACKAGE.broker** is a minimalistic, platform-agnostic Composer repository proxy and cache. This action automates its deployment to Cloudflare Workers.

Learn more: [package.broker](https://package.broker)

## Features

- **Thin Wrapper**: Delegates all deployment logic to `@package-broker/cloudflare` CLI (single source of truth)
- **Automated Resource Management**: Discovers or creates D1 databases, KV namespaces, R2 buckets, and Queues
- **Idempotent Operations**: Safe to run multiple times, handles existing resources gracefully
- **Custom Domain Support**: Configure custom domains with CNAME setup instructions
- **Hybrid Approach**: Generates `wrangler.toml` at runtime (never committed to repository)
- **Secret Management**: Idempotent secret handling (only sets if missing)

## ⚠️ Prerequisites: Repository Must Be Initialized

**This action requires your repository to be initialized with committed dependency files:**

- **`package.json`** - Must include `@package-broker/cloudflare` as a dependency
- **`package-lock.json`** - **REQUIRED** - The action uses `npm ci` for reproducible installs

### Why These Prerequisites?

1. **Reproducibility**: `package-lock.json` ensures `npm ci` installs exact dependency versions (including `@package-broker/cloudflare` and its pinned `wrangler` version)
2. **DRY Principle**: All deployment logic lives in `@package-broker/cloudflare` - this action is just a thin wrapper
3. **Version Control**: Track dependency versions and ensure CI matches local development
4. **Team Collaboration**: Other developers can clone and work with the project immediately

**Note**: The `wrangler.toml` is generated at runtime and should **not** be committed. It does not contain secrets (like `ENCRYPTION_KEY`), which are managed separately via Cloudflare secrets or GitHub Secrets.

### Recommended Workflow

1. **Initialize locally**:
   ```bash
   npm install @package-broker/cloudflare @package-broker/main @package-broker/ui
   # This creates package.json and package-lock.json
   ```
2. **Commit** `package.json` and `package-lock.json`
3. **Use this action** for automated deployments - it will call `@package-broker/cloudflare` CLI in CI mode

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
| `working_directory` | ❌ | `.` | Directory containing `package.json` / `package-lock.json` (relative to repository root) |
| `worker_name` | ❌ | Repository name | Worker name (alphanumeric, hyphens, underscores only) |
| `tier` | ❌ | `free` | Cloudflare Workers tier: `free` or `paid` |
| `node_version` | ❌ | `20` | Node.js version to use |
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

This action is a **thin wrapper** around `@package-broker/cloudflare` CLI:

1. **Validation**: Validates required inputs (API token, account ID, encryption key)
2. **Working Directory**: Resolves the directory containing your `package.json` / `package-lock.json`
3. **Dependency Check**: **Requires** `package-lock.json` to exist (fails fast if missing)
4. **Install Dependencies**: Runs `npm ci` to install exact versions from lockfile
5. **Deploy via CLI**: Calls `package-broker-cloudflare deploy --ci --json` with all configuration
6. **Parse Output**: Extracts `worker_url`, `database_id`, `kv_namespace_id` from JSON output
7. **Set Outputs**: Makes deployment results available to subsequent workflow steps

All deployment logic (resource discovery/creation, `wrangler.toml` generation, migrations, deployment) is handled by `@package-broker/cloudflare`, ensuring a single source of truth.

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

### "Invalid format for Authorization header" or "Invalid CLOUDFLARE_API_TOKEN format"

This error indicates the Cloudflare API token has formatting issues:

1. **Check for newlines/whitespace**: When copying the token from Cloudflare dashboard, ensure there are no leading/trailing spaces or newlines
2. **Verify token type**: Make sure you're using a **Cloudflare API Token** (not an API Key). Get one at: https://dash.cloudflare.com/profile/api-tokens
3. **Token format**: The token should be alphanumeric with hyphens/underscores only, typically 40+ characters
4. **Recreate the secret**: In GitHub, delete and recreate the `CLOUDFLARE_API_TOKEN` secret, ensuring no extra characters are included

**To fix**: 
- Go to GitHub Settings → Secrets and variables → Actions → Secrets
- Edit `CLOUDFLARE_API_TOKEN`
- Copy the token directly from Cloudflare (without any spaces or newlines)
- Save the secret

### "package-lock.json is required but not found"

This action **requires** `package-lock.json` to exist. To fix:

1. **Initialize your repository locally**:
   ```bash
   npm install @package-broker/cloudflare @package-broker/main @package-broker/ui
   ```
2. **Commit both files**:
   ```bash
   git add package.json package-lock.json
   git commit -m "chore: add package-broker dependencies"
   ```

The action uses `npm ci` which requires a lockfile for reproducible installs.

### "package.json is being updated/modified"

This action **does not modify** `package.json`. If you see changes:

1. Check other workflow steps that might be editing it
2. Check for `postinstall` scripts in dependencies that modify package.json
3. Ensure your `package.json` includes `@package-broker/cloudflare` as a dependency

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
