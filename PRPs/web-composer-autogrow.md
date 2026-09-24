# PRP: Web widget — auto-growing composer (WhatsApp-style)

> **Version:** 1.0
> **Created:** 2026-09-24
> **Status:** Completed (web#1, merged 2026-09-24)
> **Service:** `web/` (`chasqui-stack/web`) · **Branch:** `feat/composer-autogrow` (already created in the `web/` submodule, off `main` @ `690dafd`)
> **Executor:** Cursor (Grok 4.7) — read this whole file before touching code.

---

## Goal

The web widget's composer is a single-line `<input type="text">`. A long
message scrolls horizontally inside a pill and the visitor can't see what they
wrote (screenshot: *"Gracia por tu ayuda, fue muy ut…"* cut off).

Replace it with an **auto-growing `<textarea>`** that behaves like WhatsApp:

| Content | Behavior |
|---|---|
| Short text | One line, same height and pill look as today (40px). |
| Longer text | Wraps and **grows line by line** so all text is visible. |
| Too much text | Stops growing at a **max height (6 lines ≈ 140px)**, then scrolls **vertically** inside the field. |
| Send / clear | Goes back to one line. |

Buttons (attach, mic, send) stay **anchored to the bottom** of the composer
as it grows (WhatsApp keeps `+` and send at the bottom row).

**Non-goals:** CSS `field-sizing: content` (not in Firefox, only recent
Safari — may come later as progressive enhancement), the iOS "<16px font
zooms on focus" quirk, a character limit/counter, rich text, changing how the
message is sent (the payload is still `{ type: 'text', text }`).

## Why

- Visitors write long messages (contact details, explanations). Hiding the
  text makes typos easy to miss and feels broken next to WhatsApp/Telegram.
- Newlines: with a textarea, **Shift+Enter** inserts a line break, so a
  visitor can write multi-paragraph messages. The bubbles already render
  `white-space: pre-wrap` + canonical Markdown (ADR-007), so newlines display
  correctly with no backend change.
- UI-only change. No contract, gateway, or core change (the core never knows
  a channel exists — AGENTS.md).

## What

### Success criteria

- [ ] The composer field is a `<textarea class="text" rows="1">`, **not** an `<input>`.
- [ ] Empty or short text: height 40px, pill look (radius 20px), same visuals as today.
- [ ] Each wrapped line or newline grows the field by one line (20px) until 140px.
- [ ] Past 140px the field scrolls vertically (thin scrollbar); it never scrolls horizontally.
- [ ] **Enter** sends. **Shift+Enter** inserts a newline and does **not** send.
- [ ] Enter while an IME composition is active (accents/dead keys, CJK) does **not** send.
- [ ] After Send (text or image + caption) the field goes back to 40px.
- [ ] Pasting a long text grows the field right away.
- [ ] Attach / mic / send stay bottom-aligned while the field grows. With one line they look centered, same as today.
- [ ] If the log was scrolled to the bottom, it stays at the bottom when the composer grows (the last message is not hidden behind the taller composer).
- [ ] The `waiting` lock still disables the textarea (the `locked` logic does not change).
- [ ] `npm run typecheck`, `npm test`, `npm run build` pass. New tests cover the items above.
- [ ] README feature line mentions the multi-line composer.

---

## All Needed Context

### Documentation & References

```yaml
- file: web/AGENTS.md
  why: Service role, layout (src/widget/*), dev commands, Shadow-DOM isolation rule.
- file: web/src/widget/ui.tsx
  why: THE file. Composer JSX is at the bottom of App(); `input` state, sendComposed(), logRef scroll effect.
- file: web/src/widget/styles.ts
  why: Shadow-DOM CSS string. `.composer` and `.composer .text` rules change. Note the global
       `:host, * { box-sizing: border-box; }` at the top (matters for the height math).
- file: web/tests/widget/ui.test.tsx
  why: Test harness (fakeApi, mount, openPanel, typeText, clickSend). Helpers cast to
       HTMLInputElement — they must become HTMLTextAreaElement.
- file: docs/design/adr-011-web-channel.md
  why: Widget constraints (Preact via preact/compat, Shadow DOM, zero host deps).
- url: https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/isComposing
  why: IME guard for Enter.
- url: https://developer.mozilla.org/en-US/docs/Web/CSS/field-sizing
  why: Why we do NOT rely on it (browser support) — JS resize is the implementation.
```

### Current code (what exists today)

`web/src/widget/ui.tsx` — the composer field:

```tsx
<input
  class="text"
  type="text"
  placeholder="Type a message…"
  value={input}
  disabled={locked}
  onInput={(e) => setInput((e.currentTarget as HTMLInputElement).value)}
  onKeyDown={(e) => {
    if (e.key === 'Enter') void sendComposed()
  }}
/>
```

`web/src/widget/styles.ts` — relevant rules:

```css
.composer {
  min-height: 60px; background: #FFFFFF; border-top: 1px solid #E7E5E4;
  padding: 8px 12px; display: flex; align-items: center; gap: 8px;
}
.composer .icon-btn { width: 32px; height: 32px; flex: none; ... }
.composer .text {
  flex: 1; min-width: 0; border: none; outline: none; background: #F5F5F4;
  padding: 10px 14px; border-radius: 999px; font-size: 14px; font-family: inherit; color: #1C1917;
}
.composer .text:disabled { opacity: .35; cursor: not-allowed; }
.composer .send { width: 36px; height: 36px; flex: none; ... }
```

`sendComposed()` already calls `setInput('')` on both paths (text, and image +
caption). The resize must react to that **programmatic** clear, not only to
user `input` events → drive the resize from a layout effect on `[input]`.

The log auto-scroll effect already exists:

```tsx
useEffect(() => {
  if (logRef.current) logRef.current.scrollTop = logRef.current.scrollHeight
}, [messages, waiting, open])
```

### Desired structure (files to modify)

```
web/
├── src/widget/ui.tsx        # MODIFY: <input> → <textarea>, textRef, autosize layout effect,
│                            #         Enter/Shift+Enter/IME keydown, keep log pinned to bottom
├── src/widget/styles.ts     # MODIFY: .composer align-items:flex-end; .composer .text textarea rules
├── tests/widget/ui.test.tsx # MODIFY: helpers → HTMLTextAreaElement; ADD autogrow + keyboard tests
└── README.md                # MODIFY: feature sentence mentions the multi-line, auto-growing composer
```

No new files. No new dependencies.

### Known gotchas & project conventions

```text
1. English-only codebase: code, comments, test names, aria labels (AGENTS.md parent rule).
   The placeholder stays "Type a message…".

2. Preact, not React. Hooks come from 'preact/hooks' (useLayoutEffect is exported there).
   JSX uses `class`, not `className` (see the existing file). Keep that style.

3. Shadow DOM: all CSS lives in the `styles` template string in styles.ts. Don't add
   inline <style> or external CSS. Inline `style` for the computed height is fine
   (el.style.height), same as the existing `style="display:none"` on the file input.

4. box-sizing is border-box globally (`:host, * {...}`). The textarea has no border, so
   `el.style.height = el.scrollHeight + 'px'` is exact (scrollHeight includes padding).
   If you ever add a border, add it to the height.

5. Resize algorithm MUST reset first: `el.style.height = 'auto'` and THEN read
   scrollHeight. Without the reset the field never shrinks (scrollHeight >= current height).

6. Cap via CSS `max-height: 140px` + `overflow-y: auto`. JS sets
   `height = min(scrollHeight, MAX)`. Toggle `overflowY` to 'hidden' under the cap and 'auto'
   at the cap. Otherwise some browsers flash a scrollbar while growing.

7. The programmatic clear (setInput('')) fires NO input event → resize from
   `useLayoutEffect(() => autosize(), [input])`, not only from onInput. useLayoutEffect (not
   useEffect) avoids a one-frame flash at the old height.

8. Enter handling on a textarea:
     if (e.key === 'Enter' && !e.shiftKey && !e.isComposing && e.keyCode !== 229) {
       e.preventDefault()   // otherwise a stray "\n" lands in the field
       void sendComposed()
     }
   keyCode 229 = Safari's IME signal (isComposing can be false there on the final Enter).

9. The panel is always mounted (`.panel` toggles visibility). While hidden with
   `visibility:hidden` it IS laid out, so scrollHeight is valid. Still add `open` to the
   effect deps so the height is right on open.

10. Keep the log pinned: the log is `flex:1`, so when the composer grows the log shrinks and
   the last bubble can slip under the fold. In autosize(): read "was at bottom"
   (log.scrollHeight - log.scrollTop - log.clientHeight < 8) BEFORE resizing; after
   resizing, if it was, set log.scrollTop = log.scrollHeight. If the visitor scrolled up to
   read history, don't touch it.

11. jsdom does no layout: scrollHeight/clientHeight are 0. Tests must stub them with
   Object.defineProperty(el, 'scrollHeight', { configurable: true, get: () => N })
   and then assert on el.style.height. Don't try to test real wrapping in jsdom. That's
   the manual Level 4 check.

12. jsdom KeyboardEvent supports { key, shiftKey, isComposing } in the init dict. Dispatch
   with `new KeyboardEvent('keydown', { key: 'Enter', bubbles: true, ... })`. Preact
   listens to keydown on the element, so dispatching on the textarea works.

13. Existing tests read `.value` and `.disabled` on `.composer .text`. Those still work on a
   textarea, but the casts must change to HTMLTextAreaElement (typecheck is part of the gate).

14. `white-space` on the textarea: default (pre-wrap) is correct. Do NOT copy `.msg` rules.
   Add `overflow-wrap: anywhere` so a long URL/email without spaces wraps instead of
   forcing horizontal scroll.
```

---

## Implementation Blueprint

### Constants (top of `ui.tsx`, next to `WAITING_TIMEOUT_MS`)

```ts
/** Composer grows line by line up to this height (≈6 lines), then scrolls. Keep in sync with styles.ts. */
const COMPOSER_MAX_HEIGHT_PX = 140
```

### Tasks (in execution order)

**Task 1 — Styles (`src/widget/styles.ts`)**

Replace the `.composer` and `.composer .text` rules. Everything else stays.

```css
.composer {
  min-height: 60px; background: #FFFFFF; border-top: 1px solid #E7E5E4;
  padding: 10px 12px; display: flex; align-items: flex-end; gap: 8px;
}
/* Bottom-anchored buttons: with a 40px single-line field, these margins center
   them on the row exactly like the old align-items:center. */
.composer .icon-btn { margin-bottom: 4px; }   /* 32px tall → (40-32)/2 */
.composer .send { margin-bottom: 2px; }       /* 36px tall → (40-36)/2 */

.composer .text {
  flex: 1; min-width: 0; display: block; margin: 0;
  border: none; outline: none; resize: none; background: #F5F5F4;
  padding: 10px 14px; border-radius: 20px;
  font-size: 14px; line-height: 20px; font-family: inherit; color: #1C1917;
  height: 40px; min-height: 40px; max-height: 140px;   /* keep in sync with COMPOSER_MAX_HEIGHT_PX */
  overflow-y: hidden; overflow-x: hidden; overflow-wrap: anywhere;
  scrollbar-width: thin; scrollbar-color: #D6D3D1 transparent;
}
.composer .text::placeholder { color: #A8A29E; }
.composer .text:disabled { opacity: .35; cursor: not-allowed; }
```

Merge `margin-bottom` into the existing `.composer .icon-btn` / `.composer .send`
rules instead of adding duplicate selectors. Don't add rules that already exist.
Note the vertical padding moves 8px → 10px so the single-line composer stays
60px tall (10 + 40 + 10). Check `.composer .attach` (a `<label class="icon-btn">`)
gets the same margin (it does, through `.icon-btn`).

Why radius 20px and not 999px: at 40px height, 20px *is* a pill, and when it
grows it becomes a rounded rectangle (WhatsApp look). 999px makes a tall
field look like a capsule/oval.

**Task 2 — Component (`src/widget/ui.tsx`)**

1. Import `useLayoutEffect` from `'preact/hooks'`.
2. Add `const textRef = useRef<HTMLTextAreaElement>(null)` next to `logRef`.
3. Add the autosize routine + effect (put it near the existing log-scroll effect):

```tsx
/** Grow the composer to fit its content up to COMPOSER_MAX_HEIGHT_PX, then scroll (WhatsApp-style). */
function autosizeComposer() {
  const el = textRef.current
  if (!el) return
  const log = logRef.current
  const pinned = !!log && log.scrollHeight - log.scrollTop - log.clientHeight < 8
  el.style.height = 'auto'
  const next = Math.min(el.scrollHeight, COMPOSER_MAX_HEIGHT_PX)
  el.style.height = `${next}px`
  el.style.overflowY = el.scrollHeight > COMPOSER_MAX_HEIGHT_PX ? 'auto' : 'hidden'
  if (pinned && log) log.scrollTop = log.scrollHeight
}

useLayoutEffect(() => {
  autosizeComposer()
}, [input, open])
```

   `scrollHeight` of 0 (jsdom, or not laid out) gives `height: 0px`. The CSS
   `min-height: 40px` covers it visually. That's fine and tests rely on it
   (they stub scrollHeight).

4. Replace the `<input … />` with:

```tsx
<textarea
  ref={textRef}
  class="text"
  rows={1}
  placeholder="Type a message…"
  aria-label="Message"
  value={input}
  disabled={locked}
  onInput={(e) => setInput((e.currentTarget as HTMLTextAreaElement).value)}
  onKeyDown={(e) => {
    // Enter sends; Shift+Enter is a newline; never send mid-IME composition (229 = Safari IME).
    if (e.key === 'Enter' && !e.shiftKey && !e.isComposing && e.keyCode !== 229) {
      e.preventDefault()
      void sendComposed()
    }
  }}
/>
```

5. Don't change `sendComposed()`: `input.trim()` already drops trailing
   newlines and whitespace, and embedded newlines are kept in `text` on purpose.
   Check that a message of only newlines (`"\n\n"`) is **not** sent. It trims to
   `''` and hits the existing `if (!text) return`.

**Task 3 — Tests (`tests/widget/ui.test.tsx`)**

1. Change the helper casts: `typeText` uses `HTMLTextAreaElement`, and every
   `querySelector('.composer .text') as HTMLInputElement` becomes
   `HTMLTextAreaElement` (4 places).
2. Add helpers:

```ts
function composer(root: HTMLElement): HTMLTextAreaElement {
  return root.querySelector('.composer .text') as HTMLTextAreaElement
}

/** jsdom does no layout — fake the content height the browser would report. */
function stubScrollHeight(el: HTMLElement, px: number): void {
  Object.defineProperty(el, 'scrollHeight', { configurable: true, get: () => px })
}

async function pressKey(el: HTMLElement, init: KeyboardEventInit): Promise<void> {
  await act(async () => {
    el.dispatchEvent(new KeyboardEvent('keydown', { bubbles: true, cancelable: true, ...init }))
  })
}
```

3. New tests (inside the existing `describe('widget App')`):

   - **`renders the composer as a single-row textarea`**: `composer(root).tagName === 'TEXTAREA'`, `rows === 1`.
   - **`sends on Enter`**: type `'hola'`, `pressKey(ta, { key: 'Enter' })` → `api.send` called once with `{ type: 'text', text: 'hola' }`.
   - **`inserts a newline on Shift+Enter instead of sending`**: type, `pressKey(ta, { key: 'Enter', shiftKey: true })` → `api.send` not called.
   - **`does not send on Enter during IME composition`**: `pressKey(ta, { key: 'Enter', isComposing: true })` → not called.
   - **`sends multi-line text with its newlines preserved`**: type `'line 1\nline 2'`, click send → payload `text === 'line 1\nline 2'`.
   - **`does not send a message made only of newlines`**: type `'\n\n'`, click send → not called.
   - **`grows with content up to the max height, then scrolls`**:
     stub scrollHeight 80, type → `style.height === '80px'`, `style.overflowY === 'hidden'`;
     stub 400, type more → `style.height === '140px'`, `style.overflowY === 'auto'`.
   - **`shrinks back after sending`**: stub 100, type → `'100px'`; stub 40, click send → `'40px'`
     (this proves the resize runs on the programmatic `setInput('')`).
   - Keep all existing tests green, including the lock test (`input.disabled` → textarea).

   Stub **before** `typeText` because the layout effect reads scrollHeight when
   `input` changes.

**Task 4 — Docs (`README.md`)**

In the "What the visitor gets" paragraph, add *"a multi-line composer that
grows as you type (Enter sends, Shift+Enter for a new line)"*. One sentence.
No ADR: this is a UI detail, not an architectural decision.

**Task 5 — Build the bundle**

`npm run build` regenerates `dist/widget.js`. `dist/` is **gitignored**, so
don't commit it. The build only checks that the bundle compiles and its size.

---

## Validation Loop

Run from `web/`.

### Level 1: Types

```bash
npm run typecheck
```

### Level 2: Unit tests

```bash
npm test
```

Expected: the whole suite is green (server + widget), and the new autogrow and
keyboard tests appear in the `widget App` output.

### Level 3: Build

```bash
npm run build     # vite → dist/widget.js; size should stay ~11 kB gzip (README claim)
```

### Level 4: Manual visual QA (real browser — required, jsdom can't check layout)

```bash
npm run dev       # gateway on :8002
# open http://localhost:8002/demo  (core optional for pure composer checks — sending just shows typing)
```

Check each one, preferably in Chrome **and** Safari:

1. Empty composer: one line, 60px composer strip, icons vertically centered, pill look like before.
2. Type ~1 line: no growth. Keep typing past the width: grows to 2 lines, text wraps, no horizontal scroll.
3. Keep typing to 6+ lines: stops at 140px, vertical thin scrollbar, caret stays visible while typing.
4. Shift+Enter adds a line (grows). Enter sends: bubble shows the line breaks, composer back to 1 line.
5. Paste a long paragraph: grows right away to the cap.
6. Paste a long URL/email with no spaces: wraps, no horizontal scroll.
7. Attach, mic and send stay on the bottom row as it grows.
8. With the log scrolled to the bottom, growing the composer keeps the last message visible.
   Scroll the log up, then grow the composer: your scroll position is not changed.
9. While waiting (after send), the textarea is disabled/dimmed like the old input.
10. Type an accented char with a dead key (´ + e on a Spanish/ES-intl layout): Enter while composing does not send.
11. Narrow viewport (~360px wide, mobile emulation): still no horizontal overflow.

---

## Final Checklist

- [ ] `<textarea rows=1>` replaces the `<input>`. `textRef` wired.
- [ ] `autosizeComposer()` resets to `auto` before reading `scrollHeight`, caps at `COMPOSER_MAX_HEIGHT_PX`, toggles `overflowY`.
- [ ] `useLayoutEffect` on `[input, open]` (covers programmatic clear).
- [ ] Enter / Shift+Enter / IME (`isComposing` + keyCode 229) handled, with `preventDefault` on send.
- [ ] Log stays pinned only when it was already at the bottom.
- [ ] CSS: `align-items: flex-end`, button bottom margins, radius 20px, 40px→140px, `resize: none`, `overflow-wrap: anywhere`.
- [ ] `140` appears in both `styles.ts` and `ui.tsx` with a "keep in sync" comment.
- [ ] Tests updated + new tests pass. `npm run typecheck` clean.
- [ ] `npm run build` OK. Level 4 manual checklist done.
- [ ] README sentence updated.
- [ ] Conventional commits on `feat/composer-autogrow`, e.g. `feat(widget): auto-growing multi-line composer (WhatsApp-style)`.

## Anti-Patterns to Avoid

- ❌ Measuring with a hidden mirror `<div>`/canvas. `height:auto` → `scrollHeight` is enough.
- ❌ Resizing only in `onInput` (the field would stay tall after Send).
- ❌ Counting `\n` to derive rows (ignores soft wraps).
- ❌ `field-sizing: content` as the only mechanism (Firefox has no support).
- ❌ New dependencies (`react-textarea-autosize` etc.). The widget is zero-dep and size-sensitive.
- ❌ Touching `api.ts`, the server, or the canonical payload. This is UI-only.
- ❌ Leaving `border-radius: 999px` (a tall field turns into an oval).
- ❌ Forcing the log to the bottom on every keystroke when the visitor scrolled up.

## Notes

- Parent repo follow-up (after merge): bump the `web` submodule pointer in the
  parent (`chore: bump web — auto-growing composer`), same as other bumps in
  `git log`.
- Confidence for one-pass success: **9/10**. The only real risk is visual
  (button alignment and the exact max height), and Level 4 covers it.
