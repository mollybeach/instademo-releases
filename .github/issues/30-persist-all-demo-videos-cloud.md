# Persist all demo videos to Supabase and propagate deletes to the cloud

**Priority:** P1 — recordings must survive across devices / reinstalls
**Area:** `Services/CloudProjectSync.swift`, `Services/SupabaseStorageService.swift`, `Services/DemoVideoDeletion.swift`, `Views/AutoExploreView.swift`

---

## Problem

Demo recordings are only partially synced to Supabase:

- **Auto Explore recordings stay local-only.** When an Auto Explore run finishes,
  the resulting video is written to disk but never uploaded to Storage or recorded
  in the `recordings` table.
- **Deletion does not propagate.** Trashing a local demo removes the file on disk
  but leaves the Storage object and the `recordings` row behind, so the cloud
  accumulates orphans and other devices still see the deleted demo.

Net effect: "My Demos" is not a faithful, durable record of the user's demos.

## Proposed work

- Add `SupabaseStorageService.delete(bucket:path:)` (treat HTTP 404 as success).
- Add `CloudProjectSync.deleteRecording(localURL:)` that removes both the Storage
  object and the matching `recordings` row.
- Wire `CloudProjectSync.deleteRecording` into `DemoVideoDeletion.trash` so a local
  delete also clears the cloud copy.
- Upload Auto Explore recordings on finish (observe `runner.recordingURL` in
  `AutoExploreView` and call the existing completed-recording cloud sync).
- Ensure the live `recordings` table has the columns the upload path writes
  (`storage_path`, `display_name`).

## Acceptance criteria

- [ ] Auto Explore recordings appear in Storage + the `recordings` table after a run.
- [ ] Deleting a demo locally removes its Storage object and `recordings` row.
- [ ] No orphaned Storage objects/rows remain after a delete.
- [ ] `supabase/schema.sql` reflects the columns the app writes.
