---
title: "Building a Modern Development Platform: Versioning Your Azure DevOps Pipeline 🏷️"
date: 2025-11-11T12:00:00-07:00
draft: false
categories: ["platform","azure-devops","versioning","cicd","automation"]
description: "Automating semantic versioning in Azure DevOps with automated tagging and branch management"
---

[Series Posts](https://brianpsheridan.com/categories.html#platform)

> 💻 **Source Code:** The complete versioning pipeline and configuration files are available in the [`aspire-tools-azure-devops`](https://github.com/two4suited/blog-platform-aspire/tree/aspire-tools-azure-devops) branch of our GitHub repository.

## The Problem: Breaking Changes in Shared Templates 💥

Remember our [pipeline templates post](https://brianpsheridan.com/categories.html#platform)? Teams across your organization reference your template repository to keep their pipelines consistent and DRY.

But there's a critical issue:

**Most teams point to the `main` branch:**
```yaml
resources:
  repositories:
    - repository: templates
      type: git
      name: YourProject/pipeline-templates
      ref: refs/heads/main  # ⚠️ Always pulls latest
```

This works great until you make a breaking change to your templates:
- ❌ You rename a parameter: `buildConfiguration` → `configuration`
- ❌ You restructure stages: 3 stages becomes 2 stages
- ❌ You change variable names: `appName` → `projectName`
- ❌ **Everyone's pipelines break immediately**

Suddenly your entire organization is blocked, unable to deploy. Teams are scrambling to fix their pipelines. Production deployments are halted. All because you tried to improve your templates.

## The Solution: Versioned Templates 🎯

With versioning, teams pin to specific versions:

```yaml
resources:
  repositories:
    - repository: templates
      type: git
      name: YourProject/pipeline-templates
      ref: refs/tags/v1.2.0  # ✅ Pinned to specific version
```

Now you can safely make breaking changes in a new major version:
- ✅ v1.2.0 continues to work for existing teams (old parameter names, old structure)
- ✅ v2.0.0 introduces breaking changes (new parameter names, new structure)
- ✅ Teams upgrade on their schedule, at their pace
- ✅ Your pipelines remain stable and predictable

## Introduction 🚀

An **automated versioning pipeline** solves this by creating a disciplined release process for your templates:
- ✅ Generates semantic version numbers (MAJOR.MINOR.PATCH)
- 🏷️ Creates Git tags automatically
- 🌿 Maintains release version branches
- 📊 Provides version information to downstream pipelines
- 🔄 Prevents duplicate versions
- 🔐 Enables safe, controlled template changes

## Why Automated Versioning? 🤔

### The Breaking Change Scenario ❌

Imagine you share pipeline templates across 50 teams in your organization:

```yaml
# Current template: stages/build.yml
parameters:
  - name: buildConfiguration
    type: string
    default: 'Release'
```

You want to improve naming, so you rename the parameter:

```yaml
# Updated template: stages/build.yml
parameters:
  - name: configuration      # ← Changed from buildConfiguration
    type: string
    default: 'Release'
```

**What happens at 3 AM:**
- 🚨 All 50 teams' pipelines start failing
- 😱 Teams can't deploy because their pipelines reference `buildConfiguration`
- 📞 Your team gets paged frantically
- ⏹️ Production deployments are blocked
- 😤 Teams lose trust in shared templates

### The Versioning Advantage ✅

With versioning, you release breaking changes in a new major version:

**v1.2.0** (current - stable for everyone)
```yaml
parameters:
  - name: buildConfiguration  # Original parameter name
    type: string
```

**v2.0.0** (new - breaking changes allowed)
```yaml
parameters:
  - name: configuration       # Improved parameter name
    type: string
```

Now:
- ✅ Teams on v1.2.0 continue working without any changes
- ✅ New teams can adopt v2.0.0 with modern parameter names
- ✅ Teams upgrade at their own pace
- ✅ No surprises, no emergency calls at 3 AM
- ✅ Version history is clear: what changed, when, and why

### Manual Versioning Problems ❌

Without automation:
- 🐛 Human errors lead to duplicate versions
- 📉 Inconsistent version numbering across teams
- 🤷 Unclear when versions were created or who released them
- 🔄 Difficult to track which template version each project uses
- ⚠️ Hard to communicate breaking changes to teams

### Automated Versioning Benefits ✅

An automated pipeline solves these problems:
- ✨ **Consistency**: Same versioning strategy everywhere
- 🔒 **Traceability**: Git tags mark exactly what's in each version
- 🔄 **Repeatability**: Runs the same way every time
- 📊 **Transparency**: Entire team sees version history
- ⚡ **Efficiency**: No manual tagging or branch creation
- 🛡️ **Safety**: Prevents duplicate versions and accidental overwrites

## The Versioning Strategy 🎯

### Semantic Versioning

We use **Semantic Versioning** (SemVer) format: `MAJOR.MINOR.PATCH`

```
v2.0.0
│ │ └─ PATCH: Bug fixes, internal improvements (no new features)
│ └───── MINOR: New features, backward compatible
└─────── MAJOR: Breaking changes, significant changes
```

**Examples:**
```
v1.0.0  → Initial release
v1.1.0  → New feature added (backward compatible)
v1.1.1  → Bug fix
v2.0.0  → Breaking change, new major release
```

### Components

Our versioning system consists of:
- **Major version**: Major release number
- **Minor version**: Feature releases
- **Patch version**: Bug fix releases
- **Pre-release identifiers**: `-alpha`, `-beta`, `-rc1` for testing releases

## Using Version Numbers in Other Pipelines 🔗

### Reference Pipeline Templates with Version Branch or Tag

When teams reference your shared pipeline templates, they have two safe options:

#### Option 1: Reference the Release Branch (Recommended)

Use the `releases/vX.latest` branch to get bug fixes automatically:

```yaml
resources:
  repositories:
    - repository: templates
      type: git
      name: YourProject/pipeline-templates
      ref: refs/heads/releases/v1.latest  # ✅ Always v1.x.x with latest patches

trigger:
  - main

extends:
  template: stages/build.yml@templates
  parameters:
    appName: 'WeatherApp.Api'
    buildConfiguration: 'Release'
    projectPath: 'src/WeatherApp.Api'
```

**Benefits:**
- 🔄 Automatically gets bug fixes (v1.0.1, v1.0.2, v1.1.0)
- 🛡️ Protected from breaking changes (v2.0.0 won't affect you)
- ⚡ No manual version updates needed

#### Option 2: Reference a Specific Tag

Pin to an exact version for complete control:

```yaml
resources:
  repositories:
    - repository: templates
      type: git
      name: YourProject/pipeline-templates
      ref: refs/tags/v1.0.0  # ✅ Frozen at exactly v1.0.0

trigger:
  - main

extends:
  template: stages/build.yml@templates
  parameters:
    appName: 'WeatherApp.Api'
    buildConfiguration: 'Release'
    projectPath: 'src/WeatherApp.Api'
```

**Benefits:**
- 🔒 Completely frozen behavior - templates never change
- 📋 Predictable for compliance/audit scenarios
- ⚠️ Must manually update tags to get bug fixes

**Comparison:**

| Approach | Benefit | Trade-off |
|----------|---------|-----------|
| `refs/heads/releases/v1.latest` | Get bug fixes (v1.0.1, v1.1.0) automatically | Must trust template maintainers for PATCH/MINOR updates |
| `refs/tags/v1.0.0` | Complete control, frozen behavior | Must manually update to get bug fixes |
| `refs/heads/main` | Always latest features | ❌ Breaking changes could break you overnight |

**Recommendation:** Use `releases/vX.latest` for your major version. You get stability (no breaking changes) plus automatic bug fixes. Only use specific tags if you need complete control for compliance reasons.

## Pipeline Architecture 🏗️

The versioning pipeline has two core files:

### 1. versioning.yaml - Configuration

This file holds the version number and is updated when you want to release a new version.

### 2. versioning-pipeline.yaml - Automation

This pipeline reads the version from `versioning.yaml`, creates Git tags, and manages release branches.

## Setting Up the Versioning Pipeline 📦

### Step 1: Create the Versioning Folder

Create a `versioning/` directory in your repository root:

```bash
mkdir -p versioning
cd versioning
```

### Step 2: Create versioning.yaml

Create `versioning/versioning.yaml` with your initial version:

```yaml
variables:
  # Version Components
  major: 1
  minor: 0
  revision: 0
  
  # Pre-release version support
  preRelease: ''  # Set to 'alpha', 'beta', 'rc1', etc. for pre-release versions or leave empty for stable
  
  # Composite Version Numbers
  version: '$(major).$(minor).$(revision)'
  
  # Prerelease version uses same base version as stable
  prereleaseVersion: '$(major).$(minor).$(revision)-$(preRelease)'
```

**Version Components Explained:**

| Component | Purpose | Example |
|-----------|---------|---------|
| `major` | Major version number | `1` in v1.0.0 |
| `minor` | Minor version number | `0` in v1.0.0 |
| `revision` | Patch version number | `0` in v1.0.0 |
| `preRelease` | Pre-release suffix | `alpha`, `beta`, `rc1`, or empty for stable |
| `version` | Full semantic version | `1.0.0` |
| `prereleaseVersion` | Version with pre-release suffix | `1.0.0-alpha` or `1.0.0` if stable |

### Step 3: Create versioning-pipeline.yaml

Create `versioning-pipeline.yaml` at the repository root:

```yaml
trigger:
  branches:
    include:
    - main
  paths:
    include:
    - versioning/versioning.yaml

variables:
- template: versioning/versioning.yaml

pool:
  vmImage: 'ubuntu-latest'

steps:
- checkout: self
  persistCredentials: true
  clean: true

- task: Bash@3
  displayName: 'Setup Git Configuration'
  inputs:
    targetType: 'inline'
    script: |
      git config user.name "Azure DevOps Pipeline"
      git config user.email "noreply@azuredevops.com"
      
- task: Bash@3
  displayName: 'Prepare Version Variables'
  inputs:
    targetType: 'inline'
    script: |
      echo "Current version: $(version)"
      echo "Prerelease version: $(prereleaseVersion)"
      echo "Major version: $(major)"
      echo "Minor version: $(minor)"
      echo "Revision: $(revision)"
      echo "Pre-release: '$(preRelease)'"
      
      # Determine branch suffix and version tag based on prerelease
      if [ -z "$(preRelease)" ]; then
        BRANCH_SUFFIX="latest"
        VERSION_TAG="v$(version)"
      else
        BRANCH_SUFFIX="$(preRelease)"
        VERSION_TAG="v$(prereleaseVersion)"
      fi
      
      MAJOR_BRANCH="releases/v$(major).$BRANCH_SUFFIX"
      
      echo "Version tag: $VERSION_TAG"
      echo "Major branch: $MAJOR_BRANCH"
      
      # Set variables for subsequent steps
      echo "##vso[task.setvariable variable=versionTag]$VERSION_TAG"
      echo "##vso[task.setvariable variable=majorBranch]$MAJOR_BRANCH"
      echo "##vso[task.setvariable variable=branchSuffix]$BRANCH_SUFFIX"

- task: Bash@3
  displayName: 'Check Versioning Requirements'
  inputs:
    targetType: 'inline'
    script: |
      VERSION_TAG="$(versionTag)"
      echo "Target version: $VERSION_TAG"
      
      if [ -z "$(preRelease)" ]; then
        # For stable releases, check if tag exists
        echo "Checking stable release tag: $VERSION_TAG"
        if git tag -l | grep -q "^$VERSION_TAG$"; then
          echo "##vso[task.logissue type=warning]Stable release tag $VERSION_TAG already exists! Skipping versioning."
          echo "##vso[task.setvariable variable=shouldSkip]true"
          echo "##vso[task.setvariable variable=skipReason]Stable tag $VERSION_TAG already exists"
        else
          echo "Tag $VERSION_TAG does not exist, proceeding with stable release..."
          echo "##vso[task.setvariable variable=shouldSkip]false"
        fi
      else
        # For prereleases, skip tag creation entirely
        echo "Prerelease detected: $(preRelease)"
        echo "Skipping tag creation for prerelease builds..."
        echo "##vso[task.setvariable variable=shouldSkip]false"
      fi

- task: Bash@3
  displayName: 'Delete Existing Major Version Branch'
  condition: and(succeeded(), ne(variables['shouldSkip'], 'true'))
  inputs:
    targetType: 'inline'
    script: |
      MAJOR_BRANCH="$(majorBranch)"
      
      echo "Target branch to recreate: $MAJOR_BRANCH"
      
      # Check if branch exists locally
      if git branch --list | grep -q "$MAJOR_BRANCH"; then
        echo "Deleting local branch: $MAJOR_BRANCH"
        git branch -D "$MAJOR_BRANCH"
      fi
      
      # Check if branch exists on remote
      if git ls-remote --heads origin "$MAJOR_BRANCH" | grep -q "$MAJOR_BRANCH"; then
        echo "Deleting remote branch: $MAJOR_BRANCH"
        git push origin --delete "$MAJOR_BRANCH"
      else
        echo "Remote branch $MAJOR_BRANCH does not exist"
      fi

- task: Bash@3
  displayName: 'Set Build Number'
  condition: and(succeeded(), ne(variables['shouldSkip'], 'true'))
  inputs:
    targetType: 'inline'
    script: |
      VERSION_TAG="$(versionTag)"
      
      # Remove 'v' prefix from tag for build number
      BUILD_NUMBER_VALUE=$(echo "$VERSION_TAG" | sed 's/^v//')
      
      echo "Setting build number to: $BUILD_NUMBER_VALUE"
      echo "##vso[build.updatebuildnumber]$BUILD_NUMBER_VALUE"

- task: Bash@3
  displayName: 'Create Version Tag'
  condition: and(succeeded(), ne(variables['shouldSkip'], 'true'), eq(variables['preRelease'], ''))
  inputs:
    targetType: 'inline'
    script: |
      VERSION_TAG="$(versionTag)"
      
      echo "Creating stable release tag: $VERSION_TAG"
      
      TAG_MESSAGE="Version $(version) - Created by Azure DevOps Pipeline"
      
      git tag -a "$VERSION_TAG" -m "$TAG_MESSAGE"
      
      echo "Pushing tag to remote..."
      git push origin "$VERSION_TAG"
      
      echo "✓ Successfully created and pushed tag: $VERSION_TAG"

- task: Bash@3
  displayName: 'Create Major Version Branch'
  condition: and(succeeded(), ne(variables['shouldSkip'], 'true'))
  inputs:
    targetType: 'inline'
    script: |
      MAJOR_BRANCH="$(majorBranch)"
      BRANCH_TYPE=$(echo "$(branchSuffix)")
      
      if [ "$BRANCH_TYPE" = "latest" ]; then
        echo "Creating stable release branch: $MAJOR_BRANCH"
      else
        echo "Creating prerelease branch: $MAJOR_BRANCH (prerelease: $BRANCH_TYPE)"
      fi
      
      git checkout -b "$MAJOR_BRANCH"
      
      echo "Pushing branch to remote..."
      git push -u origin "$MAJOR_BRANCH"
      
      echo "✓ Successfully created and pushed branch: $MAJOR_BRANCH"
      
      # Switch back to source branch
      echo "Current branch before checkout: $(git branch --show-current)"
      echo "Target branch: $(Build.SourceBranchName)"
      
      if git show-ref --verify --quiet refs/heads/$(Build.SourceBranchName); then
        echo "Switching back to branch: $(Build.SourceBranchName)"
        git checkout $(Build.SourceBranchName)
      else
        echo "Branch $(Build.SourceBranchName) doesn't exist locally, staying on current branch"
        echo "Available local branches:"
        git branch --list
      fi

- task: Bash@3
  displayName: 'Version Summary'
  inputs:
    targetType: 'inline'
    script: |
      echo "=================================="
      echo "         VERSION SUMMARY          "
      echo "=================================="
      
      if [ "$(shouldSkip)" = "true" ]; then
        echo "⚠️  VERSIONING SKIPPED"
        echo "Reason: $(skipReason)"
        echo "Target Version: $(versionTag)"
        echo "Target Branch: $(majorBranch)"
      else
        echo "✅ VERSIONING COMPLETED"
        echo "Created Branch: $(majorBranch)"
        if [ "$(branchSuffix)" != "latest" ]; then
          echo "Type: Pre-release ($(branchSuffix)) - No tag created"
        else
          echo "Type: Stable release"
          echo "Created Tag: $(versionTag)"
        fi
      fi
      
      echo ""
      echo "Build Information:"
      echo "Source Branch: $(Build.SourceBranchName)"
      echo "Build Number: $(Build.BuildNumber)"
      if [ -z "$(preRelease)" ]; then
        echo "Original Version: v$(version)"
        echo "Prerelease Version: N/A"
      else
        echo "Original Version: v$(version)"  
        echo "Prerelease Version: v$(prereleaseVersion)"
      fi
      echo "Final Version Used: $(versionTag)"
      echo "=================================="
      
      echo "Available tags:"
      git tag --sort=-version:refname | head -10
      
      echo ""
      echo "Available branches matching releases/v*:"
      git branch -r | grep "releases/v" | head -10 || echo "No release version branches found"
```

## Understanding the Pipeline Steps 🔍

### Step 1: Git Configuration ⚙️

```bash
git config user.name "Azure DevOps Pipeline"
git config user.email "noreply@azuredevops.com"
```

Configures Git so the pipeline can commit and tag with proper identity.

### Step 2: Prepare Version Variables 📝

```bash
# Determine if this is a stable or prerelease version
if [ -z "$(preRelease)" ]; then
  BRANCH_SUFFIX="latest"
  VERSION_TAG="v$(version)"
else
  BRANCH_SUFFIX="$(preRelease)"
  VERSION_TAG="v$(prereleaseVersion)"
fi

MAJOR_BRANCH="releases/v$(major).$BRANCH_SUFFIX"
```

Converts version numbers and pre-release status into variables for subsequent steps:
- 🏷️ **Stable releases** (empty preRelease): Create `releases/v1.latest` branch and `v1.0.0` tag
- 🧪 **Pre-releases** (alpha/beta): Create `releases/v1.alpha` branch, no tag created
- 📦 Sets `versionTag`, `majorBranch`, and `branchSuffix` for other steps to use

### Step 3: Check Versioning Requirements ✅

```bash
if [ -z "$(preRelease)" ]; then
  # For stable releases, check if tag exists
  if git tag -l | grep -q "^$VERSION_TAG$"; then
    # Skip to prevent duplicate tags
    shouldSkip=true
  fi
else
  # For prereleases, always proceed (no tag check needed)
  shouldSkip=false
fi
```

**Stable releases** only: Prevents creating duplicate tags. If version already exists, skips remaining steps.

**Pre-releases**: Always proceeds (doesn't create tags, so no duplicate risk).

### Step 4: Delete Existing Major Version Branch 🧹

```bash
git branch -D "$MAJOR_BRANCH"
git push origin --delete "$MAJOR_BRANCH"
```

Deletes the old `releases/v1.latest` or `releases/v1.alpha` branch so we can create a fresh one pointing to the current commit.

**Why?** These branches always point to the latest release of their type. When you release v1.1.0, the `releases/v1.latest` branch moves to this new commit.

### Step 5: Set Build Number 📊

```bash
BUILD_NUMBER_VALUE=$(echo "$VERSION_TAG" | sed 's/^v//')
echo "##vso[build.updatebuildnumber]$BUILD_NUMBER_VALUE"
```

Updates the Azure DevOps build number to match your version (e.g., "1.0.0" instead of "12345").

### Step 6: Create Version Tag 🏷️

```bash
# Only for stable releases (preRelease is empty)
git tag -a "$VERSION_TAG" -m "Version $(version)"
git push origin "$VERSION_TAG"
```

**Stable releases only**: Creates a Git tag marking this exact point in code.

**Pre-releases**: Skipped (set by condition: `eq(variables['preRelease'], '')`)

### Step 7: Create Major Version Branch 🌿

```bash
git checkout -b "$MAJOR_BRANCH"
git push -u origin "$MAJOR_BRANCH"
```

Creates a branch for this release series:
- **Stable**: `releases/v1.latest` - tracks all v1.x patch releases
- **Pre-release**: `releases/v1.alpha` - tracks pre-release builds

### Step 8: Version Summary 📊

Displays what was created, distinguishing between stable and pre-release builds.

## Workflow: How to Release a New Version 📋

### Releasing v1.0.0 → v1.1.0 (Minor Release)

**1. Update versioning.yaml:**
```yaml
variables:
  major: 1
  minor: 1          # ← Incremented
  revision: 0       # ← Reset to 0
```

**2. Commit and push:**
```bash
git add versioning/versioning.yaml
git commit -m "chore: bump version to 1.1.0"
git push origin main
```

**3. Pipeline automatically:**
- ✅ Detects the change to `versioning/versioning.yaml`
- ✅ Creates tag `v1.1.0`
- ✅ Creates/updates branch `releases/v1.latest`
- ✅ Publishes results

### Releasing v1.0.0 → v1.0.1 (Patch Release)

**1. Update versioning.yaml:**
```yaml
variables:
  major: 1
  minor: 0
  revision: 1       # ← Incremented
```

**2. Commit and push:**
```bash
git add versioning/versioning.yaml
git commit -m "chore: bump version to 1.0.1"
git push origin main
```

### Releasing v1.0.0 → v2.0.0 (Major Release)

**1. Update versioning.yaml:**
```yaml
variables:
  major: 2          # ← Incremented
  minor: 0          # ← Reset to 0
  revision: 0       # ← Reset to 0
```

**2. Commit and push:**
```bash
git add versioning/versioning.yaml
git commit -m "chore: bump version to 2.0.0 (breaking changes)"
git push origin main
```

## Using Version Numbers in Other Pipelines 🔗

### Reference Pipeline Templates with Version Branch or Tag

When teams reference your shared pipeline templates, they have two safe options:

#### Option 1: Reference the Release Branch (Recommended)

Use the `releases/vX.latest` branch to get bug fixes automatically:

```yaml
resources:
  repositories:
    - repository: templates
      type: git
      name: YourProject/pipeline-templates
      ref: refs/heads/releases/v1.latest  # ✅ Always v1.x.x with latest patches

trigger:
  - main

extends:
  template: stages/build.yml@templates
  parameters:
    appName: 'WeatherApp.Api'
    buildConfiguration: 'Release'
    projectPath: 'src/WeatherApp.Api'
```

**Benefits:**
- 🔄 Automatically gets bug fixes (v1.0.1, v1.0.2, v1.1.0)
- 🛡️ Protected from breaking changes (v2.0.0 won't affect you)
- ⚡ No manual version updates needed

#### Option 2: Reference a Specific Tag

Pin to an exact version for complete control:

```yaml
resources:
  repositories:
    - repository: templates
      type: git
      name: YourProject/pipeline-templates
      ref: refs/tags/v1.0.0  # ✅ Frozen at exactly v1.0.0

trigger:
  - main

extends:
  template: stages/build.yml@templates
  parameters:
    appName: 'WeatherApp.Api'
    buildConfiguration: 'Release'
    projectPath: 'src/WeatherApp.Api'
```

**Benefits:**
- 🔒 Completely frozen behavior - templates never change
- 📋 Predictable for compliance/audit scenarios
- ⚠️ Must manually update tags to get bug fixes

**Comparison:**

| Approach | Benefit | Trade-off |
|----------|---------|-----------|
| `refs/heads/releases/v1.latest` | Get bug fixes (v1.0.1, v1.1.0) automatically | Must trust template maintainers for PATCH/MINOR updates |
| `refs/tags/v1.0.0` | Complete control, frozen behavior | Must manually update to get bug fixes |
| `refs/heads/main` | Always latest features | ❌ Breaking changes could break you overnight |

**Recommendation:** Use `releases/vX.latest` for your major version. You get stability (no breaking changes) plus automatic bug fixes. Only use specific tags if you need complete control for compliance reasons.

## Pre-Release Versions 🧪

## Pre-Release Versions 🧪

For testing releases before going to production, use pre-release identifiers:

```yaml
variables:
  major: 1
  minor: 0
  revision: 0
  preRelease: 'alpha'  # ← Set for testing
  prereleaseVersion: '$(major).$(minor).$(revision)-$(preRelease)'
```

This creates version `1.0.0-alpha`:
- ✅ Pre-release tags like `v1.0.0-alpha`, `v1.0.0-beta`, `v1.0.0-rc1`
- ✅ Final release uses empty `preRelease` value
- ✅ Pre-release packages typically get lower priority in feeds

**Workflow:**
```
1.0.0-alpha    → Testing with early adopters
1.0.0-beta     → Feature complete, bug fixing
1.0.0-rc1      → Release candidate
1.0.0          → Final production release
```

## Best Practices 🎯

### 1. **Communicate Breaking Changes** 📢

When releasing a major version, document exactly what changed:

```markdown
## v2.0.0 - Breaking Changes ⚠️

### Changed Parameters
- `buildConfiguration` → `configuration`
- `appPath` → `projectPath`
- `dockerRegistry` → `registryName`

### Removed Parameters
- `legacyBuildFormat` (use standardized format instead)
- `customScriptPath` (merged into build template)

### New Features
- Added support for multi-configuration builds
- Added native TypeScript build support

### Migration Guide
Update your azure-pipelines.yml:
```yaml
# OLD (v1.x)
ref: refs/heads/releases/v1.latest

# NEW (v2.0+)
ref: refs/heads/releases/v2.latest
```

Also update your parameter references:
```yaml
# OLD
buildConfiguration: Release

# NEW
configuration: Release
```


### 2. **Provide Migration Path** 🛤️

Give teams clear steps to upgrade:

```markdown
## Upgrading from v1.2.0 to v2.0.0

1. Review breaking changes above
2. Update your vars.yml with new parameter names
3. Update resource reference: `ref: refs/tags/v2.0.0`
4. Run a test pipeline build
5. If successful, merge to main

Estimated time: 15 minutes
```

### 3. **Consider Deprecation Periods** ⏳

For critical parameters, support both old and new names during a transition:

```yaml
# Accept both old and new parameter names
parameters:
  - name: configuration
    type: string
  - name: buildConfiguration  # Deprecated alias
    type: string
    default: ''

# In your steps, use whichever is provided
- script: |
    if [ -z "${{ parameters.buildConfiguration }}" ]; then
      CONFIG="${{ parameters.configuration }}"
    else
      echo "⚠️ Warning: buildConfiguration is deprecated, use configuration instead"
      CONFIG="${{ parameters.buildConfiguration }}"
    fi
```

This gives teams time to migrate without immediate breakage.

### 4. **Publish Release Notes** 📰

Always publish release notes with each version:

```
📌 Pipeline Templates v2.0.0 Released

🎉 New Features:
- Multi-configuration build support
- Native TypeScript compilation

⚠️ Breaking Changes:
- Parameter names changed (see migration guide)
- Removed legacy script support

📖 Migration Guide: [docs/upgrade-v1-to-v2.md](docs/upgrade-v1-to-v2.md)

💬 Questions? Post in #devops-templates Slack channel
```

### 5. **Notify Teams of Updates** 🔔

Proactively communicate version releases to teams:

- 📧 Send email with migration guide for breaking changes
- 💬 Post in team Slack channels
- 📊 Create metrics dashboard showing which teams use which versions
- 🎯 Set upgrade deadlines for unsupported versions

### 6. **Commit Messages Should Reference Versions** 📝

```bash
git commit -m "chore: bump version to 2.1.0 - new feature: X, bugfix: Y"
```

Makes it easy to understand what changed in each version.

### 7. **Create Release Notes** 📄

When creating a version, document what changed:

```markdown
## v2.1.0

### Features ✨
- New dashboard visualizations
- Export to CSV functionality

### Bug Fixes 🐛
- Fixed timeout in background jobs
- Corrected calculation in reporting module

### Breaking Changes ⚠️
None - all changes are backward compatible

### Upgrade Path
No action needed. Existing projects can upgrade whenever convenient.
```

### 8. **Use Protected Branches** 🔒

Protect the `main` branch to prevent accidental version bumps:
- Require pull requests
- Require approvals
- Run all tests before merging

### 9. **Tag After Testing** ✅

Consider this workflow:
1. Develop features on feature branches
2. Merge to `main` with pre-release version (e.g., `2.1.0-rc1`)
3. Run full testing pipeline across multiple sample projects
4. When tests pass, bump to final version (e.g., `2.1.0`)
5. Pipeline creates the official tag
6. Notify teams of new version availability

### 10. **Automate Version Bumping** 🔄

For advanced scenarios, consider automating version bumping based on commit messages:
- Commit with `[major]` tag → bumps MAJOR version
- Commit with `[minor]` tag → bumps MINOR version
- Commit with `[patch]` tag → bumps PATCH version

This reduces manual steps and improves consistency.

## Troubleshooting 🔧

### Issue: "Tag v2.0.0 already exists"

**Solution:** This is intentional! The pipeline prevents duplicate versions. Options:
1. Bump the version in `versioning.yaml` and try again
2. Delete the tag: `git tag -d v2.0.0 && git push origin :refs/tags/v2.0.0`

### Issue: "Permission denied" when pushing tag

**Solution:** Ensure the pipeline has permission to push:
- In Azure DevOps project settings → Pipelines → Settings
- Enable "Make secrets available to builds of forks"
- Grant the pipeline identity push permissions to the repository

### Issue: Version variables not available in other pipelines

**Solution:** Make sure other pipelines reference the versioning template:

```yaml
variables:
  - template: versioning/versioning.yaml
```

This must be at the pipeline root, not in stages.

## Conclusion 🎉

An automated versioning pipeline is a cornerstone of a mature platform. It ensures:

- ✅ **Consistency**: Same versioning everywhere
- 📊 **Traceability**: Git tags mark exact code for each version
- 🔒 **Safety**: Prevents duplicate versions and accidental overwrites
- ⚡ **Efficiency**: No manual tagging or branch management
- 🤝 **Transparency**: Entire team sees version history

By implementing this versioning pipeline, your team gains confidence in releases and can easily roll back to any previous version if needed.

**Key Takeaways:**
1. Store version numbers in a single `versioning.yaml` file
2. Use semantic versioning: MAJOR.MINOR.PATCH
3. Automate tag creation and release branch management
4. Reference version variables in all build pipelines
5. Use pre-release identifiers for testing releases

Start versioning your platform today!

---

**Resources:** 📚

- [Semantic Versioning](https://semver.org/)
- [Azure Pipelines Variables](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/variables)
- [Git Tagging](https://git-scm.com/book/en/v2/Git-Basics-Tagging)
- [Our GitHub Repository](https://github.com/two4suited/blog-platform-aspire)
