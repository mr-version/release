# Complete Version Release Action

A comprehensive composite action that orchestrates the complete release workflow for monorepo projects using Mister.Version.

## Description

This action combines version calculation, tag creation, and report generation into a single, streamlined release workflow. It's designed to be the primary action for most release scenarios, handling everything from version detection to creating release artifacts.

## Usage

```yaml
- uses: mr-version/release@v1
  with:
    create-tags: true
    generate-report: true
```

## Inputs

### Repository Settings

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `repository-path` | Path to the Git repository root | No | `.` |
| `projects` | Glob pattern or comma-separated list of project files to process | No | `**/*.csproj` |

### Version Calculation

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `prerelease-type` | Type of prerelease (alpha, beta, rc, none) | No | `none` |
| `tag-prefix` | Prefix for version tags | No | `v` |
| `force-version` | Force a specific version for all projects | No | - |

### Tag Creation

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `create-tags` | Create git tags for the calculated versions | No | `true` |
| `create-global-tags` | Create global tags for major releases | No | `false` |
| `global-tag-strategy` | Strategy for global tags (major-only, all, none) | No | `major-only` |
| `tag-message-template` | Template for tag messages | No | `Release {type} {version}` |
| `sign-tags` | Sign tags with GPG | No | `false` |

### Report Generation

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `generate-report` | Generate a version report | No | `true` |
| `report-format` | Report output format (text, json, csv, markdown) | No | `markdown` |
| `report-file` | File path to save the report | No | - |
| `post-report-to-pr` | Post the report as a PR comment | No | `false` |

### Filtering Options

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `only-changed` | Only process projects with version changes | No | `true` |
| `include-all-projects` | Include all projects in the report (not just changed ones) | No | `false` |

### Behavior Options

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `dry-run` | Show what would be done without making changes | No | `false` |
| `fail-on-no-changes` | Fail if no version changes are detected | No | `false` |
| `fail-on-existing-tags` | Fail if a tag already exists | No | `false` |

### GitHub Integration

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `token` | GitHub token for PR comments and API access | No | `${{ github.token }}` |

## Outputs

### Version Calculation Outputs

| Output | Description |
|--------|-------------|
| `projects` | JSON array of all analyzed projects with their versions |
| `changed-projects` | JSON array of projects with version changes |
| `has-changes` | Whether any version changes were detected |

### Tagging Outputs

| Output | Description |
|--------|-------------|
| `tags-created` | JSON array of tags that were created |
| `tags-count` | Total number of tags created |

### Report Outputs

| Output | Description |
|--------|-------------|
| `report-content` | The generated report content |
| `report-file` | Path to the generated report file |

## Examples

### Basic Release

```yaml
name: Release
on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - uses: mr-version/release@v1
```

### Prerelease Workflow

```yaml
name: Prerelease
on:
  push:
    branches: [develop, beta]

jobs:
  prerelease:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - uses: mr-version/release@v1
        with:
          prerelease-type: ${{ github.ref == 'refs/heads/beta' && 'beta' || 'alpha' }}
          create-global-tags: false
```

### Manual Release with Approval

```yaml
name: Manual Release
on:
  workflow_dispatch:
    inputs:
      dry-run:
        description: 'Perform a dry run'
        type: boolean
        default: true
      prerelease:
        description: 'Prerelease type'
        type: choice
        options:
          - none
          - alpha
          - beta
          - rc

jobs:
  release:
    runs-on: ubuntu-latest
    environment: ${{ inputs.dry-run && 'staging' || 'production' }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - uses: mr-version/release@v1
        with:
          dry-run: ${{ inputs.dry-run }}
          prerelease-type: ${{ inputs.prerelease }}
          sign-tags: ${{ !inputs.dry-run }}
```

### Pull Request Preview

```yaml
name: PR Release Preview
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - uses: mr-version/release@v1
        with:
          dry-run: true
          post-report-to-pr: true
          comment-header: '## 🚀 Release Preview'
```

### Multi-Stage Release

```yaml
name: Multi-Stage Release
on:
  push:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    outputs:
      has-changes: ${{ steps.release.outputs.has-changes }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - uses: mr-version/release@v1
        id: release
        with:
          dry-run: true
          fail-on-no-changes: false

  release:
    needs: validate
    if: needs.validate.outputs.has-changes == 'true'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - uses: mr-version/release@v1
        with:
          create-tags: true
          sign-tags: true
          report-file: release-notes.md
      
      - uses: actions/upload-artifact@v3
        with:
          name: release-notes
          path: release-notes.md
```

### Conditional Features

```yaml
name: Conditional Release
on:
  push:
    branches: [main, develop]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Determine Settings
        id: settings
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "prerelease=none" >> $GITHUB_OUTPUT
            echo "create-tags=true" >> $GITHUB_OUTPUT
          else
            echo "prerelease=beta" >> $GITHUB_OUTPUT
            echo "create-tags=false" >> $GITHUB_OUTPUT
          fi
      
      - uses: mr-version/release@v1
        with:
          prerelease-type: ${{ steps.settings.outputs.prerelease }}
          create-tags: ${{ steps.settings.outputs.create-tags }}
          generate-report: true
```

## Workflow Components

This composite action internally uses:
1. **Setup** - Installs Mister.Version CLI
2. **Calculate** - Determines version changes
3. **Tag** - Creates git tags (if enabled)
4. **Report** - Generates version reports (if enabled)
5. **Summary** - Creates GitHub Actions summary

## Best Practices

### 1. Always Use Full History
```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0  # Required for version calculation
```

### 2. Use Environments for Production
```yaml
jobs:
  release:
    environment: production
    steps:
      - uses: mr-version/release@v1
```

### 3. Test with Dry Run
```yaml
- uses: mr-version/release@v1
  with:
    dry-run: true
```

### 4. Sign Production Tags
```yaml
- uses: mr-version/release@v1
  with:
    sign-tags: true
```

### 5. Generate Release Artifacts
```yaml
- uses: mr-version/release@v1
  with:
    report-file: CHANGELOG.md
    report-format: markdown
```

## Troubleshooting

### No Changes Detected
- Ensure conventional commit format is used
- Check `fetch-depth: 0` is set
- Verify project files are tracked

### Tag Creation Fails
- Check for existing tags with `fail-on-existing-tags: false`
- Ensure push permissions for tags
- Verify GPG configuration if signing

### Report Not Generated
- Check `generate-report: true` is set
- Verify output format is valid
- Check file permissions for report-file

## GitHub Actions Summary

The action automatically creates a summary in the GitHub Actions UI showing:
- Number of projects analyzed
- Number of projects with changes
- Number of tags created
- Report generation status
- Dry run indicator

## License

This action is part of the Mister.Version project and is licensed under the MIT License.