# Notarized DMG release crashes on publish and targets the wrong repo

**Priority:** P0 — release pipeline cannot publish the download
**Area:** `scripts/dist-direct.sh`, `.github/workflows/release-dmg.yml`

---

## Problem

The "Release DMG" job builds, notarizes, and staples `InstaDemo-<date>.dmg`
successfully, then aborts on the final publish step:

```text
./scripts/dist-direct.sh: line 122: RELEASE_TAG: unbound variable
Error: Process completed with exit code 1.
```

Two distinct bugs in the publish block:

1. **Non-ASCII char glued to a variable.** The publish `echo` line ended with a
   Unicode ellipsis (`…`) immediately after `$RELEASE_TAG`. Under `set -u`,
   bash reads the variable name as `RELEASE_TAG…` and aborts. (Same class of bug
   as the earlier `RELEASE_REPO…` failure.)

2. **Publish ignores `RELEASE_REPO`.** The workflow passes
   `RELEASE_REPO=mollybeach/instademo-releases` and a `RELEASES_TOKEN` PAT scoped
   to that repo, and the website "Download for macOS" button is built from
   `${RELEASE_REPO}`. But `dist-direct.sh` never references `RELEASE_REPO` — its
   `gh release create/upload` calls target the **current** repo
   (`mollybeach/instademo`) and the printed download URL is hardcoded to it. The
   release lands in the wrong place (or fails auth), so the button points at a
   non-existent asset.

## Proposed fix

- Brace-wrap variables and use ASCII (`${RELEASE_TAG}`, `...`) in the publish block.
- Honor `RELEASE_REPO` for all `gh release` calls and the download URL, defaulting
  to the current repo when unset:

```bash
REPO="${RELEASE_REPO:-mollybeach/instademo}"
gh release view   "$RELEASE_TAG" --repo "$REPO" >/dev/null 2>&1 \
  || gh release create "$RELEASE_TAG" --repo "$REPO" --title "$RELEASE_TAG" --notes "InstaDemo $APP_VERSION"
gh release upload "$RELEASE_TAG" "$DMG_PATH" --repo "$REPO" --clobber
```

- Sweep `dist-direct.sh` for any other non-ASCII characters adjacent to variable
  expansions.

## Acceptance criteria

- [ ] `dist-direct.sh` passes `bash -n` and contains no non-ASCII chars touching `$vars`.
- [ ] The DMG is published to `mollybeach/instademo-releases` under the computed tag.
- [ ] The printed/website download URL matches the repo the asset was uploaded to.
- [ ] The Release DMG workflow completes green end-to-end.
