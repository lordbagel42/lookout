# Screenshot Removal Feature Spec

Allows integrating programs (e.g. Fallout) to remove individual screenshots from a
completed session and recompile the timelapse. Editing happens via the internal API —
the integrating program builds its own UI.

---

## Scope

- Mark one or more screenshots as excluded (soft delete — rows are kept)
- Fetch the screenshot list with preview URLs so the integrating program can render a filmstrip
- Trigger a recompile after exclusions are set
- Editing is only available while the session's raw screenshots still exist in R2

Out of scope: time range trimming, speed adjustment, editing from the desktop app.

---

## Architecture Overview

```
Integrating program (e.g. Fallout)
  │
  ├── GET  /api/internal/sessions/:sessionId/screenshots
  │        → filmstrip data + preview URLs
  │
  ├── DELETE /api/internal/sessions/:sessionId/screenshots/:screenshotId
  │        → sets excluded = true on the row
  │
  └── POST /api/internal/sessions/:sessionId/recompile
           → resets session to 'stopped', enqueues compile job
           → old video stays accessible until new one overwrites it
```

All three endpoints are authenticated with the same API key mechanism used by
the existing internal routes (`requireApiKey` middleware).

---

## Database Changes

### 1. Migration: `0014_screenshot_excluded.sql`

File location: `packages/server/drizzle/0014_screenshot_excluded.sql`

```sql
ALTER TABLE screenshots ADD COLUMN excluded boolean NOT NULL DEFAULT false;
```

No index needed — the compile query already filters by `session_id` first, so a
full scan of that session's rows is negligible.

### 2. Schema: `packages/server/src/db/schema.ts`

Add to the `screenshots` table definition inside the column object:

```ts
excluded: boolean("excluded").notNull().default(false),
```

Place it after the `sampled` column for logical grouping.

---

## Server Changes

### New endpoints in `packages/server/src/routes/internal.ts`

#### `GET /api/internal/sessions/:sessionId/screenshots`

Returns all confirmed screenshots for the session, ordered by `minute_bucket` then
`captured_at`. Each row includes a preview URL — a redirect to a presigned R2 GET URL,
following the same pattern as `/api/media/:sessionId/thumbnail.jpg`.

Response shape:
```json
{
  "screenshots": [
    {
      "id": "uuid",
      "minuteBucket": 0,
      "capturedAt": "2025-01-01T12:00:00.000Z",
      "excluded": false,
      "previewUrl": "https://lookout.example.com/api/media/:sessionId/screenshots/:screenshotId.jpg"
    }
  ],
  "canEdit": true
}
```

`canEdit` is `true` when `screenshotsPurgedAt IS NULL` on the session. When false,
the preview URLs will 404 and editing is not possible.

#### `DELETE /api/internal/sessions/:sessionId/screenshots/:screenshotId`

Sets `excluded = true` on the given screenshot row. Idempotent — excluding an already
excluded screenshot returns 200.

Guards:
- Session must exist and belong to the API key's program (or any session for a global key).
- `screenshotsPurgedAt` must be NULL (i.e. R2 objects still exist). Return 409 with
  `{ "error": "Screenshots have been purged and can no longer be edited" }` otherwise.
- Screenshot must belong to the session. Return 404 if not found or mismatched.

Response: `200 { "excluded": true }`

To un-exclude (restore a frame), use a `PATCH` on the same route with body
`{ "excluded": false }` — this is optional and can be skipped if the integrating
program doesn't need undo. If implemented, it follows the same guards.

#### `POST /api/internal/sessions/:sessionId/recompile`

Triggers a fresh compilation using the current exclusion set.

Guards:
- Session status must be `complete` or `failed`. Return 409 otherwise with a
  descriptive message (e.g. `"Session is still compiling"` or `"Session has not
  finished yet"`).
- `screenshotsPurgedAt` must be NULL. Return 409 if purged.
- At least one non-excluded confirmed screenshot must remain. Return 422 with
  `{ "error": "No screenshots remain after exclusions" }` otherwise.

Steps:
1. Reset `sampled = false` on all screenshots for this session (the compile step
   re-derives which frames are sampled each run).
