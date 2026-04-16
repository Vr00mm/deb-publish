# deb-publish

A GitHub Action for publishing `.deb` packages to APT repositories hosted on GitHub Pages with **proper concurrency control**.

## 🎯 Why This Architecture?

This action solves critical problems with naive APT publishing:

✅ **No race conditions** - Sequential processing in pages repo prevents concurrent metadata corruption  
✅ **Consistent GPG state** - Single source of truth for signing keys  
✅ **Transactional updates** - Repository stays consistent even during failures  
✅ **Centralized control** - One workflow manages all publications  
✅ **No secret sprawl** - GPG keys stored only in pages repo  

## 🔄 How It Works

**Two-mode operation:**

1. **Dispatch Mode** (from app repos): Upload `.deb` artifact → Send dispatch event to pages repo
2. **Publish Mode** (in pages repo): Download artifact → Update APT repo → Sign → Push

```
┌────────────────┐                ┌─────────────────────┐
│   App Repo A   │─────┐          │   Pages Repository  │
└────────────────┘     │          │                     │
                       ├─dispatch─▶│  Sequential Queue   │
┌────────────────┐     │          │  (concurrency key)  │
│   App Repo B   │─────┘          │                     ││
└────────────────┘                 └─────────────────────┘
```

---

## Table of contents

