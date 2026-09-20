# MarkVault — Development Plan
## From Static Prototype to Vercel-Deployable Multi-Client App

### 1. Current State

- Single `index.html` prototype.
- Features: upload `.md`, file list + search, `---` slide splitting, Prev/Next + dots + keyboard, fullscreen present, collapsible sidebar, dark theme, `localStorage` persistence.
- Limitations:
  - No backend. Data lives in one browser only.
  - No users, no sharing. Cannot open same deck on classroom PC + personal laptop.
  - No real file storage. Large decks hit `localStorage` quota (~5 MB).
  - Single-client only. No sync, no present links, no access control.

This is correct for a prototype. Do not add more features to the static file. Freeze it and migrate.

### 2. Target Vision

A teacher can:
1. Log in from any device.
2. Upload `.md` decks once.
3. Open, present, and share them in class via link.
4. Work offline-tolerant, load fast on projectors.

Multi-client means: same account + same decks accessible from multiple browsers/devices concurrently, with consistent state.

### 3. Recommended Stack (Vercel-Native)

Keep it boring and deployable:

- Frontend + Backend: Next.js 14+ (App Router) on Vercel.
  - Why: Vercel first-class, SSR for fast first load, API Routes / Server Actions built in, no separate server to manage.
- Language: TypeScript.
- Styling: keep current CSS or Tailwind. Do not rewrite UI yet, port it.
- Markdown: `marked` + `highlight.js` (already used) or `remark`/`rehype` if you want sanitization. Add `DOMPurify` for rendering untrusted markdown.
- Auth: Auth.js (NextAuth) with Google + email OTP. Classrooms need low-friction login.
- Database: Vercel Postgres (Neon) + Prisma/Drizzle.
  - Alternative: Supabase Postgres if you want built-in auth + storage in one.
- File storage: Vercel Blob for raw `.md` files. Store metadata + parsed slides in Postgres, raw file in Blob.
- Cache / Realtime (optional later): Vercel KV (Upstash) for present-session state.

Do not start with microservices, Docker, or custom servers. They fight Vercel's model.

### 4. Architecture

```
Browser (Teacher)  ──┐
                      ├──> Vercel Edge (Next.js)
Browser (Classroom) ──┘         ├──> Auth.js (session)
                                ├──> API Routes / Server Actions
                                ├──> Postgres (users, decks, shares)
                                └──> Vercel Blob (raw .md)
```

- Client is stateless. `localStorage` becomes cache only, Postgres is source of truth.
- Present mode is read-only render of `decks -> slides[]`.
- Share links are signed tokens, not auth-bypass of full account.

### 5. Data Model (Minimal)

```
User(id, email, name, createdAt)
Deck(id, ownerId, name, blobUrl, contentHash, slideCount, createdAt, updatedAt)
Share(id, deckId, token, expiresAt, allowDownload)
```

- Do not store per-slide rows in v1. Store full markdown, split to slides on render (`splitSlides` logic ports directly). Add slide cache later if needed.
- `contentHash` enables dedupe + "already uploaded" detection.
- Keep `uploadedAt`, `size`, `wordCount` as derived metadata.

### 6. Phased Roadmap

**Phase 0 — Freeze (1 day)**
- Lock current `index.html` as `prototype-v1`. No new UI tweaks.
- Extract pure functions to reuse: `splitSlides`, slide render, keyboard nav.

**Phase 1 — Next.js Migration (2-4 days)**
- `npx create-next-app@latest markvault --typescript --app`
- Port UI 1:1 to `/app/page.tsx`: header, sidebar, stage, resizer.
- Port `splitSlides` + `marked` rendering as client components.
- Deploy empty shell to Vercel to validate pipeline. No DB yet, still `localStorage`.

**Phase 2 — Persistence API (3-5 days)**
- Add Postgres + Prisma schema for `User`, `Deck`.
- API:
  - `POST /api/decks` (upload, validate .md, 2 MB limit, store to Blob + DB)
  - `GET /api/decks` (list)
  - `GET /api/decks/:id` (get + render)
  - `DELETE /api/decks/:id`
- Replace `localStorage` reads with `fetch` + optimistic cache. Keep `localStorage` as offline fallback only.

**Phase 3 — Auth + Multi-Client (4-6 days)**
- Add Auth.js. Protect `/api/decks/*` by `ownerId = session.user.id`.
- Test: upload on laptop, present on classroom PC. Same account, same list.
- Handle concurrency: last-write-wins for v1, `updatedAt` conflict toast if stale. No OT/CRDT yet.

**Phase 4 — Classroom Sharing (3-4 days)**
- `POST /api/decks/:id/share` generates `/p/:token` public present-only page.
- Public page: no sidebar, no upload, only slides + fullscreen. Ideal for projector or student link.
- Optional expiry + disable download.

**Phase 5 — Harden for Deploy (2-3 days)**
- Input validation (Zod), file type + size limits server-side, HTML sanitization.
- Rate limiting, error boundaries, loading/empty states.
- Lighthouse pass: code-split `highlight.js` languages, preload first slide.
- `vercel.json`, env vars, preview + production branches.

Total realistic solo scope: 3-4 weeks part-time for MVP.

### 7. Multi-Client Specifics

1. Source of truth is server, not browser. Clients poll or revalidate on focus (`SWR` or Server Actions revalidation), no websockets in v1.
2. Offline: classroom WiFi fails. Keep service-worker cache of last opened deck + present page static generation where possible.
3. Present isolation: presenter view (with sidebar + notes later) vs audience view (`/p/:token` clean). Build this split early, it avoids rewrites.
4. Conflict policy: decks are replace-on-upload in v1. Rename is metadata-only PATCH. No live co-editing until v2.
5. Security: never expose Blob URLs directly long-lived. Use short-lived signed URLs or proxy via API with auth check.

### 8. Vercel Deployment Checklist

- [ ] Next.js app builds with `next build` locally with no errors.
- [ ] Env: `DATABASE_URL`, `BLOB_READ_WRITE_TOKEN`, `NEXTAUTH_SECRET`, `NEXTAUTH_URL`.
- [ ] Prisma migrate on deploy (`vercel-build` hook or Vercel Postgres integration).
- [ ] `POST /api/decks` max body configured (Vercel 4.5 MB serverless limit — enforce 2 MB client + server).
- [ ] Custom domain + HTTPS, preview deploys for PRs.
- [ ] Logging: Vercel logs + Sentry for client parse errors.

### 9. Opinion

Do not evolve the static file into a backend by bolting on Firebase snippets or PHP. Port it once to Next.js and delete the old path.

Keep v1 deliberately thin: auth + upload/list/present/share. Resist editor, collaboration, folders, and themes until teachers actually request them. Your current prototype already proves the core interaction (upload md -> slides -> fullscreen). The risk is not UI, it is state sync across devices. Solve that with Postgres + Blob + Auth and you have a deployable product.

If you want lowest effort alternative: keep frontend static on Vercel + Supabase (auth + Postgres + storage) with no Next.js API. Faster, but you will rewrite to Next.js anyway once share links and SSR matter. Start with Next.js.

### 10. Immediate Next Action

1. Init Next.js repo in `MarkVault-v2/`, deploy blank to Vercel.
2. Copy `splitSlides` + slide CSS verbatim.
3. Implement `GET/POST /api/decks` with Vercel Postgres + Blob.
4. Add Google login, lock API to owner.
5. Ship `/p/:token` read-only present page for class use.
