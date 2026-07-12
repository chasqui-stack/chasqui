# PRP: Sprint 15.2 — Document-RAG: Knowledge Base admin page

> **Version:** 1.0
> **Created:** 2026-07-12
> **Status:** Ready
> **Executor:** Bryan (@BryanDev2023) · **Reviewer:** Willy (@willywg)
> **Series:** 2/4 — requires 15.1 merged (its endpoints are this page's API)

---

## Goal

A new **Knowledge Base** page in the admin panel (`/knowledge`): a dashed **dropzone**
(drag & drop or click-to-browse) that uploads documents, a table of documents with
**live processing states** (uploading → processing → indexed / error), delete with
confirmation, error reprocess, and a search-preview box to tune retrieval — the FAQ
page's sibling, same look and feel.

**Non-goals:** editing document content, folders/collections, bulk upload UX beyond
multi-file drop, the agent tool (15.3).

## Why

- 15.1's endpoints are operator-facing; without UI the feature doesn't exist for the
  Chasqui persona (a non-technical operator uploading a price list PDF).
- The upload-with-states pattern (optimistic row + polling) is new to the panel —
  worth doing well once; future modules will copy it.

## What

1. `src/pages/KnowledgeBasePage.tsx` + route `/knowledge` + sidebar entry (icon:
   `BookOpen` from lucide) between FAQ and Tools.
2. `src/hooks/useKnowledge.ts` — React Query hooks against
   `/admin/modules/knowledge/*` (list with conditional polling, upload multipart,
   delete, reprocess, search preview).
3. Dropzone card: dashed border, hover/drag-over highlight (amber ring — the one
   accent), accepts `.pdf,.docx,.txt,.md,.html`, multiple files (upload sequentially).
4. Documents table: filename, type badge, size (human), status badge, chunk count,
   uploaded date, row actions (reprocess when `error`, delete always).
5. States rendered from the API's `status` + a client-side `uploading` state while
   the POST is in flight. While ANY doc is `pending|processing` the list polls.
6. i18n: every string via `t()` in BOTH `en.json` and `es.json` (parity test enforces).

### Success Criteria

- [ ] Drop 2 files → both appear immediately (uploading), transition to processing
      and land on **Indexed** with chunk count, without a manual refresh.
- [ ] Oversize/unsupported/duplicate uploads surface the API error as a toast
      (translated), and the table stays consistent.
- [ ] An `error` document shows a terracotta badge, its `error_detail` on hover
      (title attr or tooltip), and a working **Reprocess** action.
- [ ] Delete asks confirmation (`ConfirmDialog` destructive) and removes the row.
- [ ] Search preview returns scored chunks with their source filename (FAQ analog).
- [ ] Polling runs ONLY while something is in flight (no eternal 2s polling).
- [ ] `npm run build`, `npm run lint`, `npm test` all pass (locales parity included).
- [ ] Zero hardcoded UI strings; zero new colors (tokens only, DESIGN.md rules).

---

## All Needed Context

### Documentation & References

```yaml
- file: admin/AGENTS.md                       # panel conventions + design rules
- file: admin/DESIGN.md                       # tokens; amber = single accent; terracotta = destructive only
- file: admin/src/pages/FaqPage.tsx           # THE blueprint: table + dialogs + search preview + toasts
- file: admin/src/hooks/useFaq.ts             # hook idiom: BASE const, queryKey, invalidate
- file: admin/src/hooks/useContacts.ts        # polling idiom: refetchInterval + { poll: true } (line ~21)
- file: admin/src/pages/ConversationDetailPage.tsx  # file input handling (onPickFile, ~line 280)
- file: admin/src/components/shared/          # ConfirmDialog, StatusBadge, SearchInput, Pagination
- file: admin/src/components/layout/Sidebar.tsx     # navigation[] array (~line 14)
- file: admin/src/router.tsx                  # route children (~line 15)
- file: admin/src/locales/en.json + es.json   # add the "knowledge" section to BOTH
- file: admin/src/locales/locales.test.ts     # the parity guard that will catch you
- file: admin/src/lib/api-client.ts           # axios + JWT; multipart just works (don't set Content-Type manually — axios does it for FormData)
```

### Key decisions

1. **Upload transport = multipart `FormData`** (`file` field), NOT the composer's
   base64-data-URI pattern — documents are bigger and the 15.1 endpoint takes
   `UploadFile`. Axios sets the multipart boundary itself; do not override headers.
2. **Conditional polling:** `useDocuments({ poll })` with
   `refetchInterval: poll ? 2500 : undefined`; the page computes
   `poll = docs.some(d => d.status === "pending" || d.status === "processing") || upload.isPending`.
   (Same shape as `INBOX_POLL_MS` in `useContacts.ts`, faster tick.)
3. **Status → UI mapping** (extend `StatusBadge` usage or inline variants):
   `uploading` (client-only, muted + spinner) · `pending`/`processing` (muted,
   pulse) · `ready` → label **Indexed** (success tint) · `error` (destructive tint).
4. **Multi-file drop uploads sequentially** (simple `for … await` over
   `dataTransfer.files`) — no parallel-upload complexity, backend dedupes anyway.
5. **No new deps.** No react-dropzone: native `onDrop`/`onDragOver` + hidden
   `<input type="file" multiple accept=".pdf,.docx,.txt,.md,.html">` is enough.
6. **Types** go in `src/types/api.ts` (`KnowledgeDocument`, `KnowledgeSearchHit`)
   mirroring 15.1's response models.

### Known Gotchas

```typescript
// 1. locales.test.ts fails the build if en.json/es.json keys diverge — add the
//    whole "knowledge" block to BOTH, with identical keys and placeholders.
// 2. onDragOver MUST call e.preventDefault() or onDrop never fires (browser opens
//    the file instead). Also guard a dragging state for the highlight styling.
// 3. Don't set { "Content-Type": "multipart/form-data" } on the axios call —
//    setting it manually drops the boundary parameter. Pass FormData bare.
// 4. Node >= 22 for dev/build (nvm use 22 — .nvmrc).
// 5. Amber is never body text on light surfaces (WCAG); use it for the drag-over
//    ring/border and dark-mode primary only. Terracotta strictly for error/destroy.
// 6. Keep table cells for facts; explanations (error_detail) go in title/tooltip.
// 7. isPending from useMutation disables the dropzone button — prevent double fire.
// 8. Invalidate ["knowledge", "documents"] after upload/delete/reprocess mutations.
```

---

## Implementation Blueprint

### Task order

```yaml
Task 1 - Types:
  - MODIFY admin/src/types/api.ts: KnowledgeDocument {id, filename, mime_type,
    size_bytes, status, error_detail, chunk_count, created_at, updated_at},
    KnowledgeSearchHit {content, similarity, document_id, filename}
Task 2 - Hooks:
  - CREATE admin/src/hooks/useKnowledge.ts
    (BASE = "/admin/modules/knowledge"; useDocuments({poll}), useUploadDocument
     (FormData), useDeleteDocument, useReprocessDocument, useKnowledgeSearch —
     mirror useFaq.ts + useContacts.ts polling)
Task 3 - Page skeleton + wiring:
  - CREATE admin/src/pages/KnowledgeBasePage.tsx
  - MODIFY admin/src/pages/index.ts (export)
  - MODIFY admin/src/router.tsx (path "knowledge")
  - MODIFY admin/src/components/layout/Sidebar.tsx (nav.knowledge, BookOpen icon)
Task 4 - Dropzone:
  - Card with border-2 border-dashed; drag-over → amber ring (ring-primary/50 in
    dark, border-primary/60); hidden multi-file input; sequential upload loop;
    per-file toast on rejection (surface API detail message translated)
Task 5 - Documents table:
  - shadcn Table like FaqPage; status badges per decision 3; size formatter
    (KB/MB); reprocess button only when status==="error"; delete → ConfirmDialog
Task 6 - Search preview:
  - Bottom card: SearchInput + button → useKnowledgeSearch; list hits with
    similarity % badge + source filename (FaqPage preview block is the mirror)
Task 7 - i18n:
  - MODIFY admin/src/locales/en.json + es.json: "knowledge" section (title,
    subtitle, dropzone hint, browse, columns, statuses, actions, deleteTitle,
    deleteConfirm, toasts, searchTitle, searchHint, empty state)
Task 8 - Tests:
  - locales parity passes by construction
  - CREATE a small component test if reasonable (e.g., status-badge mapping or a
    hook unit) following SchemaForm.test.tsx idiom — don't force a full-page test
Task 9 - Validation:
  - npm run lint && npm run build && npm test; manual e2e vs live core (below)
```

## Validation Loop

```bash
cd admin && source ~/.nvm/nvm.sh && nvm use && npm install
npm run lint && npm run build && npm test          # all green

# Manual e2e (core from 15.1 running on :8090):
npm run dev   # login → Knowledge Base page
# 1. Drop a PDF + a .md together → watch uploading → processing → Indexed
# 2. Drop a .doc → translated error toast, no ghost row
# 3. Drop the same PDF again → duplicate toast (409 detail surfaced)
# 4. Kill GOOGLE_API_KEY on core, upload → error badge → restore key → Reprocess → Indexed
# 5. Search preview: query something inside the PDF → scored chunks + filename
# 6. Delete → confirm dialog → row gone
# 7. Switch language to ES (header) → every new string translated
```

## Final Checklist

- [ ] Route + sidebar entry + page render behind ProtectedRoute like siblings
- [ ] Dropzone: drag & drop AND click-to-browse; multi-file; drag-over highlight
- [ ] Live states without manual refresh; polling stops when idle
- [ ] Error docs: terracotta badge + detail + Reprocess
- [ ] Delete with ConfirmDialog; search preview scored
- [ ] en/es complete (parity test green); no hardcoded strings; tokens only
- [ ] lint + build + test green
- [ ] PR from your fork → `chasqui-stack/admin`, branch `feat/knowledge-page`,
      title `feat: knowledge base page — upload, states, delete (Sprint 15.2)`,
      body `Closes #<admin-issue-number>`

## Anti-Patterns to Avoid

- ❌ Hardcoded UI strings or a key missing in one locale (the parity test bites)
- ❌ New hex colors / gradients — tokens only, amber is the single accent
- ❌ Manual multipart Content-Type header (boundary loss)
- ❌ Unconditional refetchInterval (eternal polling)
- ❌ Base64 data-URI upload (that's the composer's WhatsApp path, not this one)
- ❌ Direct axios calls in the page — data goes through hooks (useKnowledge.ts)
