# Putting the chat widget on a website

The web channel needs **no third-party account** — no tokens, no verification,
no webhook. You run the gateway, allow the sites that may embed the widget, and
paste one `<script>` tag. This guide ends with the exact `.env` mapping.

## 1. Run the gateway

`chasqui new` scaffolds it when you opt in to the web channel. Manually:

```bash
cd web && npm install
npm run build          # bundles dist/widget.js (~11 kB gzip)
npm run dev            # gateway on :8002
```

And point the **core** at it (this is what makes deferred replies reach the
browser — ADR-004/008):

```bash
# core/.env
CHANNEL_WEB_SEND_URL=http://localhost:8002/send
```

## 2. The values

| Wizard asks / `.env` var | What it is |
|---|---|
| `WEB_ALLOWED_ORIGINS` | Comma-separated origins allowed to embed the widget — the customer sites, e.g. `https://acme.com,https://www.acme.com`. Anything else gets a 403. `*` allows any origin (**dev only, never production**). |
| `INTERNAL_API_KEY` | Must equal the core's. Lives only server-side — the browser never sees it. |
| `CORE_URL` | Where the core lives (default `http://localhost:8090`). |
| `ERROR_REPLY` / `UNSUPPORTED_REPLY` | The two user-facing literals, shown only when the core is unreachable — set them in your users' language. |
| `RATE_LIMIT_WINDOW_MS` / `RATE_LIMIT_MAX` | Public-route rate limit per visitor (default 30/min). |
| `HISTORY_LIMIT` | How many past messages rehydrate when the widget reopens (default 50). |

## 3. Embed it

Paste on any page of an allowed origin:

```html
<script src="https://chat.your-domain.com/widget.js"
        data-gateway="https://chat.your-domain.com"></script>
```

That's the whole integration: a floating bubble appears bottom-right; clicking
it opens the chatbox. `data-gateway` may be omitted when the script is served
by the gateway itself. Styles are Shadow-DOM isolated — the host page's CSS
can't break the widget and vice versa.

## 4. Talk to your agent

Open the page and say hi. Same core, same memory, FAQ-RAG and tools as every
other channel — plus:

- **Live replies** over SSE (the deferred turn lands in the open chatbox).
- **Images and voice notes**, both directions the contract allows.
- **Rehydration** — close the tab mid-turn, reopen, the reply is there.
- Visitors are **anonymous**: a UUID in `localStorage` is the contact identity
  (§10 analog). Cleared storage = a new visitor. Capturing a name/email is a
  system-prompt concern, not the channel's.

## Production notes

- Serve the gateway over **HTTPS** (mic capture requires a secure context).
- Keep `WEB_ALLOWED_ORIGINS` tight — it is the only thing standing between
  your agent and every website on the internet.
- The visitor→SSE map is **in-memory**: run a single replica (or sticky
  sessions). Horizontal scale needs a shared pub/sub — documented follow-up.

---

Stuck? The gateway logs every relay attempt (`npm run dev` output), and
`GET /health` on the gateway plus the demo page (`/demo.html`, dev only) are
the fastest smoke tests.
