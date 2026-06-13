# Getting a Telegram bot token

Everything the Telegram gateway needs comes from **@BotFather** — Telegram's
official bot for creating bots. ~2 minutes, no developer account, no business
verification, no number to register. This guide ends with the exact `.env`
mapping.

## 1. Create the bot

1. In any Telegram app, open a chat with **[@BotFather](https://t.me/BotFather)**
   (the one with the blue verified check).
2. Send `/newbot`.
3. Give it a **name** (the display name, e.g. `Acme Assistant`).
4. Give it a **username** — must be unique and end in `bot`
   (e.g. `acme_assistant_bot`).
5. BotFather replies with a **token** like `8123456789:AAH...`. That's your
   `TELEGRAM_BOT_TOKEN`.

That's it — no token expiry, no permissions to configure.

## 2. The values

| Wizard asks / `.env` var | Where it is |
|---|---|
| `TELEGRAM_BOT_TOKEN` | The token @BotFather gave you in step 1. Keep it secret. |
| `TELEGRAM_WEBHOOK_SECRET` | **You invent this one** — any random string. The gateway passes it to Telegram on `setWebhook`, and Telegram echoes it back in the `X-Telegram-Bot-Api-Secret-Token` header on every call so the gateway can verify authenticity. `chasqui new` generates it for you. |

## 3. The webhook (how messages reach your gateway)

**Local dev** — the gateway registers the webhook for you on startup:

```bash
ngrok http 8001                      # the Telegram gateway port
# in telegram/.env:
TELEGRAM_WEBHOOK_URL=https://<your-id>.ngrok-free.app/webhook
```

Restart the gateway: it calls Telegram's `setWebhook` (with your
`TELEGRAM_WEBHOOK_SECRET`) and points the bot at your tunnel. New ngrok URL =
update the var and restart. Verify any time with:

```bash
curl "https://api.telegram.org/bot<TOKEN>/getWebhookInfo"
# url set, last_error_message empty, pending_update_count low
```

**Production** — set `TELEGRAM_WEBHOOK_URL` to your public gateway URL
(`https://tg.<your-domain>/webhook`) once; the gateway registers it on boot.
No dashboard step.

## 4. Talk to your bot

Open `t.me/<your_bot_username>`, hit **Start**, and send a message. The agent
replies in the same chat — same core, same memory, FAQ-RAG and tools as every
other channel.

> Group chats are out of scope (Chasqui is one thread per contact). Use the
> bot in a 1:1 direct chat.

---

Stuck? The gateway logs every webhook and send attempt — `make dev` output is
the first place to look. `getWebhookInfo` (above) shows Telegram's side: a
non-empty `last_error_message` usually means the tunnel URL is stale or the
gateway was down when Telegram tried to deliver.
