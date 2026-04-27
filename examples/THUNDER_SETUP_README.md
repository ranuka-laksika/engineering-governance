# Thunder Repository - Dependency Validation Setup

## Overview

The Thunder repository now has automatic dependency validation against the approved dependency registry in `engineering-governance/dependency-registry/go.yaml`.

## What Was Set Up

A GitHub Actions workflow has been added to Thunder repo at:
```
thunder/.github/workflows/dependency-registry-validation.yml
```

## How It Works

When a PR is created in Thunder repo that modifies Go dependency files (`go.mod`, `go.sum`):

1. **Detects Changes**: Identifies which dependencies were added or updated
2. **Fetches Registry**: Downloads the approved dependency list from `wso2/engineering-governance`
3. **Validates**: Checks each dependency:
   - ✅ **APPROVED** if module exists in registry AND version satisfies constraint (e.g., `>=v1.2.3`)
   - ❌ **BLOCKED** if module is NOT in registry OR version doesn't satisfy constraint
4. **Posts Comment**: Adds detailed validation report to the PR
5. **Blocks/Allows PR**:
   - Workflow **fails** (blocks merge) if any dependencies are blocked
   - Workflow **passes** if all dependencies are approved

## Example Workflow Execution

### Scenario 1: All Dependencies Approved ✅

PR adds `github.com/golang-jwt/jwt/v5@v5.3.1`

Registry has:
```yaml
- module: github.com/golang-jwt/jwt/v5
  versions:
    - version: ">=v5.3.1"
```

**Result**: ✅ Validation passes, PR can be merged

---

### Scenario 2: Dependency Not in Registry ❌

PR adds `github.com/unknown/package@v1.0.0`

Registry does not have this module.

**Result**: ❌ Validation fails, PR is blocked

**Comment on PR**:
```
❌ BLOCKED: Module github.com/unknown/package is not in the approved dependency registry

Action Required:
1. Submit PR to engineering-governance to add this dependency
2. Wait for approval and merge
3. Re-run this validation
```

---

### Scenario 3: Version Too Old ❌

PR adds `github.com/redis/go-redis/v9@v9.15.0`

Registry has:
```yaml
- module: github.com/redis/go-redis/v9
  versions:
    - version: ">=v9.17.3"
```

**Result**: ❌ Validation fails (v9.15.0 < v9.17.3)

---

## Validation Rules

### Version Constraint Matching

The workflow uses **range-based** validation:

| Constraint in Registry | PR Version | Result |
|------------------------|------------|--------|
| `>=v1.2.3` | `v1.5.0` | ✅ Pass |
| `>=v1.2.3` | `v1.2.3` | ✅ Pass |
| `>=v1.2.3` | `v1.2.0` | ❌ Fail |
| `>=v2.0.0` | `v1.9.0` | ❌ Fail |
| `pseudo` | any pseudo-version | ✅ Pass |

### What Gets Validated

- ✅ **Direct dependencies** (explicit `require` in go.mod)
- ✅ **ADDED** dependencies (new to go.mod)
- ✅ **UPDATED** dependencies (version changed)
- ❌ **Indirect dependencies** (marked with `// indirect`) - NOT validated
- ❌ **Removed dependencies** - ignored

## Adding New Approved Dependencies

If a dependency is blocked, to add it to the registry:

1. Create PR in `wso2/engineering-governance`
2. Edit `dependency-registry/go.yaml`
3. Add new entry:
```yaml
- module: github.com/your/package
  versions:
    - version: ">=v1.0.0"  # Minimum approved version
      allowed_scopes:
        - "*"  # Allow all projects, or specify Thunder only
```
4. Get PR reviewed and merged
5. Re-run Thunder PR validation

## Workflow Triggers

The workflow runs when:
- PR is **opened**, **synchronized** (new commits), or **reopened**
- AND changes include `**/go.mod` or `**/go.sum` files

It checks **all** go.mod files in Thunder repo (including subdirectories like `tools/i18n-extractor/go.mod`).

## Integration with Existing Dependency Guard

Thunder already has a `dependency-guard.yml` workflow that:
- Blocks package installs when dependency files change
- Requires `dependencies-approved` label to bypass

The new `dependency-registry-validation.yml` is **complementary**:
- `dependency-guard.yml`: Prevents supply chain attacks by blocking installs
- `dependency-registry-validation.yml`: Ensures dependencies meet governance standards

Both workflows run independently and both must pass for PR to merge.

## Testing the Workflow

To test the validation:

1. Create a test branch in Thunder repo
2. Modify a go.mod file:
   ```bash
   cd tools/i18n-extractor
   go get github.com/test/package@v1.0.0  # Use an unapproved package
   ```
3. Create PR
4. Check workflow runs and posts validation report
5. Verify PR is blocked

## Maintenance

### Updating the Registry

The registry is maintained at:
- `wso2/engineering-governance/dependency-registry/go.yaml`

### Workflow Maintenance

The workflow file is at:
- `thunder/.github/workflows/dependency-registry-validation.yml`

Updates to the validation logic should be made in the Thunder repo.

## Technical Details

### Dependencies
- Python packages: `packaging`, `pyyaml`
- GitHub Actions: `actions/checkout@v4`, `actions/setup-go@v5`, `actions/github-script@v7`

### Permissions Required
```yaml
permissions:
  pull-requests: write  # To post comments
  contents: read        # To read code and registry
```

### Registry Access
Uses `${{ secrets.GITHUB_TOKEN }}` to fetch from `wso2/engineering-governance`.

If engineering-governance is private and Thunder doesn't have access, you may need to:
1. Make engineering-governance public, OR
2. Use a PAT (Personal Access Token) with repo access, OR
3. Configure GitHub App permissions
