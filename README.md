# deb-publish

A GitHub Action that builds a `.deb` package from a staged directory and publishes it to an APT repository hosted on GitHub Pages.

- Builds the `.deb` using `dpkg-deb`
- Maintains a versioned pool (all published versions are preserved)
- Regenerates `Packages`, `Packages.gz`, and `Release` on every publish
- Optionally signs the repository with GPG (`Release.gpg` + `InRelease`)
- Exports the public key as `pubkey.gpg` for easy client setup
- Retries push on concurrent conflicts

## Versioning

The package version is read from the `Version` field in `DEBIAN/control`. Since the control file is static in your repo, you update it at build time — typically from a git tag.

### Update version from a git tag

```yaml
on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    steps:
      - uses: actions/checkout@v6

      - name: Set package version from tag
        run: |
          VERSION=${GITHUB_REF_NAME#v}   # strips the leading v: v1.2.3 → 1.2.3
          sed -i "s/^Version:.*/Version: $VERSION/" package/DEBIAN/control

      - uses: Vr00mm/deb-publish@v1
        with:
          package-path: ./package
          ...
```

Every published `.deb` is kept in the pool — users can install a specific version:
```bash
sudo apt install my-package=1.2.3
```

To list available versions:
```bash
apt-cache showpkg my-package
```

## Usage

```yaml
- uses: Vr00mm/deb-publish@v1
  with:
    package-path: ./package
    token: ${{ secrets.PAGES_TOKEN }}
    pages-repo: Vr00mm/vr00mm.github.io
    pages-url: https://vr00mm.github.io
    gpg-private-key: ${{ secrets.GPG_PRIVATE_KEY }}
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `package-path` | yes | | Path to the package staging directory (see [Package structure](#package-structure)) |
| `token` | yes | | GitHub token with `contents: write` on the pages repository |
| `pages-repo` | yes | | Repository hosting the APT repo (`owner/repo`) |
| `pages-url` | yes | | Base URL of the GitHub Pages site (e.g. `https://owner.github.io`) |
| `pages-branch` | no | `master` | Branch of the pages repository |
| `deb-dir` | no | `deb` | Subdirectory in the pages repo for the APT repository |
| `distribution` | no | `stable` | APT distribution name (e.g. `stable`, `focal`, `jammy`) |
| `component` | no | `main` | APT component (e.g. `main`, `contrib`) |
| `gpg-private-key` | no | | ASCII-armored GPG private key for signing. Leave empty to skip signing. |
| `gpg-passphrase` | no | | Passphrase for the GPG key. Leave empty if the key has none. |

## Outputs

| Output | Description |
|---|---|
| `deb-file` | Filename of the built `.deb` package |
| `package-name` | Package name from `DEBIAN/control` |
| `package-version` | Package version from `DEBIAN/control` |
| `package-arch` | Package architecture from `DEBIAN/control` |

## Package structure

The `package-path` directory must follow the standard Debian staging layout — files placed where they should be installed on the target system:

```
package/
  DEBIAN/
    control       ← required: package metadata
    postinst      ← optional: script run after install
    prerm         ← optional: script run before removal
  usr/
    bin/
      my-binary   ← installs to /usr/bin/my-binary
  etc/
    my-app.conf   ← installs to /etc/my-app.conf
  lib/
    systemd/
      system/
        my-app.service  ← installs to /lib/systemd/system/my-app.service
```

### DEBIAN/control (minimal)

```
Package: my-app
Version: 1.0.0
Architecture: amd64
Maintainer: Your Name <you@example.com>
Description: Short description
 Longer description on continuation lines (leading space required).
Depends: systemd
```

## Prerequisites

### 1. GitHub Pages repository

You need a repository with GitHub Pages enabled.

### 2. PAGES_TOKEN secret

Create a **fine-grained personal access token** at https://github.com/settings/personal-access-tokens/new:

- **Repository access:** your pages repo only
- **Permissions:** `Contents` → Read and write

Add it as a secret named `PAGES_TOKEN`.

### 3. GPG key (recommended)

Generate a signing key:

```bash
gpg --batch --gen-key <<EOF
Key-Type: RSA
Key-Length: 4096
Name-Real: Your Name
Name-Email: you@example.com
%no-passphrase
%commit
EOF

# Export private key → paste into GPG_PRIVATE_KEY secret
gpg --armor --export-secret-keys you@example.com

# Export public key → commit to your pages repo as pubkey.gpg
gpg --armor --export you@example.com > pubkey.gpg
```

Without GPG signing the action still works but clients must use `trusted=yes`.

## Client setup

### With GPG signing (recommended)

```bash
curl -fsSL https://owner.github.io/pubkey.gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/owner.gpg

echo "deb [arch=amd64 signed-by=/usr/share/keyrings/owner.gpg] https://owner.github.io/deb stable main" \
  | sudo tee /etc/apt/sources.list.d/owner.list

sudo apt update
sudo apt install my-package
```

### Without GPG signing

```bash
echo "deb [arch=amd64 trusted=yes] https://owner.github.io/deb stable main" \
  | sudo tee /etc/apt/sources.list.d/owner.list

sudo apt update
sudo apt install my-package
```

## Examples

### Basic

```yaml
- uses: Vr00mm/deb-publish@v1
  with:
    package-path: ./package
    token: ${{ secrets.PAGES_TOKEN }}
    pages-repo: Vr00mm/vr00mm.github.io
    pages-url: https://vr00mm.github.io
    gpg-private-key: ${{ secrets.GPG_PRIVATE_KEY }}
    gpg-passphrase: ${{ secrets.GPG_PASSPHRASE }}
```

### Full release workflow example

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - name: Build binary
        run: |
          CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o package/usr/bin/my-app ./cmd/my-app
        working-directory: src

      - name: Set package version
        run: |
          VERSION=${GITHUB_REF_NAME#v}
          sed -i "s/^Version:.*/Version: $VERSION/" package/DEBIAN/control
          chmod 755 package/DEBIAN/postinst package/DEBIAN/prerm

      - name: Publish deb package
        uses: Vr00mm/deb-publish@v1
        with:
          package-path: ./package
          token: ${{ secrets.PAGES_TOKEN }}
          pages-repo: Vr00mm/vr00mm.github.io
          pages-url: https://vr00mm.github.io
          gpg-private-key: ${{ secrets.GPG_PRIVATE_KEY }}
          gpg-passphrase: ${{ secrets.GPG_PASSPHRASE }}
```
