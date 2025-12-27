# CI/CD Setup Summary

This document summarizes the CI/CD setup for the `deploy-action` repository.

## CI Workflow Overview

The `.github/workflows/ci.yml` workflow includes three jobs:

### 1. Validate Job
- **Purpose**: Validates action syntax and structure
- **Runs on**: All pushes and PRs
- **Checks**:
  - Action YAML syntax (required fields)
  - Required inputs are marked correctly
  - Required outputs are defined
  - Shell script best practices

### 2. Integration Test Job
- **Purpose**: Tests the action with real Cloudflare resources
- **Runs on**: 
  - Main branch (when organization secrets are available)
  - Manual workflow dispatch
  - **NOT on PRs** (for security - prevents secret exposure)
- **Requirements**: Organization-level secrets must be configured
- **What it does**:
  - Executes the action with test credentials
  - Creates test resources (D1, KV, R2, Worker)
  - Verifies outputs are set correctly
  - Cleans up test resources automatically

### 3. Lint Docs Job
- **Purpose**: Validates documentation quality
- **Runs on**: All pushes and PRs
- **Checks**:
  - README.md exists
  - Required documentation sections are present

## Branch Protection

See [BRANCH_PROTECTION.md](BRANCH_PROTECTION.md) for detailed setup instructions.

**Quick Setup**:
1. Go to **Settings** → **Branches**
2. Add protection rule for `main`
3. Require status checks: `validate`, `lint-docs`
4. Optionally require PR reviews

## Organization Secrets Setup

### Why Organization Secrets?

Organization-level secrets allow:
- **Shared access**: Multiple repositories can use the same test credentials
- **Centralized management**: Update secrets in one place
- **Security**: Secrets are not exposed in PRs from forks
- **CI testing**: Enable integration tests without exposing secrets

### Required Secrets

**Secrets** (Settings → Secrets and variables → Actions → Secrets):
- `CLOUDFLARE_API_TOKEN` - Test Cloudflare API token
- `TEST_ENCRYPTION_KEY` - Test encryption key (Base64-encoded)

**Variables** (Settings → Secrets and variables → Actions → Variables):
- `CLOUDFLARE_ACCOUNT_ID` - Test Cloudflare account ID

### Setup Steps

1. **Navigate to Organization Settings**
   - Go to your GitHub organization
   - **Settings** → **Secrets and variables** → **Actions**

2. **Add Secrets**
   - Click **New repository secret**
   - Add each secret listed above
   - Set repository access to `package-broker/deploy-action`

3. **Add Variables**
   - Click **New repository variable**
   - Add `CLOUDFLARE_ACCOUNT_ID`
   - Set repository access to `package-broker/deploy-action`

4. **Verify Access**
   - Go to repository → **Settings** → **Secrets and variables** → **Actions**
   - Organization secrets should be listed (read-only)

### Test Account Recommendations

**Best Practice**: Use a dedicated Cloudflare test account:

1. Create a separate Cloudflare account (or sub-account) for testing
2. Generate a scoped API token with minimal permissions:
   - Workers Scripts: Edit
   - D1: Edit
   - KV: Edit
   - R2: Edit
   - Account Settings: Read (optional)
3. **Do NOT** grant:
   - DNS management (not needed)
   - Zone settings (not needed)
   - Admin permissions

### Security Considerations

- **Organization secrets are NOT available to PRs from forks** (GitHub security feature)
- Integration tests only run on main branch or manual dispatch
- Test resources are automatically cleaned up after tests
- Use a dedicated test account (not production)

## Testing the CI Setup

### Test Validation Job

1. Create a test branch
2. Make a breaking change (e.g., remove `required: true` from an input)
3. Open a PR
4. Verify the `validate` job fails
5. Fix the issue
6. Verify the job passes

### Test Integration Job

1. Ensure organization secrets are configured
2. Push to main branch (or manually dispatch workflow)
3. Check the **Actions** tab
4. Verify `integration-test` job runs
5. Verify test resources are created and cleaned up

### Test Branch Protection

1. Create a PR with failing CI checks
2. Verify the PR is blocked from merging
3. Fix the issues
4. Verify the PR can be merged after checks pass

## Troubleshooting

### "Organization secrets not available"

- **For PRs**: This is expected - secrets are not available to PRs from forks
- **For main branch**: Verify secrets are set at organization level and repository access is granted

### "Integration test not running"

- Check if organization secrets are configured
- Verify the condition: `if: (github.ref == 'refs/heads/main' || github.event_name == 'workflow_dispatch') && (secrets.CLOUDFLARE_API_TOKEN != '' && vars.CLOUDFLARE_ACCOUNT_ID != '')`
- Integration tests only run on main branch or manual dispatch

### "Required status checks must pass"

- Check which checks are failing in the **Actions** tab
- Fix the issues in your PR
- Push new commits to trigger re-runs

### "Test resources not cleaned up"

- Check the cleanup step logs
- Manually clean up using the commands shown in the logs
- Resources are named: `test-action-ci-{run_id}-{resource}`

## Workflow Status Badge

Add a status badge to your README:

```markdown
![CI](https://github.com/package-broker/deploy-action/workflows/CI/badge.svg)
```

## References

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub Branch Protection](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- [Organization Secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-an-organization)
