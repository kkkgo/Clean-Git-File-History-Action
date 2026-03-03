# Clean Git File History Action

A GitHub Action that removes large files and/or specific filenames from your entire git history, while **automatically preserving the latest version of every matched file in HEAD**.

---


## Usage

Create a workflow file in your repository at `.github/workflows/clean-history.yml`:

```yaml
name: Clean Git File History

on:
  workflow_dispatch:
    inputs:
      size_threshold:
        description: 'Size threshold (e.g. 10M, 500K). Leave empty to skip.'
        required: false
        default: ''
      filenames:
        description: 'Comma-separated filenames or glob patterns (e.g. *.zip,*.psd). Leave empty to skip.'
        required: false
        default: ''

jobs:
  clean:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: kkkgo/Clean-Git-File-History-Action@main
        with:
          size_threshold: ${{ github.event.inputs.size_threshold }}
          filenames: ${{ github.event.inputs.filenames }}
```

Then go to **Actions → Clean Git File History → Run workflow**, fill in your inputs, and click **Run workflow**.

---

## Inputs

| Input | Required | Description | Example |
|---|---|---|---|
| `size_threshold` | No* | Strip history of any file version larger than this size. Supports `K` / `M` / `G` suffixes. | `10M`, `500K`, `2G` |
| `filenames` | No* | Comma-separated filenames or glob patterns to strip from history. Matched against both full path and filename. | `*.zip,*.psd,secret.key` |

> \* At least one input must be provided. Both can be used together — a blob is stripped if it matches **either** condition.

---

## Examples

**Strip all files over 10 MB from history:**
```yaml
- uses: kkkgo/Clean-Git-File-History-Action@main
  with:
    size_threshold: 10M
```

**Strip specific file types from history:**
```yaml
- uses: kkkgo/Clean-Git-File-History-Action@main
  with:
    filenames: '*.zip,*.tar.gz,*.psd'
```

**Strip by both size and filename (either condition triggers removal):**
```yaml
- uses: kkkgo/Clean-Git-File-History-Action@main
  with:
    size_threshold: 10M
    filenames: '*.xz,*.dat,*.md5sum,*.sha256sum'
```

**Strip files in a specific subdirectory:**
```yaml
- uses: kkkgo/Clean-Git-File-History-Action@main
  with:
    filenames: 'assets/*.psd,build/*.zip'
```

---

## How It Works

Git stores every version of every file as a **blob object**, uniquely identified by a SHA1 hash of its content. This action works in four steps:

1. **Scan** — walks the entire commit history with `git rev-list --objects --all` to collect every blob SHA that matches your size or filename filters
2. **Protect** — scans `HEAD` (your latest commit) and removes those blob SHAs from the delete list, so the current version of every file is always kept safe
3. **Strip** — passes the final delete list to `git filter-repo --strip-blobs-with-ids`, which rewrites the entire commit history removing those blobs from every commit's tree
4. **Push** — runs `git gc` to physically reclaim disk space, then force-pushes all branches and tags

The key insight: **blob SHAs are content-addressed**. Two files with the same content share the same SHA regardless of their name. If a blob is referenced by HEAD under any filename, it is protected. There is no risk of accidentally deleting a file because it shares a name with another — the protection is based entirely on SHA identity, not filename.

---
## Safety

- **HEAD is always protected.** The action collects the blob SHAs of all matched files currently in HEAD before running, and excludes them from the strip list. Your latest file versions are never deleted.
- **Content-addressed protection.** Protection is based on blob SHA, not filename. If two differently-named files share identical content (and thus the same SHA), protecting one protects both.
- **No false deletions from name collisions.** A blob is only stripped if its SHA appears in the delete list *and* is not present in HEAD. Filename is only used for matching — it plays no role in the actual deletion.

---

## Important Notes

>  **This rewrites git history and is irreversible.** Test on a fork first.

- All collaborators must **re-clone** the repository after this runs — existing clones have diverged history and cannot `git pull`
- GitHub may retain cached copies of deleted blobs for up to 90 days; contact GitHub Support if immediate removal is required (e.g. for leaked secrets)
- Open Pull Requests referencing rewritten commits may have broken diff views
- No extra secrets or tokens needed — uses the built-in `GITHUB_TOKEN`