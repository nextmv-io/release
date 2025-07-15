# Release Workflow

A reusable GitHub Actions workflow for automated package releases across Nextmv
repositories.

## Overview

This workflow automates the release process by:

- **Version Detection**: Automatically detects version changes in your package
- **Tag Management**: Creates and pushes version tags when releases are needed
- **Release Creation**: Generates GitHub releases with auto-generated notes
- **Smart Filtering**: Only releases when appropriate based on branch and
  version type
- **Multi-Language Support**: Supports both Go and Python projects
- **Slack Integration**: Notifies Slack channels for stable releases

## Features

### Intelligent Release Logic

The workflow implements smart release logic:

- ✅ **Dev versions** (alpha, beta, rc, dev, pre, a, b) are released from
  **feature branches**
- ✅ **Stable versions** are released from the **main branch** (`develop`)
- ❌ **Stable versions** are **NOT** released from feature branches
- ❌ **Dev versions** are **NOT** released from the main branch
- ❌ **No release** if version file hasn't changed
- ❌ **No release** if the version tag already exists

### Supported Languages

- **Python**: Uses `hatch` to determine version from `pyproject.toml`
- **Go**: Reads version from a `VERSION` file

## Usage

To use this workflow in your repository, create a workflow file (e.g.,
`.github/workflows/release.yml`) in your project:

```yaml
name: Release

on:
  push:
    branches: [develop, "feature/*"]

jobs:
  release:
    uses: nextmv-io/release/.github/workflows/release.yml@develop
    with:
      BRANCH: ${{ github.ref_name }}
      REPOSITORY: your-repo-name
      LANGUAGE: python  # or 'go'
      PACKAGE_NAME: your-package-name
      PACKAGE_LOCATION: .  # or path to your package
      VERSION_FILE: __about__.py  # or VERSION for Go
    secrets: inherit
```

### Example: Python Project (nextpipe)

```yaml
jobs:
  release:
    uses: nextmv-io/release/.github/workflows/release.yml@develop
    with:
      BRANCH: ${{ github.ref_name }}
      REPOSITORY: nextpipe
      LANGUAGE: python
      PACKAGE_NAME: nextpipe
      PACKAGE_LOCATION: .
      VERSION_FILE: __about__.py
    secrets: inherit
```

### Example: Go Project

```yaml
jobs:
  release:
    uses: nextmv-io/release/.github/workflows/release.yml@develop
    with:
      BRANCH: ${{ github.ref_name }}
      REPOSITORY: my-go-project
      LANGUAGE: go
      PACKAGE_NAME: my-go-project
      PACKAGE_LOCATION: .
      VERSION_FILE: VERSION
    secrets: inherit
```

## Input Parameters

| Parameter | Required | Description | Example |
|-----------|----------|-------------|---------|
| `BRANCH` | ✅ | The branch that triggered the workflow | `${{ github.ref_name }}` |
| `REPOSITORY` | ✅ | The repository name | `nextpipe` |
| `LANGUAGE` | ✅ | The programming language (`python` or `go`) | `python` |
| `PACKAGE_NAME` | ✅ | The name of the package to release | `nextpipe` |
| `PACKAGE_LOCATION` | ✅ | The location of the package to release | `.` or `packages/core` |
| `VERSION_FILE` | ✅ | The file that contains the version | `__about__.py` or `VERSION` |

## Outputs

The workflow provides the following outputs:

| Output | Description | Example |
|--------|-------------|---------|
| `VERSION` | The version of the package | `v1.2.3` |
| `RELEASE_NEEDED` | Whether a release was created | `true` or `false` |
| `SHOULD_NOTIFY_SLACK` | Whether Slack should be notified | `true` or `false` |

## Required Secrets

The following secrets must be available in your repository (typically inherited
from the organization):

- `SLACK_URL_MISSION_CONTROL`: Slack webhook URL for release notifications
- `NEXTMVBOT_SSH_KEY`: SSH key for the nextmv-bot user
- `NEXTMVBOT_SIGNING_KEY`: GPG signing key for the nextmv-bot user

## Version File Requirements

### Python Projects

For Python projects, the workflow uses `hatch version` to determine the
version. Ensure your `pyproject.toml` is properly configured:

```toml
[project]
dynamic = ["version"]

[tool.hatch.version]
path = "your_package/__about__.py"
```

### Go Projects

For Go projects, create a `VERSION` file in your repository root containing
just the version number:

```
1.2.3
```

## Workflow Behavior

1. **Setup**: Configures git with bot credentials and clones the repository
2. **Language Setup**: Installs language-specific dependencies (Python/Go)
3. **Change Detection**: Uses path filters to detect if the version file
   changed
4. **Version Extraction**: Reads the version from the appropriate file
5. **Release Logic**: Determines if a release is needed based on the rules
   above
6. **Tag & Release**: Creates the git tag and GitHub release if needed
7. **Outputs**: Sets workflow outputs for downstream jobs

## Integration with Other Workflows

You can use the workflow outputs in subsequent jobs:

```yaml
jobs:
  release:
    uses: nextmv-io/release/.github/workflows/release.yml@develop
    with:
      # ... your inputs
    secrets: inherit

  notify:
    needs: release
    if: needs.release.outputs.SHOULD_NOTIFY_SLACK == 'true'
    runs-on: ubuntu-latest
    steps:
      - name: Notify team
        run: echo "Released version ${{ needs.release.outputs.VERSION }}"
```

## Troubleshooting

### Release Not Created

Check that:

- Your version file has actually changed in the commit
- You're not trying to release a stable version from a feature branch
- You're not trying to release a dev version from the main branch
- The version tag doesn't already exist

### Permission Issues

Ensure your repository has the required secrets and the workflow has
appropriate permissions for creating releases and tags.
