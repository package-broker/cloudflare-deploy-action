# Branch Protection Setup Guide

This guide explains how to set up branch protection rules for the `deploy-action` repository to ensure code quality and prevent broken deployments.

## Overview

Branch protection rules enforce that:
1. All CI checks must pass before merging
2. PRs require review (optional, but recommended)
3. Direct pushes to main are prevented (optional)
4. Status checks are required

## Setup Instructions

### Step 1: Enable Branch Protection

1. Go to your repository on GitHub
2. Navigate to **Settings** → **Branches**
3. Under **Branch protection rules**, click **Add rule**
4. Configure the following:

### Step 2: Configure Branch Protection Rule

**Branch name pattern**: `main`

**Protection Settings**:

#### Required Status Checks
- ✅ **Require status checks to pass before merging**
- ✅ **Require branches to be up to date before merging**
- Select the following required checks:
  - `validate` (from CI workflow - **required**)
  - `lint-docs` (from CI workflow - **required**)
  - `integration-test` (optional - only runs if organization secrets are configured and on main branch)

#### Pull Request Reviews (Optional but Recommended)
- ✅ **Require a pull request before merging**
- ✅ **Require approvals**: `1` (or more)
- ✅ **Dismiss stale pull request approvals when new commits are pushed**

#### Restrictions (Optional)
- ✅ **Do not allow bypassing the above settings** (for administrators)
- ⚠️ **Restrict pushes that create files larger than 100 MB** (recommended)

#### Other Settings
- ⚠️ **Require conversation resolution before merging** (optional)
- ⚠️ **Require linear history** (optional - keeps history clean)

### Step 3: Save the Rule

Click **Create** to save the branch protection rule.

## What This Enforces

1. **All PRs must pass CI**: The `validate` and `lint-docs` jobs must succeed
2. **Integration tests**: If organization secrets are configured, integration tests will run
3. **Documentation quality**: README and documentation are validated
4. **Code quality**: Action syntax and structure are validated

## Testing Branch Protection

1. Create a test branch
2. Make a change that breaks validation (e.g., remove `required: true` from an input)
3. Open a PR
4. Verify that the PR is blocked with "Required status checks must pass"
5. Fix the issue
6. Verify the PR can be merged after checks pass

## Organization-Level Secrets Setup

To enable integration testing in CI, set up organization-level secrets:

### Step 1: Navigate to Organization Settings

1. Go to your GitHub organization
2. Navigate to **Settings** → **Secrets and variables** → **Actions**

### Step 2: Add Organization Secrets

Add the following **secrets**:

- **`CLOUDFLARE_API_TOKEN`**: Cloudflare API token for testing
  - Create a dedicated test account or use a test token
  - Use minimal permissions (Workers, D1, KV, R2)
  - **Important**: This token will be used by all repositories in the organization

- **`TEST_ENCRYPTION_KEY`**: Base64-encoded encryption key for testing
  - Generate with: `openssl rand -base64 32`
  - This is a test key, not for production use

### Step 3: Add Organization Variables

Add the following **variables**:

- **`CLOUDFLARE_ACCOUNT_ID`**: Cloudflare account ID for testing
  - Use a test account or dedicated test account ID

### Step 4: Configure Repository Access

1. In organization secrets/variables settings
2. Click on each secret/variable
3. Under **Repository access**, select:
   - **Selected repositories** → Choose `package-broker/deploy-action`
   - Or **All repositories** (if you want it available to all repos)

### Step 5: Verify Access

1. Go to the repository
2. Navigate to **Settings** → **Secrets and variables** → **Actions**
3. You should see organization secrets listed (they're read-only at repo level)

## Security Considerations

### Organization Secrets

- **Scope**: Organization secrets are available to all repositories you grant access to
- **Access Control**: Only organization owners and members with appropriate permissions can manage secrets
- **Audit**: GitHub logs all secret access in the audit log
- **Best Practice**: Use a dedicated test Cloudflare account for CI testing

### Test Account Setup

1. Create a separate Cloudflare account for testing (or use a test sub-account)
2. Generate a scoped API token with minimal permissions:
   - Workers Scripts: Edit
   - D1: Edit
   - KV: Edit
   - R2: Edit
3. Use this token only for CI/testing purposes
4. Rotate the token periodically

### Token Permissions

The test token should have:
- **Workers** → Workers Scripts → Edit
- **D1** → Edit
- **KV** → Edit
- **R2** → Edit
- **Account** → Account Settings → Read (optional, for better error messages)

**Do NOT** grant:
- DNS management (not needed for testing)
- Zone settings (not needed for testing)
- Admin permissions

## Troubleshooting

### "Required status checks must pass"

- Check the **Actions** tab to see which checks failed
- Fix the issues in your PR
- Push new commits to trigger re-runs

### "Organization secrets not available"

- Verify secrets are set at organization level
- Check repository access is granted
- For PRs from forks, secrets are intentionally not available (security)

### Integration tests not running

- Check if organization secrets are configured
- Verify the `integration-test` job condition: `if: github.ref == 'refs/heads/main' || github.event_name == 'workflow_dispatch'`
- Integration tests only run on main branch or manual dispatch (for security)

## Alternative: Repository-Level Secrets

If you prefer not to use organization secrets:

1. Set secrets at repository level (Settings → Secrets and variables → Actions)
2. Update the CI workflow to always run integration tests (remove the conditional)
3. **Note**: This means secrets are only available to this repository, not shared across the organization

## References

- [GitHub Branch Protection Rules](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- [GitHub Organization Secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-an-organization)
- [GitHub Actions Security](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
