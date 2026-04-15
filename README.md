# deb-publish

A GitHub Action that publishes a pre-built `.deb` package to an APT repository hosted on GitHub Pages.

**What it does on every run:**

- Copies the provided `.deb` into the pool, preserving all previously published versions
- Regenerates `Packages`, `Packages.gz`, and `Release`
- Signs `Release.gpg` and `InRelease` with your GPG key (optional but recommended)
- Exports `pubkey.gpg` so clients can add your key with one command
- Pushes to your pages repository with automatic retry on concurrent conflicts

---

## Table of contents

1. [Prerequisites](#1-prerequisites)
   - [1.1 GitHub Pages repository](#11-github-pages-repository)
   - [1.2 Enable GitHub Pages](#12-enable-github-pages)
   - [1.3 Create a Personal Access Token](#13-create-a-personal-access-token)
   - [1.4 Generate a GPG signing key](#14-generate-a-gpg-signing-key)
   - [1.5 Add secrets to your source repository](#15-add-secrets-to-your-source-repository)
2. [Usage](#2-usage)
   - [Minimal example](#minimal-example)
   - [Full release workflow](#full-release-workflow)
3. [Versioning](#3-versioning)
4. [Client setup](#4-client-setup)
5. [Inputs](#inputs)
6. [Outputs](#outputs)

---

## 1. Prerequisites

### 1.1 GitHub Pages repository

You need a GitHub repository with Pages enabled to store the APT repository files.
The most common choice is your **user/org pages repo** (`owner/owner.github.io`), but any repository works.

**Option A — Use your existing `owner.github.io` repo**

If you already have `https://github.com/owner/owner.github.io`, skip to [1.2](#12-enable-github-pages).
The APT repository will live at `https://owner.github.io/apt/` by default.

**Option B — Create a dedicated repository**

1. Go to <https://github.com/new>
2. Name it `owner.github.io` (replaces your current user pages) **or** any other name (e.g. `apt`)
3. Set visibility to **Public** (required for free GitHub Pages)
4. Initialize with a README so the default branch exists
5. Click **Create repository**

> If you use a non-`owner.github.io` repo, the Pages URL will be
> `https://owner.github.io/repo-name` — set `pages-url` accordingly.

---

### 1.2 Enable GitHub Pages

1. Open the pages repository on GitHub
2. Go to **Settings → Pages**
3. Under **Build and deployment**, set **Source** to **Deploy from a branch**
4. Select the branch (`master` or `main`) and folder **/ (root)**
5. Click **Save**

GitHub will show the published URL (e.g. `https://owner.github.io`). It may take a minute for the first deployment.

---

### 1.3 Create a Personal Access Token

The action needs to push commits to your pages repository.
Create a **fine-grained personal access token** scoped only to that repo.

1. Go to <https://github.com/settings/personal-access-tokens/new>
2. Fill in:
   - **Token name**: `deb-publish` (or any name you like)
   - **Expiration**: choose a duration (1 year is a common balance)
   - **Resource owner**: your account or org
3. Under **Repository access**, select **Only select repositories** and pick your pages repo
4. Under **Permissions → Repository permissions**, set **Contents** to **Read and write**
5. Leave everything else as **No access**
6. Click **Generate token** and copy the value immediately — it will not be shown again

Keep this token; you will add it as a secret in [1.5](#15-add-secrets-to-your-source-repository).

---

### 1.4 Generate a GPG signing key

Signing is optional but strongly recommended. Without it, clients must use `trusted=yes` which suppresses authentication warnings.

#### Generate the key

Run the following on your local machine (Linux/macOS) or in WSL:

```bash
gpg --batch --gen-key <<EOF
Key-Type: RSA
Key-Length: 4096
Name-Real: Your Name
Name-Email: you@example.com
Expire-Date: 0
%no-passphrase
%commit
EOF
```

To add a passphrase (more secure), replace `%no-passphrase` with:

```
Passphrase: your-passphrase-here
```

#### Find the key ID

```bash
gpg --list-secret-keys --keyid-format LONG you@example.com
```

Output example:

```
sec   rsa4096/AABBCCDD11223344 2024-01-01 [SC]
      FINGERPRINT...
uid           [ultimate] Your Name <you@example.com>
```

The key ID is the part after `rsa4096/`: `AABBCCDD11223344`.

#### Export the private key

This is the value you will store as the `GPG_PRIVATE_KEY` secret:

```bash
gpg --armor --export-secret-keys AABBCCDD11223344
```

Copy the entire output including the `-----BEGIN PGP PRIVATE KEY BLOCK-----` header and footer.

#### Export the public key (optional pre-commit)

The action automatically exports and pushes `pubkey.gpg` on every run.
You can also export it manually to commit it to your pages repo in advance:

```bash
gpg --armor --export AABBCCDD11223344 > pubkey.gpg
```

---

### 1.5 Add secrets to your source repository

Open the repository that **contains your source code and workflow** (not the pages repo).

Go to **Settings → Secrets and variables → Actions → New repository secret** and add:

| Secret name | Value |
|---|---|
| `PAGES_TOKEN` | The fine-grained PAT from [1.3](#13-create-a-personal-access-token) |
| `GPG_PRIVATE_KEY` | The full ASCII-armored private key from [1.4](#14-generate-a-gpg-signing-key) |
| `GPG_PASSPHRASE` | The passphrase you set, or leave empty if you used `%no-passphrase` |

> `GPG_PRIVATE_KEY` and `GPG_PASSPHRASE` are only required if you want signed releases.
> You can skip them and omit those inputs from the workflow — the action will still publish but without signing.

---

## 2. Usage

### Minimal example

```yaml
- name: Publish .deb to APT repository
  uses: Vr00mm/deb-publish@v1
  with:
    deb-path: ./dist/my-app_1.2.3_amd64.deb
    token: ${{ secrets.PAGES_TOKEN }}
    pages-repo: owner/owner.github.io
    pages-url: https://owner.github.io
    gpg-private-key: ${{ secrets.GPG_PRIVATE_KEY }}
    gpg-passphrase: ${{ secrets.GPG_PASSPHRASE }}
```

### Full release workflow

This workflow triggers on a version tag (`v1.2.3`), builds a Go binary,
packages it into a `.deb`, then publishes it.

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write   # needed to create a GitHub Release

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: stable

      - name: Build binary
        run: |
          CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
            go build -o package/usr/bin/my-app ./cmd/my-app

      - name: Stamp version into control file
        run: |
          VERSION=${GITHUB_REF_NAME#v}   # strips leading 'v': v1.2.3 → 1.2.3
          sed -i "s/^Version:.*/Version: $VERSION/" package/DEBIAN/control
          chmod 755 package/DEBIAN/postinst package/DEBIAN/prerm

      - name: Build .deb package
        id: build
        run: |
          dpkg-deb --build --root-owner-group package /tmp/
          DEB_FILE=$(ls /tmp/*.deb | head -1)
          echo "deb-path=$DEB_FILE" >> $GITHUB_OUTPUT

      - name: Publish .deb to APT repository
        uses: Vr00mm/deb-publish@v1
        with:
          deb-path: ${{ steps.build.outputs.deb-path }}
          token: ${{ secrets.PAGES_TOKEN }}
          pages-repo: owner/owner.github.io
          pages-url: https://owner.github.io
          gpg-private-key: ${{ secrets.GPG_PRIVATE_KEY }}
          gpg-passphrase: ${{ secrets.GPG_PASSPHRASE }}

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true
```

### Publishing to a subdirectory of the pages repo

If your pages repo is not `owner.github.io` but e.g. `owner/apt`:

```yaml
- uses: Vr00mm/deb-publish@v1
  with:
    deb-path: ${{ steps.build.outputs.deb-path }}
    token: ${{ secrets.PAGES_TOKEN }}
    pages-repo: owner/apt
    pages-url: https://owner.github.io/apt   # note the /repo-name suffix
    pages-branch: main
    gpg-private-key: ${{ secrets.GPG_PRIVATE_KEY }}
    gpg-passphrase: ${{ secrets.GPG_PASSPHRASE }}
```

### Multiple distributions

Run the action twice with different `distribution` values:

```yaml
- uses: Vr00mm/deb-publish@v1
  with:
    deb-path: ${{ steps.build.outputs.deb-path }}
    distribution: jammy
    token: ${{ secrets.PAGES_TOKEN }}
    pages-repo: owner/owner.github.io
    pages-url: https://owner.github.io
    gpg-private-key: ${{ secrets.GPG_PRIVATE_KEY }}
    gpg-passphrase: ${{ secrets.GPG_PASSPHRASE }}

- uses: Vr00mm/deb-publish@v1
  with:
    deb-path: ${{ steps.build.outputs.deb-path }}
    distribution: noble
    token: ${{ secrets.PAGES_TOKEN }}
    pages-repo: owner/owner.github.io
    pages-url: https://owner.github.io
    gpg-private-key: ${{ secrets.GPG_PRIVATE_KEY }}
    gpg-passphrase: ${{ secrets.GPG_PASSPHRASE }}
```

---

## 3. Versioning

The package version is read from the `Version` field in `DEBIAN/control`.
The recommended pattern is to keep a placeholder in the file and overwrite it at build time from the git tag:

```bash
VERSION=${GITHUB_REF_NAME#v}   # v1.2.3 → 1.2.3
sed -i "s/^Version:.*/Version: $VERSION/" package/DEBIAN/control
```

Every published `.deb` is kept in the pool — older versions are never deleted.
Clients can install a specific version:

```bash
sudo apt install my-package=1.2.3
```

To list all available versions:

```bash
apt-cache showpkg my-package
```

---

## 4. Client setup

### With GPG signing (recommended)

Replace `owner` with your GitHub username/org and `my-package` with your package name.

```bash
# 1. Download and install the repository signing key
curl -fsSL https://owner.github.io/apt/pubkey.gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/owner.gpg

# 2. Add the APT source
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/owner.gpg] \
  https://owner.github.io/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/owner.list

# 3. Update and install
sudo apt update
sudo apt install my-package
```

> If you used a custom `deb-dir`, replace `deb` in the URL and source line with your value.
> If you used a custom `distribution` or `component`, update those in the source line too.

### Without GPG signing

```bash
echo "deb [arch=amd64 trusted=yes] https://owner.github.io/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/owner.list

sudo apt update
sudo apt install my-package
```

> `trusted=yes` tells apt to skip signature verification. This is fine for internal or personal use,
> but not recommended for packages distributed publicly.

---

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `deb-path` | yes | | Path to the pre-built `.deb` file to publish. |
| `token` | yes | | Fine-grained PAT with `Contents: Read and write` on the pages repository |
| `pages-repo` | yes | | Repository hosting the APT repo (`owner/repo`) |
| `pages-url` | yes | | Base URL of the GitHub Pages site (`https://owner.github.io` or `https://owner.github.io/repo`) |
| `pages-branch` | no | `master` | Branch of the pages repository |
| `deb-dir` | no | `apt` | Subdirectory inside the pages repo that holds the APT repository files |
| `distribution` | no | `stable` | APT distribution name (e.g. `stable`, `focal`, `jammy`, `noble`) |
| `component` | no | `main` | APT component (e.g. `main`, `contrib`, `non-free`) |
| `gpg-private-key` | no | | ASCII-armored GPG private key. Leave empty to publish without signing. |
| `gpg-passphrase` | no | | Passphrase for the GPG key. Leave empty if the key was generated with `%no-passphrase`. |

## Outputs

| Output | Description |
|---|---|
| `deb-file` | Filename of the published `.deb` (e.g. `my-app_1.2.3_amd64.deb`) |
| `package-name` | `Package` field read from the `.deb` metadata |
| `package-version` | `Version` field read from the `.deb` metadata |
| `package-arch` | `Architecture` field read from the `.deb` metadata |

Use outputs in subsequent steps:

```yaml
- name: Publish .deb to APT repository
  id: publish
  uses: Vr00mm/deb-publish@v1
  with:
    deb-path: ${{ steps.build.outputs.deb-path }}
    ...

- run: echo "Published ${{ steps.publish.outputs.deb-file }}"
```
