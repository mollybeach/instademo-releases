# My Demos should reflect the Supabase recordings table, not just local disk

**Priority:** P1 — gallery should show cloud demos before files are downloaded
**Area:** `Views/MyDemosView.swift`, `Services/CloudProjectSync.swift`

---

## Problem

"My Demos" only scans the local recordings folder. Demos recorded on another
device (or lost on reinstall) are invisible until their files happen to exist on
disk, so the gallery does not reflect the user's real cloud library.

## Proposed work

- Add `CloudProjectSync.fetchAllRecordings()` returning the user's rows from the
  `recordings` table (ordered by `created_at` desc).
- In `MyDemosView.refresh()`, scan local recordings and **merge** them with the
  cloud rows:
  - de-duplicate local vs cloud entries,
  - annotate local entries with cloud metadata,
  - surface cloud-only entries (with a `storageRef` + `cloudDurationSec`).
- Show an "in cloud, not downloaded" affordance (e.g. `icloud.and.arrow.down`
  badge) and an `existsLocally` flag on the demo model.
- On-demand download: wrap play / reveal / share / export in an
  `ensureDownloaded()` / `withLocalCopy()` helper that fetches the file first and
  shows a download HUD.
- Deletion of a cloud-only item should remove the cloud row + object directly
  (no local-file trash attempt).

## Acceptance criteria

- [ ] Gallery lists demos from the `recordings` table even when no local file exists.
- [ ] Cloud-only items are visually distinguished and download on first use.
- [ ] Play/reveal/share/export work for cloud-only items via on-demand download.
- [ ] Deleting a cloud-only item removes its row and Storage object.
