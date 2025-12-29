# Versioning Policy

This repository uses **Conventional Commits** with automated versioning and releases.

## Version Strategy

We follow [Semantic Versioning](https://semver.org/) (SemVer):
- **MAJOR** (1.0.0 → 2.0.0): Breaking changes
- **MINOR** (1.0.0 → 1.1.0): New features (backward compatible)
- **PATCH** (1.0.0 → 1.0.1): Bug fixes (backward compatible)

## Automated Versioning

Versions are calculated automatically based on commit messages when merging to `main`:

### Commit Types

| Type | Version Bump | Description |
|------|--------------|-------------|
| `feat:` | **Minor** | New feature |
| `fix:` | **Patch** | Bug fix |
| `perf:` | **Patch** | Performance improvement |
| `feat!:` or `BREAKING CHANGE:` | **Major** | Breaking change |
| `docs:`, `style:`, `refactor:`, `test:`, `build:`, `ci:`, `chore:` | **None** | No version bump |

### Breaking Changes

To trigger a **major version bump**, use one of these formats:

```bash
# Option 1: Use ! after type
git commit -m "feat!: remove deprecated input"

# Option 2: Include BREAKING CHANGE in body
git commit -m "feat: change input format

BREAKING CHANGE: The 'old_input' parameter has been removed"
```

## Release Workflow

### Automatic Release on Merge

When you merge a PR to `main`:

1. **Version Calculation**: Analyzes all commits since last tag
2. **Version Update**: Updates `package.json` if version bump is needed
3. **Tag Creation**: Creates and pushes Git tag (e.g., `v1.2.3`)
4. **GitHub Release**: Creates a GitHub Release with changelog

### Manual Release

If you need to create a release manually:

```bash
# Update version in package.json
npm version 1.2.3 --no-git-tag-version

# Commit and push
git add package.json
git commit -m "chore(release): bump version to 1.2.3"
git push origin main

# Create and push tag
git tag v1.2.3
git push origin v1.2.3
```

## Using the Action

### Version Tags

Users can reference the action using:

- **Specific version**: `package-broker/cloudflare-deploy-action@v1.2.3`
- **Major version**: `package-broker/cloudflare-deploy-action@v1` (points to latest v1.x.x)
- **Latest**: `package-broker/cloudflare-deploy-action@main` (not recommended for production)

### Best Practice

Always pin to a specific version for production:

```yaml
uses: package-broker/cloudflare-deploy-action@v1.2.3
```

Use major version tag for automatic patch/minor updates:

```yaml
uses: package-broker/cloudflare-deploy-action@v1
```

## Commit Message Format

All commits must follow [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <subject>

[optional body]

[optional footer]
```

### Examples

```bash
# Minor version bump (new feature)
git commit -m "feat: add custom domain support"

# Patch version bump (bug fix)
git commit -m "fix: handle missing resource IDs gracefully"

# Major version bump (breaking change)
git commit -m "feat!: change input parameter names

BREAKING CHANGE: 'api_token' renamed to 'cloudflare_api_token'"

# No version bump (documentation)
git commit -m "docs: update README with usage examples"
```

## Version History

See [Releases](https://github.com/package-broker/cloudflare-deploy-action/releases) for full version history and changelog.