1. [Quick Start](#quick-start)
2. [Prerequisites](#prerequisites)
   - [GitHub Pages repository](#github-pages-repository)
   - [Enable GitHub Pages](#enable-github-pages)
   - [Create a Personal Access Token](#create-a-personal-access-token)
   - [Generate a GPG signing key](#generate-a-gpg-signing-key)
   - [Add secrets](#add-secrets)
3. [Setup](#setup)
   - [Step 1: Pages repository workflow](#step-1-pages-repository-workflow)
   - [Step 2: App repository workflow](#step-2-app-repository-workflow)
4. [How it works](#how-it-works)
5. [Client setup](#client-setup)
6. [Inputs Reference](#inputs-reference)
7. [Outputs Reference](#outputs-reference)

---

## Quick Start

**In your pages repository** (`owner.github.io`), create `.github/workflows/publish-deb.yml`:

```yaml
name: Publish Package

on:
  repository_dispatch:
    types: [publish-deb]

concurrency:
  group: apt-repository-update
  cancel-in-progress: false

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: your-org/deb-publish@v2
        with:
          mode: publish
          gpg-private-key: ${{ secrets.GPG_PRIVATE_KEY }}
          gpg-passphrase: ${{ secrets.GPG_PASSPHRASE }}
```

**In your app repositories**, create `.github/workflows/release.yml`:

```yaml
name: Release

on:
  release:
    types: [published]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      
      - name: Build .deb
        run: dpkg-deb --build package/ myapp.deb
      
      - uses: your-org/deb-publish@v2
        with:
          mode: dispatch
          deb-path: myapp.deb
          token: ${{ secrets.PAGES_REPO_TOKEN }}
          pages-repo: your-org/your-org.github.io
```

---

## Prerequisites

### GitHub Pages repository

You need a repository with GitHub Pages enabled. The most common choice is `owner.github.io`.

**Create one:**
1. Go to <https://github.com/new>
2. Name it `owner.github.io` (user pages) or any name (project pages)
3. Set to **Public** (required for free Pages)
4. Initialize with README
5. Create

### Enable GitHub Pages

1. Repository → **Settings → Pages**
2. **Source:** Deploy from a branch
3. Select branch (`master`/`main`) and folder (`/ root`)  
4. Save

### Create a Personal Access Token

The dispatch action needs permission to trigger workflows in your pages repo.

1. Go to <https://github.com/settings/personal-access-tokens/new>
2. **Name:** `deb-publish-dispatch`
3. **Expiration:** 1 year
4. **Repository access:** Only select repositories → Pick your pages repo
5. **Permissions → Repository:**
   - **Contents:** Read and write
   - **Metadata:** Read-only (automatic)
6. Generate and copy the token

### Generate a GPG signing key

Signing is optional but **strongly recommended** for production repos.

```bash
gpg --batch --gen-key <<EOF
Key-Type: RSA
Key-Length: 4096
Name-Real: APT Repository Signing Key
Name-Email: apt@yourdomain.com
Expire-Date: 2y
%no-passphrase
%commit
EOF
```

Add passphrase (recommended):

```bash
gpg --batch --gen-key <<EOF
Key-Type: RSA
Key-Length: 4096
Name-Real: APT Repository Signing Key
Name-Email: apt@yourdomain.com
Expire-Date: 2y
Passphrase: your-secure-passphrase
%commit
EOF
```

**Export the key:**

```bash
# Find key ID
gpg --list-secret-keys --keyid-format LONG

# Export private key (for secrets)
gpg --armor --export-secret-keys KEY_ID

# Export public key (optional, action does this automatically)
gpg --armor --export KEY_ID > pubkey.gpg
```

### Add secrets

**In your PAGES repository** (`owner.github.io`):

Go to **Settings → Secrets → Actions → New repository secret**:

| Secret | Value |
|--------|-------|
| `GPG_PRIVATE_KEY` | Full ASCII-armored private key |
| `GPG_PASSPHRASE` | Passphrase (empty if none) |

**In your APP repositories** (each app that publishes packages):

Go to **Settings → Secrets → Actions → New repository secret**:

| Secret | Value |
|--------|-------|
| `PAGES_REPO_TOKEN` | Fine-grained PAT from earlier step |

---

## Setup

### Step 1: Pages repository workflow

In your **pages repository** (`owner.github.io`), create `.github/workflows/publish-deb.yml`:

```yaml
name: Publish Package to APT Repository

on:
  repository_dispatch:
    types: [publish-deb]

# CRITICAL: Ensures sequential processing (no race conditions)
concurrency:
  group: apt-repository-update
  cancel-in-progress: false

jobs:
  publish:
    runs-on: ubuntu-latest
    
    steps:
      - name: Publish to APT repository
        uses: your-org/deb-publish@v2  # Update to your action
        with:
          mode: publish  # Auto-detected, but explicit is clearer
          
          # GPG signing (optional but recommended)
          gpg-private-key: ${{ secrets.GPG_PRIVATE_KEY }}
          gpg-passphrase: ${{ secrets.GPG_PASSPHRASE }}
      
      - name: Report
        run: |
          echo "✅ Published: ${{ github.event.client_payload.package_name }} ${{ github.event.client_payload.package_version }}"
          echo "📦 From: ${{ github.event.client_payload.source_repo }}"
          echo "🔗 Pages URL: ${{ github.server_url }}/${{ github.repository }}"
```

**Commit and push this workflow.**

### Step 2: App repository workflow

In **each application repository** that builds packages, create `.github/workflows/release.yml`:

```yaml
name: Build and Publish

on:
  release:
    types: [published]
  workflow_dispatch:

jobs:
  build-and-dispatch:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v6
      
      # ========================================
      # Your build steps here
      # ========================================
      - name: Build .deb package
        run: |
          # Example: Build your Go/Rust/C++ app
          # make build
          # Then package it
          dpkg-deb --build package/ myapp.deb
      
      # ========================================
      # Dispatch publication
      # ========================================
      - name: Publish to APT repository
        uses: your-org/deb-publish@v2  # Update to your action
        with:
          mode: dispatch  # Auto-detected, but explicit is clearer
          
          # Required inputs
          deb-path: myapp.deb
          token: ${{ secrets.PAGES_REPO_TOKEN }}
          pages-repo: your-org/your-org.github.io  # Your pages repo
          
          # Optional configuration
          pages-branch: master
          deb-dir: apt
          distribution: stable
          component: main
```

**Commit, tag a release, and watch the magic happen!**

---

## How it works

1. **App repo** builds `.deb` → Uploads as artifact → Sends `repository_dispatch` to pages repo
2. **Pages repo** receives dispatch → Downloads artifact → Updates APT metadata → Signs → Commits → Pushes
3. **Concurrency control** ensures only one publication at a time (prevents corruption)
4. **GPG keys** stay in one place (pages repo secrets)

**The flow:**

```
App Repo A (v1.2.3)
  ├─ Build .deb
  ├─ Upload artifact
  └─ Dispatch event
       ↓
Pages Repo (queue)
  ├─ Download artifact
  ├─ Add to pool/
  ├─ Regenerate Packages
  ├─ Generate Release
  ├─ Sign with GPG
  └─ Push to master
       ↓
GitHub Pages deploys
  └─ https://owner.github.io/apt/
```

---

## Client setup

After your first package is published, users can install from your APT repository.

### Add the repository

```bash
# Add GPG key (if signed)
curl -fsSL https://your-org.github.io/apt/pubkey.gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/your-repo.gpg

# Add repository
echo "deb [arch=amd64] https://your-org.github.io/apt stable main" | sudo tee /etc/apt/sources.list.d/your-repo.list

# Update and install
sudo apt update
sudo apt install your-package-name
```

**For unsigned repositories** (not recommended):

```bash
echo "deb [arch=amd64 trusted=yes] https://your-org.github.io/apt stable main" | sudo tee /etc/apt/sources.list.d/your-repo.list
```

---

## Inputs Reference

### Common Inputs (both modes)

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `mode` | Operation mode: `dispatch` or `publish` (auto-detected if empty) | No | *(auto)* |
| `pages-branch` | Branch of pages repository | No | `master` |
| `deb-dir` | Subdirectory in pages repo for APT repository | No | `apt` |
| `distribution` | APT distribution (e.g., `stable`, `jammy`) | No | `stable` |
| `component` | APT component (e.g., `main`, `contrib`) | No | `main` |

### Dispatch Mode Only (app repos)

| Input | Description | Required |
|-------|-------------|----------|
| `deb-path` | Path to `.deb` file | **Yes** |
| `token` | GitHub token with dispatch permission on pages repo | **Yes** |
| `pages-repo` | Pages repository (`owner/repo`) | **Yes** |

### Publish Mode Only (pages repo)

| Input | Description | Required |
|-------|-------------|----------|
| `gpg-private-key` | ASCII-armored GPG private key | No |
| `gpg-passphrase` | GPG key passphrase | No |

---

## Outputs Reference

| Output | Description | Available In |
|--------|-------------|--------------|
| `mode` | Mode used: `dispatch` or `publish` | Both |
| `package-name` | Package name | Both |
| `package-version` | Package version | Both |
| `package-arch` | Package architecture | Both |
| `dispatched` | Whether dispatch succeeded | Dispatch mode |

---

## Advanced Usage

### Multiple distributions

Publish the same package to multiple distributions:

```yaml
- name: Publish to stable
  uses: your-org/deb-publish@v2
  with:
    mode: dispatch
    deb-path: myapp.deb
    token: ${{ secrets.PAGES_REPO_TOKEN }}
    pages-repo: your-org/your-org.github.io
    distribution: stable

- name: Publish to jammy
  uses: your-org/deb-publish@v2
  with:
    mode: dispatch
    deb-path: myapp.deb
    token: ${{ secrets.PAGES_REPO_TOKEN }}
    pages-repo: your-org/your-org.github.io
    distribution: jammy
```

### Project pages repository

For a project repo (not `owner.github.io`):

```yaml
pages-repo: your-org/apt-repo  # Pages URL will be https://your-org.github.io/apt-repo
```

### Version management

The package version is read from `DEBIAN/control`. Recommended pattern:

```bash
VERSION=${GITHUB_REF_NAME#v}  # v1.2.3 → 1.2.3
sed -i "s/^Version:.*/Version: $VERSION/" package/DEBIAN/control
```

All versions are preserved in `pool/`. Users can install specific versions:

```bash
sudo apt install mypackage=1.2.3
```

---

## Troubleshooting

### "Permission denied" or dispatch fails

- Ensure `PAGES_REPO_TOKEN` has **Contents: Read and write** permission
- Verify token is scoped to the correct pages repository
- Check token hasn't expired

### GPG signature errors

- Verify `GPG_PRIVATE_KEY` includes header/footer (`-----BEGIN/END-----`)
- Check passphrase matches the key
- Ensure key hasn't expired: `gpg --list-keys`

### Packages not appearing

- Check GitHub Pages deployment status
- Wait 1-2 minutes for Pages to deploy after push
- Verify `deb-dir` matches between app and pages workflows

### Concurrent publishes not queuing

- Ensure `concurrency.group: apt-repository-update` is set in pages workflow
- Check `cancel-in-progress: false` is set

---

## Migration from v1

**What changed:**
- Split into two-mode operation (dispatch + publish)
- Removed direct checkout of pages repo from app repos
- Added `mode` input
- Renamed `PAGES_TOKEN` → `PAGES_REPO_TOKEN` (convention)
- GPG secrets moved to pages repo only

**Migration steps:**

1. **In pages repo**: Add new workflow (see [Step 1](#step-1-pages-repository-workflow))
2. **In pages repo**: Move `GPG_PRIVATE_KEY` and `GPG_PASSPHRASE` secrets here
3. **In app repos**: Update workflow to use `mode: dispatch`
4. **In app repos**: Remove `pages-url` input (no longer used)
5. **In app repos**: Remove GPG secret references

---

## License

MIT

---

## Contributing

Issues and PRs welcome! See [example/](./example/) for sample workflows.
