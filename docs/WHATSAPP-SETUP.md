# Getting WhatsApp Business API credentials

Everything `chasqui new` asks for in its WhatsApp step comes from a (free)
Meta developer app. ~10 minutes, no company verification needed to start:
Meta gives you a **test number** that can message up to 5 phones — perfect
for development. This guide ends with the exact `.env` mapping.

## 1. Developer account and app

1. Go to <https://developers.facebook.com> and log in with a Facebook
   account (create one if needed) → **Get started** to enable developer
   mode.
2. **My Apps → Create app**. Use case: **Other** → type: **Business**.
   Name it whatever you like.
3. In the app dashboard, find the **WhatsApp** product → **Set up**. Meta
   creates (or asks you to pick) a *WhatsApp Business Account* (WABA) and
   gives you a test phone number.

## 2. Collect the values (WhatsApp → API Setup)

The **API Setup** page shows almost everything:

| Wizard asks / `.env` var | Where it is |
|---|---|
| `WA_PHONE_ID` | API Setup → "Phone number ID" (under the test number). **Required** — operator replies can't be sent without it. |
| `WA_WABA_ID` | API Setup → "WhatsApp Business Account ID". |
| `WA_TOKEN` | API Setup → "Temporary access token" (⚠️ expires in 24h — see §4 for a permanent one). |
| `WA_APP_ID` / `WA_APP_SECRET` | App dashboard → **App settings → Basic** ("App ID" and "App secret" → Show). |
| `WA_VERIFY_TOKEN` | **You invent this one** — `chasqui new` generates it and prints it; you never get it from Meta, you *give* it to Meta (§5). |

## 3. Allow your phone to receive messages

Test numbers only message **registered recipients** (max 5): API Setup →
"To" field → **Manage phone number list** → add your personal WhatsApp
number and confirm the code Meta sends you.

Send yourself the "hello world" template from that page once — it confirms
the number works before Chasqui enters the picture.

## 4. A token that doesn't expire (recommended)

The API Setup token dies in 24 hours. For a permanent one:

1. <https://business.facebook.com> → **Settings** (Business settings) →
   Users → **System users** → Add (role: Admin is fine for dev).
2. Select the system user → **Add assets** → Apps → your app → full
   control. Repeat for the WhatsApp account if listed.
3. **Generate new token** → choose your app → token expiration **Never** →
   permissions: `whatsapp_business_messaging` +
   `whatsapp_business_management` → Generate.
4. Use that as `WA_TOKEN`.

## 5. The webhook (how messages reach your gateway)

**Local dev** — the gateway registers the webhook for you:

```bash
ngrok http 8000                      # or your gateway port
# in whatsapp/.env:
WA_CALLBACK_URL=https://<your-id>.ngrok-free.app
```

Restart the gateway: PyWa calls Meta's API (using `WA_APP_ID`/`WA_APP_SECRET`
and your `WA_VERIFY_TOKEN`) and points the app's webhook at your tunnel. If
it races the tunnel on first boot, restart once. New ngrok URL = update the
var and restart.

**Production** — configure it once in the dashboard instead: WhatsApp →
**Configuration** → Webhook: callback URL `https://wsp.<your-domain>/`,
verify token = your `WA_VERIFY_TOKEN`, and subscribe to the **messages**
webhook field. (Leave `WA_CALLBACK_URL` unset in prod.)

## 6. Going live (later)

When you outgrow the test number: add a real phone number in the WhatsApp
product, verify your business in Business Manager, and switch the app from
Development to Live mode. None of your Chasqui config changes — same vars,
new values.

---

Stuck? The gateway logs every webhook and send attempt — `make dev` output
is the first place to look (a `131047` error = the 24h customer-service
window closed; the panel explains it too).
