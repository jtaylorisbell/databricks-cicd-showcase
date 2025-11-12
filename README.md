# Databricks Asset Bundles with Multiple Branching Strategies
This is an extension of two great resources available on this topic: [Databrick's bundle examples repo](https://github.com/databricks/bundle-examples) and [@ajalisatgi's dabs-gitflow repo](https://github.com/ajalisatgi/dabs-gitflow/tree/main)

## Overview

This project provides Databricks deployment workflows for two branching strategies:

1. **GitFlow** - A structured branching model with separate develop, release, and hotfix branches
2. **Trunk-Based Development** - A streamlined approach with continuous integration on the main branch

Databricks Asset Bundles (DABs) package the relevant Databricks assets together and deploy them across different workspaces. GitHub Actions workflows automate the deployment processes for both branching strategies.

---

## Choosing a Branching Strategy

### GitFlow
Best for teams that need:
- Formal release cycles with explicit version control
- Separate staging areas for features before production
- Support for hotfixes independent of ongoing development

### Trunk-Based Development
Best for teams that prefer:
- Continuous integration and deployment
- Rapid iteration with frequent small changes
- Simplified branching model with fewer merge conflicts
- Feature flags to manage incomplete features

    
## Prerequisites
Following are the pre-requisites that are needed to use the code in this repository to manage deployments for your specific Databricks workspaces.
* Clone this repository
* **Install [uv](https://docs.astral.sh/uv/)** - This project uses uv for Python dependency management (required)
* Provision 3 distinct Databricks workspaces: `dev`, `staging` & `prod`
* Create service principal(s):
  * Whether you use a single SP or one SP per environment is up to you. Just ensure each SP is correctly assigned to the workspaces it is targeting.
  * Create an Oauth secret for your SP(s) and make a note of the the `Client ID` and `Secret`  
* Create GitHub environments for `dev`, `staging` and `prod`
* Create the following secrets in your GitHub Environments:
  * `DATABRICKS_CLIENT_ID`
  * `DATABRICKS_CLIENT_SECRET`
* Create `DATABRICKS_HOST` as a variable in your GitHub Environments
  
---

## Dev Dependency Management

This project requires **[uv](https://docs.astral.sh/uv/)** for Python dependency management. All dependencies are defined in `pyproject.toml`.

## Running Tests and Deploying to Dev
To run tests locally, it is recommended that you use VSCode as your IDE. This is because the Databricks VSCode extension and Databricks Connect make it very easy to connect to your interactive cluster, run tests, and debug using the built-in VSCode Python debugger. Please reference the [documentation](https://docs.databricks.com/en/dev-tools/vscode-ext/index.html) to get started. The steps to debug are below:

1. Install [uv](https://docs.astral.sh/uv/getting-started/installation/) if you haven't already:
   ```bash
   curl -LsSf https://astral.sh/uv/install.sh | sh
   ```

2. Create and activate a Python environment (uv will automatically use the version specified in `.python-version`):
   ```bash
   uv venv
   source .venv/bin/activate  # On Windows: .venv\Scripts\activate
   ```

3. Install development dependencies:
   ```bash
   uv sync
   ```

4. Set `DATABRICKS_HOST` and `DATABRICKS_CLUSTER_ID` environment variables. You will also need `DATABRICKS_TOKEN` if you are using PAT authentication.
> [!TIP]
> If using OAuth, you'll need to run `databricks auth login --host $DATABRICKS_HOST`

5. Run tests:
   ```bash
   uv run pytest
   ```
6. (OPTIONAL) Deploy to dev:
   ```bash
   uv run databricks bundle deploy
   ```

# GitFlow Branching Strategy

## Standard Release Process
The following image represents the steps involved in deploying a Standard Release
![image](https://github.com/user-attachments/assets/e50ef525-60f4-4680-abb2-38ec7ea90e89)


---
## Hotfix Release Process
The following image represents the steps involved in deploying a Hotfix Release
![image](https://github.com/user-attachments/assets/73cbdd53-84b8-43ae-bfb5-2b944a3c7e65)

---

## Best Practices for GitFlow Development

1. **Feature Development**: All new features should be developed in feature branches created from `develop`
   ```bash
   git checkout develop
   git pull origin develop
   git checkout -b feature/my-feature
   ```

2. **Regular Integration**: Merge `develop` into your feature branch regularly to avoid merge conflicts
   ```bash
   git checkout feature/my-feature
   git merge develop
   ```

3. **Pull Requests**: Create PRs from feature branches to `develop` for code review
   ```bash
   git push origin feature/my-feature
   gh pr create --base develop --title "Add new feature"
   ```

4. **Release Planning**: Use the "Draft new release" workflow when ready to prepare a release
   - Choose MAJOR for breaking changes (e.g., 1.0.0 → 2.0.0)
   - Choose MINOR for new features (e.g., 1.0.0 → 1.1.0)
   - Release branches are created automatically (e.g., `release/1.1.0`)

5. **Hotfix Process**: For urgent production fixes:
   - Create a PR with the fix targeting `develop`
   - Use the "Draft new hotfix" workflow, providing the PR number
   - The workflow will create a versioned `hotfix/x.x.x` branch and close the original PR
   - Merge the hotfix PR to `main` to deploy to production

6. **Branch Protection**:
   - Protect `main` and `develop` branches requiring PR reviews
   - Ensure all tests pass before merging
   - Require linear history or squash merging for cleaner history

7. **Testing Strategy**:
   - Test features locally before creating PR to `develop`
   - Staging deployment validates changes before production
   - All tests must pass in staging before releasing to prod

8. **Version Management**:
   - Versions are automatically incremented based on release type
   - Tags are automatically created when releases are published
   - Keep release notes updated in GitHub releases

---

# Trunk-Based Development Strategy

Trunk-Based Development uses the `main` branch as the single source of truth. All developers commit frequently to main, enabling continuous integration and rapid deployment.

## Workflow Overview

```
main branch (trunk)
    │
    ├─> CI runs on every push
    │   ├─> Tests
    │   ├─> Bundle validation
    │   └─> Auto-deploy to dev
    │
    ├─> Manual/Scheduled deploy to staging
    │   ├─> Tests on staging
    │   └─> Deploy to staging workspace
    │
    └─> Manual production release
        ├─> Tests
        ├─> Deploy to prod workspace
        └─> Create git tag and GitHub release
```

## Development Workflow

### 1. Feature Development
Developers have two options:

**Option A: Direct commits to main (recommended for small changes)**
```bash
# Make changes
git checkout main
git pull origin main
# Make your changes
git add .
git commit -m "feat: add new feature"
git push origin main
```

**Option B: Short-lived feature branches (for larger changes)**
```bash
# Create short-lived branch
git checkout -b feature/my-feature
# Make changes and commit
git add .
git commit -m "feat: add new feature"
# Push and create PR
git push origin feature/my-feature
gh pr create --base main --title "Add new feature"
```

> **Important**: Feature branches should be merged within 1 day to avoid drift from main.

### 2. Continuous Integration (Automatic)
**Workflow**: `trunk-ci.yml`

Automatically runs on every push to `main`:
- ✅ Runs unit tests
- ✅ Validates Databricks bundle
- ✅ Deploys to dev workspace (on push to main)

**Triggered by**: Push to main or PR to main

### 3. Deploy to Staging
**Workflow**: `trunk-deploy-staging.yml`

Deploy the current state of main to staging when ready:
```bash
# Trigger via GitHub UI
# Go to Actions > "Trunk-Based: Deploy to Staging" > Run workflow
```

Or trigger via CLI:
```bash
gh workflow run trunk-deploy-staging.yml
```

This workflow:
- ✅ Runs tests on staging environment
- ✅ Validates bundle for staging
- ✅ Deploys to staging workspace

**Optional**: The workflow includes a daily scheduled deployment (9 AM UTC) - remove the `schedule` trigger if not needed.

### 4. Production Release
**Workflow**: `trunk-release-prod.yml`

When ready to release to production:
```bash
# Trigger via GitHub UI with version number
# Go to Actions > "Trunk-Based: Release to Production" > Run workflow
# Enter version (e.g., 1.2.0) and optional release notes
```

Or trigger via CLI:
```bash
gh workflow run trunk-release-prod.yml -f version=1.2.0 -f release_notes="New features and bug fixes"
```

This workflow:
- ✅ Runs tests
- ✅ Deploys to prod workspace
- ✅ Creates a git tag (e.g., `1.2.0`)
- ✅ Creates a GitHub release with release notes

## Best Practices for Trunk-Based Development

1. **Commit frequently**: Push small changes to main multiple times per day
2. **Keep main stable**: Every commit to main should pass all tests
3. **Use feature flags**: Hide incomplete features behind feature flags instead of using long-lived branches
4. **Quick feedback**: CI should complete in under 10 minutes
5. **Small PRs**: If using feature branches, keep them short-lived (< 1 day) and small (< 200 lines)
6. **Test locally first**: Run `uv run pytest` before pushing
7. **Progressive deployment**: Test in dev → staging → prod

## Switching Between Strategies

This repository supports both GitFlow and Trunk-Based Development workflows. You can:
- Use GitFlow exclusively (disable trunk-* workflows)
- Use Trunk-Based Development exclusively (disable draft-new-release, publish-new-release, draft-new-hotfix workflows)
- Use both simultaneously (though not recommended)

To disable workflows you're not using, simply delete the corresponding `.yml` files from `.github/workflows/`.

---


