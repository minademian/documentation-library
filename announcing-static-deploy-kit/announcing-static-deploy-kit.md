# static-deploy-kit: Deep Dive

**Author:** Mina Demian

**Last Updated:** February 2026

**Estimated Reading Time:** 25 minutes

**Implementation Time:** 1-2 hours (first-time setup), 15 minutes (subsequent projects)

## Elevator Pitch

> [`static-deploy-kit`](https://github.com/ursine-code/static-deploy-kit) is a reusable GitHub Actions CI/CD framework for Next.js projects with multi-environment deployments, semantic versioning, and comprehensive testing.  
---

## Quick Navigation

| Scenario | Start Here |
|----------|------------|
| New project setup | [Part 1: Architecture Overview](#part-1-architecture-overview) |
| Adding to existing repo | [Part 3: Integration Guide](#part-3-integration-guide) |
| Understanding the release flow | [Part 2: System Design](#part-2-system-design) |
| Customizing for your domain | [Part 4: Configuration](#part-4-configuration) |
| Debugging failed deployments | [Part 6: Troubleshooting](#part-6-troubleshooting) |

---

## Prerequisites

Before implementing this framework, ensure you have:

- [ ] GitHub repository with Actions enabled
- [ ] Next.js project with `output: "export"` in `next.config.js`
- [ ] SFTP-accessible deployment server
- [ ] Node.js 20+ and pnpm installed locally
- [ ] Experience with GitHub Actions workflows

---

## Part 1: Architecture Overview

### 1.1 High-Level System Architecture

The framework implements a complete CI/CD pipeline with three deployment contexts:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        GitHub Repository                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   Source    │  │  Workflows  │  │   Actions   │  │   Secrets   │     │
│  │    Code     │  │   (YAML)    │  │ (Reusable)  │  │  (Config)   │     │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘     │
└─────────┼────────────────┼────────────────┼────────────────┼────────────┘
          │                │                │                │
          ▼                ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      GitHub Actions Runtime                             │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                    Pipeline Stages                                │  │
│  │                                                                   │  │
│  │  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐            │  │
│  │  │  Lint   │──▶│  Type   │──▶│  Test   │──▶│  Build  │            │  │
│  │  │         │   │  Check  │   │ BDD/E2E │   │         │            │  │
│  │  └─────────┘   └─────────┘   └─────────┘   └────┬────┘            │  │
│  │                                                  │                │  │
│  │                              ┌───────────────────┼───────────────┐   │
│  │                              │                   ▼               │   │
│  │                              │  ┌─────────────────────────────┐  │   │
│  │                              │  │      Deployment Router      │  │   │
│  │                              │  └──────────────┬──────────────┘  │   │
│  │                              │     ┌───────────┼───────────┐     │   │
│  │                              │     ▼           ▼           ▼     │   │
│  │                              │ ┌───────┐  ┌─────────┐  ┌───────┐ │   │
│  │                              │ │Sandbox│  │Production│ │Release│ │   │
│  │                              │ └───┬───┘  └────┬────┘  └───┬───┘ │   │
│  │                              └─────┼───────────┼───────────┼─────┘   │
│  └────────────────────────────────────┼───────────┼───────────┼─────┘   │
└───────────────────────────────────────┼───────────┼───────────┼─────────┘
                                        │           │           │
                                        ▼           ▼           ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        Remote Server (SFTP)                             │
│                                                                         │
│  /deploy-path/                                                          │
│  ├── latest/          ◀── symlink to current production release         │
│  ├── releases/                                                          │
│  │   ├── v1.0.0/                                                        │
│  │   ├── v1.1.0/                                                        │
│  │   └── v2.0.0/      ◀── tagged releases with rollback capability      │
│  └── sandboxes/                                                         │
│      ├── feature-auth/                                                  │
│      └── fix-header/  ◀── PR preview environments                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Deployment Context Decision Tree

```
                              ┌─────────────────┐
                              │  GitHub Event   │
                              └────────┬────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
                    ▼                  ▼                  ▼
            ┌───────────────┐  ┌───────────────┐  ┌───────────────┐
            │ PR Opened/    │  │  PR Merged    │  │   Manual      │
            │ Synchronized  │  │  to main      │  │   Dispatch    │
            └───────┬───────┘  └───────┬───────┘  └───────┬───────┘
                    │                  │                  │
                    ▼                  ▼                  ▼
            ┌───────────────┐  ┌───────────────┐  ┌───────────────┐
            │   SANDBOX     │  │  PRODUCTION   │  │  Tag Deploy   │
            │  Deployment   │  │  Deployment   │  │  or Release   │
            └───────────────┘  └───────────────┘  └───────────────┘
                    │                  │                  │
                    ▼                  ▼                  ▼
            sandboxes/         latest/            releases/
            {branch}/          (symlink)          v{x.y.z}/
```

### 1.3 Component Inventory

| Component | Type | Purpose |
|-----------|------|---------|
| `release.yml` | Workflow | Production/release deployments |
| `feature-branch.yml` | Workflow | Sandbox deployments + PR previews |
| `e2e.yml` | Workflow | Reusable E2E test runner |
| `dry-run.yml` | Workflow | Pipeline simulation for testing |
| `setup-node-pnpm` | Action | Environment setup with caching |
| `build-project` | Action | Next.js build + artifact upload |
| `typecheck` | Action | TypeScript validation |
| `bdd-tests` | Action | Cucumber smoke tests |
| `create-git-tag` | Action | Semantic version management |
| `prepare-git-tag` | Action | Deployment context detection |
| `configure-deployment` | Action | Path/URL configuration |
| `deploy-to-remote-server` | Action | SFTP deployment with backup |
| `create-github-release` | Action | Release + changelog generation |

---

## Part 2: System Design

### 2.1 Workflow Orchestration

#### Release Workflow (`release.yml`)

Triggered on PR merge to `main` or manual dispatch.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        release.yml                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐                                                   │
│  │ check-debug  │ ◀── Scan commits for [DEBUG_DEPLOYMENT_INFO]      │
│  └──────┬───────┘                                                   │
│         │                                                           │
│         ▼                                                           │
│  ┌──────────────┐                                                   │
│  │ create-tag   │ ◀── Parse PR for [major]/[minor]/[patch]          │
│  └──────┬───────┘     Create semantic version tag                   │
│         │                                                           │
│    ┌────┴────┐                                                      │
│    ▼         ▼                                                      │
│ ┌──────┐ ┌──────────┐                                               │
│ │ lint │ │typecheck │  (parallel)                                   │
│ └──┬───┘ └────┬─────┘                                               │
│    └────┬─────┘                                                     │
│         ▼                                                           │
│  ┌──────────────┐                                                   │
│  │ commit-type  │ ◀── Detect ci:/chore:/docs: for test skipping     │
│  └──────┬───────┘                                                   │
│         │                                                           │
│    ┌────┴────┐                                                      │
│    ▼         ▼                                                      │
│ ┌──────┐ ┌──────┐                                                   │
│ │ bdd  │ │ e2e  │  (conditional, parallel)                          │
│ └──┬───┘ └──┬───┘                                                   │
│    └────┬───┘                                                       │
│         ▼                                                           │
│  ┌──────────────┐                                                   │
│  │    build     │ ◀── Next.js production build + artifact upload    │
│  └──────┬───────┘                                                   │
│         ▼                                                           │
│  ┌──────────────┐                                                   │
│  │   deploy     │ ◀── SFTP to production with backup                │
│  └──────┬───────┘                                                   │
│         ▼                                                           │
│  ┌──────────────┐                                                   │
│  │   release    │ ◀── GitHub release + changelog                    │
│  └──────────────┘                                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### Feature Branch Workflow (`feature-branch.yml`)

Triggered on PR open, synchronize, or reopen.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     feature-branch.yml                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐                                                   │
│  │ check-debug  │                                                   │
│  └──────┬───────┘                                                   │
│         │                                                           │
│    ┌────┴────┐                                                      │
│    ▼         ▼                                                      │
│ ┌──────┐ ┌──────────┐                                               │
│ │ lint │ │typecheck │                                               │
│ └──┬───┘ └────┬─────┘                                               │
│    └────┬─────┘                                                     │
│         ▼                                                           │
│  ┌──────────────┐                                                   │
│  │ commit-type  │                                                   │
│  └──────┬───────┘                                                   │
│         │                                                           │
│    ┌────┴────┐                                                      │
│    ▼         ▼                                                      │
│ ┌──────┐ ┌──────┐                                                   │
│ │ bdd  │ │ e2e  │                                                   │
│ └──┬───┘ └──┬───┘                                                   │
│    └────┬───┘                                                       │
│         ▼                                                           │
│  ┌──────────────┐                                                   │
│  │    build     │                                                   │
│  └──────┬───────┘                                                   │
│         ▼                                                           │
│  ┌──────────────┐                                                   │
│  │   deploy     │ ◀── SFTP to sandboxes/{branch}/                   │
│  └──────┬───────┘                                                   │
│         ▼                                                           │
│  ┌──────────────┐                                                   │
│  │  PR Comment  │ ◀── Post preview URL to pull request              │
│  └──────────────┘                                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 Semantic Versioning Engine

The `create-git-tag` action implements automatic semantic version bumping:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Version Detection Logic                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Input Sources (priority order):                                    │
│  1. PR Title: "feat: add auth [major]"                              │
│  2. PR Body: "This PR includes #minor changes"                      │
│  3. Commit message markers                                          │
│  4. Default: patch (for merged PRs without markers)                 │
│                                                                     │
│  ┌─────────────────┐                                                │
│  │  Parse Input    │                                                │
│  └────────┬────────┘                                                │
│           │                                                         │
│           ▼                                                         │
│  ┌─────────────────┐     ┌─────────────────────────────────────┐    │
│  │ Match Pattern   │────▶│ [major] | #major  →  MAJOR bump     │    │
│  │                 │     │ [minor] | #minor  →  MINOR bump     │    │
│  │                 │     │ [patch] | #patch  →  PATCH bump     │    │
│  │                 │     │ (none on merge)   →  PATCH default  │    │
│  └────────┬────────┘     └─────────────────────────────────────┘    │
│           │                                                         │
│           ▼                                                         │
│  ┌─────────────────┐                                                │
│  │ Get Latest Tag  │ ◀── git describe --tags --abbrev=0             │
│  └────────┬────────┘                                                │
│           │                                                         │
│           ▼                                                         │
│  ┌─────────────────┐                                                │
│  │ Calculate Next  │                                                │
│  │                 │     v1.2.3 + MAJOR = v2.0.0                    │
│  │                 │     v1.2.3 + MINOR = v1.3.0                    │
│  │                 │     v1.2.3 + PATCH = v1.2.4                    │
│  └────────┬────────┘                                                │
│           │                                                         │
│           ▼                                                         │
│  ┌─────────────────┐                                                │
│  │ Create & Push   │ ◀── git tag -a vX.Y.Z && git push --tags       │
│  └─────────────────┘                                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.3 Smart Test Skipping

The framework analyzes commit messages to skip tests for infrastructure-only changes:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Commit Type Detection                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Pattern Matching:                                                  │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  ^([a-f0-9]+ )?((ci|chore|docs):|[a-z]+\(ci\):)             │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  Matches:                          Does NOT Match:                  │
│  ├── ci: update workflow           ├── feat: add login              │
│  ├── chore: bump deps              ├── fix: resolve crash           │
│  ├── docs: update readme           ├── refactor: extract utils      │
│  ├── build(ci): fix cache          └── test: add unit tests         │
│  └── abc123 ci: pipeline fix                                        │
│                                                                     │
│  ┌─────────────────┐                                                │
│  │ If CI Pattern   │──▶ Skip BDD tests                              │
│  │    Matches      │──▶ Skip E2E tests                              │
│  └─────────────────┘                                                │
│                                                                     │
│  Override: [DEBUG_DEPLOYMENT_INFORMATION] forces full pipeline      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.4 Deployment Strategy

#### SFTP Deployment with Backup

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Deployment Sequence                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. BACKUP EXISTING                                                 │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │  if [ -d "$TARGET_DIR" ]; then                          │     │
│     │    mv "$TARGET_DIR" "${TARGET_DIR}.backup-${TIMESTAMP}" │     │
│     │  fi                                                      │    │
│     └─────────────────────────────────────────────────────────┘     │
│                                                                     │
│  2. CREATE DIRECTORY                                                │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │  mkdir -p "$TARGET_DIR"                                  │    │
│     └─────────────────────────────────────────────────────────┘     │
│                                                                     │
│  3. DEPLOY VIA SCP                                                  │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │  scp -r out/* user@host:$TARGET_DIR/                    │     │
│     └─────────────────────────────────────────────────────────┘     │
│                                                                     │
│  4. UPDATE SYMLINK (releases only)                                  │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │  rm -f latest && ln -sf "releases/v1.2.3" latest        │     │
│     └─────────────────────────────────────────────────────────┘     │
│                                                                     │
│  5. VERIFY DEPLOYMENT                                               │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │  if [ -f "$TARGET_DIR/index.html" ]; then               │     │
│     │    echo "Deployment verified"                            │    │
│     │  fi                                                      │    │
│     └─────────────────────────────────────────────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### Rollback Capability

The symlink-based release structure enables instant rollback:

```bash
# Current state
latest -> releases/v2.0.0

# Rollback to previous version
ssh user@host "cd /deploy-path && rm latest && ln -sf releases/v1.9.0 latest"

# Verify
curl https://example.com/version.json
```

---

## Part 3: Integration Guide

### 3.1 Repository Structure

The `static-deploy-kit` repository is organized for remote `uses:` support:

```
static-deploy-kit/
├── .github/
│   ├── actions/              # Reusable composite actions
│   │   ├── setup-node-pnpm/
│   │   ├── build-project/
│   │   ├── typecheck/
│   │   ├── bdd-tests/
│   │   ├── create-git-tag/
│   │   ├── prepare-git-tag/
│   │   ├── configure-deployment/
│   │   ├── deploy-to-remote-server/
│   │   └── create-github-release/
│   └── workflows/            # Reusable workflows
│       ├── e2e.yml
│       └── dry-run.yml
└── starter-workflows/        # Copy these to your repo
    ├── release.yml
    └── feature-branch.yml
```

### 3.2 Quick Start

**Step 1:** Copy starter workflows to your repository:

```bash
mkdir -p .github/workflows
curl -o .github/workflows/release.yml \
  https://raw.githubusercontent.com/ursine-code/static-deploy-kit/main/starter-workflows/release.yml
curl -o .github/workflows/feature-branch.yml \
  https://raw.githubusercontent.com/ursine-code/static-deploy-kit/main/starter-workflows/feature-branch.yml
```

**Step 2:** Replace the `ursine-code` placeholder with your GitHub username/org:

```bash
sed -i 's/ursine-code/your-github-username/g' .github/workflows/*.yml
```

**Step 3:** Configure repository secrets (see below).

### 3.3 Using Actions Directly

Reference individual actions in custom workflows:

```yaml
steps:
  - uses: ursine-code/static-deploy-kit/.github/actions/setup-node-pnpm@main
  - uses: ursine-code/static-deploy-kit/.github/actions/build-project@main
    with:
      artifact-name: 'build-files'
```

### 3.4 Using Reusable Workflows

Call reusable workflows from your workflows:

```yaml
jobs:
  e2e:
    uses: ursine-code/static-deploy-kit/.github/workflows/e2e.yml@main
    with:
      environment: 'production'
    secrets: inherit
```

### 3.5 Repository Secrets

Configure the following secrets in **Settings > Secrets and variables > Actions**:

| Secret | Required | Description |
|--------|----------|-------------|
| `SFTP_HOST` | Yes | Deployment server hostname |
| `SFTP_USERNAME` | Yes | SFTP authentication username |
| `SFTP_PASSWORD` | Yes | SFTP authentication password |
| `DEPLOY_BASE_PATH` | Yes | Remote base path (e.g., `/home/user/example.com`) |
| `PRODUCTION_DOMAIN` | Yes | Production domain (e.g., `example.com`) |
| `SANDBOX_DOMAIN` | No | Sandbox subdomain (e.g., `sandbox.example.com`) |

### 3.6 Next.js Configuration

Ensure your `next.config.js` includes static export:

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'export',
  // ... other config
};

module.exports = nextConfig;
```

### 3.7 Package Scripts

Add required scripts to `package.json`:

```json
{
  "scripts": {
    "build": "next build",
    "lint": "eslint . --ext .ts,.tsx",
    "type-check": "tsc --noEmit",
    "test:e2e": "playwright test",
    "test:bdd": "cucumber-js"
  }
}
```

---

## Part 4: Configuration

### 4.1 Workflow Inputs

#### Release Workflow (`workflow_dispatch`)

| Input | Type | Options | Description |
|-------|------|---------|-------------|
| `deployment_type` | choice | `manual_tag_deploy`, `automatic_deploy` | Deployment mode |
| `tag_to_deploy` | string | e.g., `v2.2.3` | Tag for manual deployment |
| `create_github_release` | boolean | `true`/`false` | Create GitHub release |
| `deploy_target` | choice | `staging`, `latest` | Target environment |
| `release_type` | choice | `none`, `patch`, `minor`, `major` | Version bump type |
| `enable_debug_output` | boolean | `true`/`false` | Verbose logging |

### 4.2 Environment Variables

Set in workflow or repository settings:

```yaml
env:
  NODE_VERSION: '20.18.0'
  PNPM_VERSION: '10.18.1'
```

### 4.3 Customizing Actions

#### Build Caching

The `setup-node-pnpm` action implements intelligent caching:

```yaml
- name: Cache pnpm store
  uses: actions/cache@v4
  with:
    path: ~/.pnpm-store
    key: pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}
    restore-keys: pnpm-

- name: Cache Next.js build
  uses: actions/cache@v4
  with:
    path: .next/cache
    key: nextjs-${{ hashFiles('**/*.ts', '**/*.tsx', 'next.config.js') }}
```

#### Test Configuration

Skip patterns can be extended in `check-commit-type` step:

```bash
# Add custom skip patterns
if echo "$COMMIT_MSG" | grep -qE '^(ci|chore|docs|style):'; then
  SHOULD_SKIP="true"
fi
```

---

## Part 5: Operations

### 5.1 Creating a Release

**Automatic (recommended):**

1. Create PR with version marker in title or body:
   ```
   feat: add user dashboard [minor]
   ```

2. Merge PR to `main`

3. Pipeline automatically:
   - Creates tag `v1.3.0`
   - Runs tests
   - Builds and deploys
   - Publishes GitHub release

**Manual:**

1. Go to **Actions > Release and Deploy**
2. Click **Run workflow**
3. Select:
   - `deployment_type`: `manual_tag_deploy`
   - `tag_to_deploy`: `v1.2.3`
   - `create_github_release`: `true`

### 5.2 Preview Deployments

1. Open pull request against `main`
2. Pipeline automatically:
   - Runs all checks
   - Deploys to `sandboxes/{branch-name}/`
   - Posts preview URL as PR comment

### 5.3 Debug Mode

Add `[DEBUG_DEPLOYMENT_INFORMATION]` to any commit message:

```bash
git commit -m "fix: resolve issue [DEBUG_DEPLOYMENT_INFORMATION]"
```

This enables verbose logging throughout the pipeline.

### 5.4 Dry Run Testing

Test pipeline changes without actual deployment:

1. Go to **Actions > Dry Run Release**
2. Configure simulation parameters
3. Review simulated outputs

---

## Part 6: Troubleshooting

### Issue 1: Deployment Skipped - No Build Artifacts

**Symptom:**
```
Deployment skipped: No build artifacts were generated
```

**Diagnosis:**
```bash
# Check Next.js config
grep -r "output" next.config.js
```

**Solution:**
Ensure `next.config.js` includes:
```javascript
output: 'export'
```

### Issue 2: Tag Already Exists

**Symptom:**
```
fatal: tag 'v1.2.3' already exists
```

**Diagnosis:**
```bash
git tag -l "v1.2.*"
```

**Solution:**
- Use a different version marker in PR
- Or delete the existing tag (if safe):
  ```bash
  git tag -d v1.2.3
  git push origin :refs/tags/v1.2.3
  ```

### Issue 3: SFTP Connection Failed

**Symptom:**
```
ssh: connect to host example.com port 22: Connection refused
```

**Diagnosis:**
1. Verify secrets are set correctly
2. Test connection locally:
   ```bash
   sftp user@host
   ```

**Solution:**
- Verify `SFTP_HOST`, `SFTP_USERNAME`, `SFTP_PASSWORD` secrets
- Check server firewall rules
- Ensure SSH service is running on server

### Issue 4: Tests Unexpectedly Skipped

**Symptom:**
Tests skipped when they should run.

**Diagnosis:**
Check commit message pattern:
```bash
git log -1 --format="%s"
```

**Solution:**
Ensure commit message doesn't match skip patterns:
- `ci:`, `chore:`, `docs:`
- `<type>(ci):`

### Issue 5: Preview URL Not Posted

**Symptom:**
Deployment succeeds but no PR comment appears.

**Diagnosis:**
Check workflow permissions in repository settings.

**Solution:**
Ensure **Settings > Actions > General > Workflow permissions** includes:
- Read and write permissions
- Allow GitHub Actions to create pull request comments

---

## Part 7: Security Considerations

### 7.1 Secret Management

- Store all credentials in GitHub Secrets, never in code
- Use dedicated deployment user with minimal permissions
- Rotate credentials periodically

### 7.2 SFTP User Permissions

Restrict the deployment user to the deployment directory only:

```bash
# /etc/ssh/sshd_config
Match User deployuser
  ChrootDirectory /home/deployuser
  ForceCommand internal-sftp
  AllowTcpForwarding no
```

### 7.3 Build-Time Variables

Variables prefixed with `NEXT_PUBLIC_` are embedded in the client bundle:

```
NEXT_PUBLIC_API_URL     # Visible in browser - OK
API_SECRET_KEY          # Server-only - never expose
```

---

## Appendix A: Action Reference

### configure-deployment

**Inputs:**
| Name | Required | Description |
|------|----------|-------------|
| `event-name` | Yes | GitHub event name |
| `ref` | Yes | GitHub ref |
| `ref-name` | Yes | GitHub ref name |
| `deployment-context` | Yes | `release`, `sandbox`, or `production` |
| `base-path` | Yes | Server base path |
| `production-domain` | Yes | Production domain |
| `sandbox-domain` | No | Sandbox domain |

**Outputs:**
| Name | Description |
|------|-------------|
| `deployment-type` | `production`, `release`, or `sandbox` |
| `target-path` | Remote deployment path |
| `deployment-url` | Public URL |
| `environment-name` | Environment identifier |

### deploy-to-remote-server

**Inputs:**
| Name | Required | Description |
|------|----------|-------------|
| `deployment-mode` | Yes | `automatic`, `manual`, `sandbox`, `skip` |
| `ssh-host` | Yes | SFTP hostname |
| `ssh-username` | Yes | SFTP username |
| `ssh-password` | Yes | SFTP password |
| `base-path` | Yes | Server base path |
| `production-domain` | Yes | Production domain |
| `build-artifact-name` | Yes | Artifact to deploy |

**Outputs:**
| Name | Description |
|------|-------------|
| `deployment-url` | Deployed URL |
| `deployment-path` | Remote path |
| `deployment-skipped` | Whether deployment was skipped |

---

## References

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Next.js Static Exports](https://nextjs.org/docs/app/building-your-application/deploying/static-exports)
- [Semantic Versioning](https://semver.org/)
- [Conventional Commits](https://www.conventionalcommits.org/)
