# Architecture Overview

## Problem Statement

The original v1 approach had critical flaws:

1. **Race Conditions**: Multiple repos could simultaneously modify the pages repo, causing:
   - Metadata corruption (Packages, Release files)
   - Lost updates (last writer wins)
   - Inconsistent GPG state

2. **GPG Inconsistency**: If one repo signs and another doesn't:
   - Unsigned push can clobber signed repo
   - Clients get signature warnings/errors
   - No single source of truth for keys

3. **Security**: GPG keys duplicated across all app repos

4. **Inefficiency**: Every push downloads entire pages repo, rebuilds everything

## Solution: Dual-Mode Architecture

### Two-Mode Operation

**Mode 1: Dispatch (from app repos)**
- Builds `.deb` package
- Uploads as GitHub artifact (1-day retention)
- Sends `repository_dispatch` event to pages repo with metadata

**Mode 2: Publish (in pages repo)**
- Receives `repository_dispatch` event
- Downloads artifact from source repo
- Updates APT repository metadata
- Signs with centralized GPG key
- Commits and pushes

### Key Benefits

✅ **Sequential Processing**: `concurrency` key ensures one-at-a-time execution  
✅ **Single GPG Source**: Keys stored only in pages repo secrets  
✅ **No Race Conditions**: Queue-based processing prevents conflicts  
✅ **Transactional**: Each publication is atomic  
✅ **Decoupled**: App repos don't need pages repo access  
✅ **Scalable**: Any number of app repos can publish  

### Flow Diagram

```
┌─────────────────┐
│   App Repo A    │
│   (myapp v1.0)  │
└────────┬────────┘
         │ 1. Build .deb
         │ 2. Upload artifact
         │ 3. Dispatch event
         ↓
┌─────────────────┐
│   App Repo B    │
│  (mylib v2.3)   │
└────────┬────────┘
         │ 1. Build .deb
         │ 2. Upload artifact
         │ 3. Dispatch event
         ↓
┌─────────────────────────────────────────┐
│        Pages Repo (owner.github.io)     │
│                                          │
│  Concurrency Group: apt-repository-update│
│  ┌────────────────────────────────────┐ │
│  │   Queue (FIFO, non-cancellable)    │ │
│  │                                    │ │
│  │  1. [myapp v1.0] ✓ Processing...  │ │
│  │  2. [mylib v2.3] ⏳ Waiting...     │ │
│  └────────────────────────────────────┘ │
│                                          │
│  For each dispatch:                      │
│   1. Download artifact from source repo  │
│   2. Extract package metadata            │
│   3. Add .deb to pool/                   │
│   4. Regenerate Packages + Packages.gz   │
│   5. Generate Release file               │
│   6. Sign Release.gpg + InRelease        │
│   7. Export pubkey.gpg                   │
│   8. Commit + Push                       │
└──────────────────────────────────────────┘
         ↓
┌─────────────────┐
│  GitHub Pages   │
│  Auto-deploys   │
└─────────────────┘
```

## Implementation Details

### Concurrency Control

In the pages repo workflow:

```yaml
concurrency:
  group: apt-repository-update
  cancel-in-progress: false  # CRITICAL: Queue instead of cancel
```

This ensures:
- Only one workflow runs at a time
- Pending runs wait in queue
- No lost publications
- Consistent repository state

### Artifact Transfer

Instead of pushing `.deb` files to pages repo directly:

1. **App repo**: Upload as artifact (1-day retention)
2. **Dispatch event**: Send artifact metadata
3. **Pages repo**: Download artifact using `gh run download`

Benefits:
- No need for app repos to have pages repo write access
- Cleaner git history (no .deb files in commits)
- Faster checkout (no large binary history)

### GPG Key Management

**Before (v1):**
```
App Repo A secrets: GPG_PRIVATE_KEY, GPG_PASSPHRASE
App Repo B secrets: GPG_PRIVATE_KEY, GPG_PASSPHRASE
App Repo C secrets: GPG_PRIVATE_KEY, GPG_PASSPHRASE
```

**After (v2):**
```
Pages Repo secrets: GPG_PRIVATE_KEY, GPG_PASSPHRASE
App Repo A secrets: PAGES_REPO_TOKEN
App Repo B secrets: PAGES_REPO_TOKEN
App Repo C secrets: PAGES_REPO_TOKEN
```

Benefits:
- Single key rotation point
- Consistent signing
- Better secret hygiene

### Mode Auto-Detection

The action automatically detects mode:

```yaml
if github.event_name == 'repository_dispatch':
  mode = 'publish'
else:
  mode = 'dispatch'
```

But explicit mode setting is recommended for clarity:

```yaml
with:
  mode: dispatch  # or 'publish'
```

## Security Considerations

### Token Permissions

**`PAGES_REPO_TOKEN` (in app repos):**
- Scope: Only pages repo
- Permission: Contents (Read + Write)
- Used for: Sending repository_dispatch + Downloading artifacts

**`GITHUB_TOKEN` (in pages repo workflow):**
- Automatic token
- Permission: Default (Read + Write contents)
- Used for: Downloading artifacts, checking out, pushing

### GPG Key Security

- Stored only in pages repo secrets
- Never leaves GitHub secrets
- Automatically cleaned from runner after use
- Passphrase-protected recommended

## Failure Modes & Recovery

### App Repo Failure
- **Where**: During build or dispatch
- **Impact**: Only that app's package fails
- **Recovery**: Retry workflow or push new commit

### Pages Repo Failure
- **Where**: During publish
- **Impact**: That package not published, queue stalls
- **State**: Repository remains consistent (no partial updates)
- **Recovery**: Fix issue, re-run failed workflow

### Concurrent Dispatch Handling
- **Scenario**: Multiple repos dispatch simultaneously
- **Behavior**: All queued sequentially
- **Result**: All packages published in order

### Artifact Expiration
- **Retention**: 1 day
- **Mitigation**: Publish workflow should run immediately
- **Recovery**: Retrigger app workflow if artifact expired

## Performance

### Before (v1)
```
Per publish:
1. Checkout pages repo (~1-5s depending on history)
2. Process package (~2-3s)
3. Sign (~1-2s)
4. Push with retries (~2-10s with conflicts)
Total: ~6-20s

Concurrent publishes: Conflicts, retries, race conditions
```

### After (v2)
```
App repo:
1. Upload artifact (~1-2s)
2. Send dispatch (~1s)
Total: ~2-3s (non-blocking)

Pages repo (per queue item):
1. Download artifact (~1-2s)
2. Checkout pages repo (~1-3s)
3. Process package (~2-3s)
4. Sign (~1-2s)
5. Push (~1-2s, no conflicts)
Total: ~6-12s

Concurrent publishes: Queued, no conflicts
```

## Migration Path

1. Add pages repo workflow (publish mode)
2. Update app repo workflows (dispatch mode)
3. Move GPG secrets to pages repo
4. Test with one app repo
5. Migrate remaining app repos
6. Remove old workflows

## Future Enhancements

- [ ] Webhook notifications on publish completion
- [ ] Publish status badges
- [ ] Multi-architecture support in single dispatch
- [ ] Artifact caching for faster re-publishes
- [ ] Repository statistics generation
- [ ] Custom post-publish hooks
