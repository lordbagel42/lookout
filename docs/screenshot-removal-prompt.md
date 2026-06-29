# AI Implementation Prompt: Screenshot Removal

Use this prompt verbatim (or lightly adapted) to ask an AI to implement the
screenshot removal feature described in `docs/screenshot-removal.md`.

---

## Prompt

Implement the screenshot removal feature for the Lookout timelapse service. The full
spec is at `docs/screenshot-removal.md` — read it first, then implement everything
described. Here is a summary of what needs to change and where.

### What this feature does

Integrating programs (like Fallout) can:
1. Fetch a list of a session's screenshots with preview URLs.
2. Mark individual screenshots as excluded (soft-delete).
3. Trigger a recompile of the timelapse, which skips excluded frames.

Editing happens via the internal API only. No changes to the desktop app or React SDK.

### Files to change

**1. `packages/server/src/db/schema.ts`**

Add `excluded: boolean("excluded").notNull().default(false)` to the `screenshots`
table, after the `sampled` column.

**2. `packages/server/drizzle/0014_screenshot_excluded.sql`** (new file)

```sql
ALTER TABLE screenshots ADD COLUMN excluded boolean NOT NULL DEFAULT false;
```

After creating this file, also run `pnpm db:generate` inside `packages/server` to
regenerate the Drizzle snapshot (it will produce a matching migration file — confirm
the SQL matches, keep both).

**3. `packages/server/src/routes/internal.ts`**

Add three new routes inside `internalRoutes()`. All are guarded by the existing
`requireApiKey` hook (already applied to the whole function). Use the existing
patterns in this file for DB queries (Drizzle ORM), error responses, and param
validation.

Route A — `GET /api/internal/sessions/:sessionId/screenshots`

- Query: all screenshots WHERE `session_id = :sessionId AND confirmed = true`,
  ordered by `minute_bucket ASC, captured_at ASC`.
- For each screenshot, compute a `previewUrl`:
  `${baseUrl}/api/media/${sessionId}/screenshots/${screenshot.id}.jpg`
  (this is a redirect endpoint you will add to sessions.ts — see below).
- Return `canEdit: session.screenshotsPurgedAt === null`.
- Return shape:
  ```json
  { "screenshots": [{ "id", "minuteBucket", "capturedAt", "excluded", "previewUrl" }], "canEdit": true }
  ```

Route B — `DELETE /api/internal/sessions/:sessionId/screenshots/:screenshotId`

- Params: `sessionId` (uuid), `screenshotId` (uuid).
- Guard: return 404 if session not found, 409 if `screenshotsPurgedAt` is not null,
  404 if screenshot not found or doesn't belong to this session.
- Action: `UPDATE screenshots SET excluded = true WHERE id = :screenshotId`.
- Return `200 { "excluded": true }`.

Route C — `POST /api/internal/sessions/:sessionId/recompile`

- Guard: return 404 if session not found.
- Guard: return 409 if `screenshotsPurgedAt` is not null
  (`{ error: "Screenshots have been purged and can no longer be edited" }`).
- Guard: return 409 if `session.status` is not `'complete'` and not `'failed'`
  (`{ error: "Session cannot be recompiled in its current status" }`).
- Guard: count non-excluded confirmed screenshots. If 0, return 422
  (`{ error: "No screenshots remain after exclusions" }`).
- Action:
  1. Reset `sampled = false` on ALL screenshots for this session.
  2. Update session: `status = 'stopped', updatedAt = new Date()`.
  3. `await boss.send(COMPILE_JOB, { sessionId })` — `boss` and `COMPILE_JOB` are
     already imported at the top of the file.
- Return `202 { "status": "compiling" }`.

**4. `packages/server/src/routes/sessions.ts`**

Add a new media proxy endpoint near the existing `/api/media/:sessionId/thumbnail.jpg`
and `/api/media/:sessionId/video.mp4` routes (around line 1236):

```
GET /api/media/:sessionId/screenshots/:screenshotId.jpg
```

- Look up the screenshot row by `screenshotId`, confirm it belongs to `sessionId`.
- Return 404 if not found.
- Look up the session; return 404 if `screenshotsPurgedAt` is not null.
- Generate a presigned `GetObjectCommand` URL for `screenshot.r2Key` (use the same
  `getSignedUrl` call and expiry already used by the thumbnail endpoint).
- `return reply.redirect(302, presignedUrl)`.

This endpoint does NOT require an API key (same as thumbnail/video media endpoints).

**5. `packages/worker/src/compile.ts`**

In the `compileTimelapse` function, find the sampling query (around line 165). It
currently reads:

```sql
WHERE session_id = ${sessionId} AND confirmed = true
```

Change it to:

```sql
WHERE session_id = ${sessionId} AND confirmed = true AND excluded = false
```

That is the only change needed in this file.

### Patterns to follow

- Drizzle ORM is used throughout — use `db.update(schema.screenshots).set(...).where(...)`
  for mutations, `db.query.screenshots.findFirst(...)` or `db.select(...).from(...)` for
  reads.
- Error responses always use `reply.code(N).send({ error: "..." })`.
- Fastify route generics: `app.get<{ Params: { sessionId: string } }>("/path", { schema: ... }, handler)`.
- UUID params use `{ type: "string", format: "uuid" }` in the JSON schema, matching the
  pattern already used for `sessionIdParamSchema` at the top of `internal.ts`.
- The `baseUrl` for building preview URLs is `process.env.BASE_URL || "http://localhost:3000"`.

### What NOT to change

- `clients/desktop/` — no changes.
- `clients/react/` — no changes.
- The session status enum in `schema.ts` — do not add new values.
- Tracked-seconds or credit-mode accounting — exclusion does not affect credited time.

### Verification

After implementing, verify:
1. TypeScript compiles: `pnpm --filter @lookout/server build` and
   `pnpm --filter @lookout/worker build`.
2. The migration SQL file exists and is valid.
3. The Drizzle snapshot is regenerated (`pnpm db:generate` in `packages/server`).
4. The three new internal routes appear in the server and match the spec.
5. The preview redirect route exists in sessions.ts.
6. The compile query in `compile.ts` includes `AND excluded = false`.