2. Update session: `status = 'stopped'`, `updatedAt = now()`.
3. Enqueue a `compile-timelapse` job via `boss.send(COMPILE_JOB, { sessionId })`.
4. Return `202 { "status": "compiling" }`.

The existing video URL remains valid until the worker overwrites it on successful
recompile. The worker's atomic claim (`status IN ('stopped', 'compiling')`) already
handles this correctly.

### Preview URL endpoint

Add to `packages/server/src/routes/sessions.ts` alongside the existing media proxy
endpoints (near line 1236):

```
GET /api/media/:sessionId/screenshots/:screenshotId.jpg
```

- Look up the screenshot row by `screenshotId`, verify it belongs to `sessionId`.
- Return 404 if not found or session's `screenshotsPurgedAt` is not null.
- Generate a presigned `GetObjectCommand` URL for the screenshot's `r2Key` (same
  pattern as the thumbnail/video media endpoints, same expiry).
- Redirect 302 to the presigned URL.

This endpoint does NOT require an API key — it uses the same open redirect approach
as thumbnail/video media. The UUID-based URL is not guessable.

---

## Worker Changes

### `packages/worker/src/compile.ts` — sampling query (line ~165)

Add `AND excluded = false` to the WHERE clause:

```sql
SELECT DISTINCT ON (minute_bucket) id, r2_key, minute_bucket, requested_at
FROM screenshots
WHERE session_id = ${sessionId}
  AND confirmed = true
  AND excluded = false
ORDER BY minute_bucket,
  ABS(EXTRACT(EPOCH FROM (requested_at - (
    ${session.startedAt!}::timestamptz
    + (minute_bucket * interval '1 minute')
    + interval '30 seconds'
  ))))
```

No other changes to `compile.ts` are needed. The recompile flow reuses the existing
`stopped → compiling → complete/failed` path.

---

## Migration Checklist

When running migrations, the next file is `0014_screenshot_excluded.sql`. Check
`packages/server/drizzle/` — the highest-numbered file is currently
`0013_announcements.sql`, so `0014` is correct. Drizzle Kit generates the migration
snapshot automatically via `pnpm db:generate` (see `packages/server/drizzle.config.ts`).

After adding the column to `schema.ts`, run:
```sh
cd packages/server && pnpm db:generate
# verify the generated SQL matches the hand-written migration above
pnpm db:migrate
```

---

## Integration Guide (for Fallout or other programs)

### Flow

1. User watches their completed timelapse and wants to remove a bad frame.
2. Program calls `GET /api/internal/sessions/:sessionId/screenshots` to fetch the
   filmstrip. Display the `previewUrl` images in a grid.
3. User clicks a frame to remove it. Program calls
   `DELETE /api/internal/sessions/:sessionId/screenshots/:screenshotId`.
4. After the user is done removing frames, they click "Recompile".
5. Program calls `POST /api/internal/sessions/:sessionId/recompile`.
6. Program polls `GET /api/internal/sessions/:sessionId` (existing endpoint) until
   `session.status === 'complete'`. The existing `videoUrl` field updates when done.

### Rendering previews

The `previewUrl` from step 2 is a redirect URL. Render it as a standard `<img src>` —
the browser follows the redirect automatically. Do not cache the URL itself; it is a
redirect to a short-lived presigned URL. Re-fetch the screenshots list if you need
to display the filmstrip again after a delay.

### Canary: `canEdit`

Always check `canEdit` before rendering the editing UI. If false, the session's R2
objects have been purged by the retention job and editing is not possible. Show a
message explaining that the session is too old to edit.

---

## What is NOT changed

- The desktop app (`clients/desktop/`) — editing is intentionally internal-API only.
- The React SDK (`clients/react/`) — no changes needed.
- Session status enum — no new values. Recompile reuses `stopped → compiling → complete`.
- Tracked seconds / credit mode accounting — screenshot exclusion does not affect the
  already-credited time. Exclusion is purely a video content choice.
- Retention job — excluded screenshots are not deleted any sooner. The row (and its
  R2 object) lives until the normal purge window.
