# FB Messenger AI Chatbot — Clothing Brand (n8n)

A simple, fast n8n workflow set that turns a Facebook Page inbox into an
AI-powered customer-support + order-taking bot for a Bangladeshi clothing brand.

## What it does

- **Voice → text.** Audio attachments are transcribed with OpenAI Whisper.
- **Banglish replies.** The LLM is prompted to answer in Bangla written in
  Roman letters, 1–2 short sentences, human and warm.
- **Reads Google Sheets before replying.** Every inbound message looks up the
  user's row first; if `ai_enabled = FALSE`, the bot stays silent — a human is
  driving. This is the "AI on/off" switch.
- **Order capture → Google Sheets.** When the user has supplied product, size,
  quantity, address, and phone, a new row is appended to the `Orders` tab.
- **Telegram handoff.** When the LLM decides `needs_human = true`
  (complaints, refunds, abuse, off-topic, repeated unresolved questions), the
  bot pings your Telegram chat with PSID + last message and flips
  `ai_enabled = FALSE` on the user row.
- **Per-user memory.** The last 5 turns are stored as JSON in the user's row
  and replayed on every call — cheap, no extra DB.
- **Auto lead follow-up.** A scheduled workflow nudges leads who went quiet
  4–22h ago (inside FB's 24h messaging window), max 2 nudges per lead.

## Architecture

```
Facebook Page
     │  (Messenger webhook)
     ▼
┌──────────────────────────────────────────────────────────────┐
│  n8n: 01-messenger-chatbot                                   │
│                                                              │
│  Webhook ─► Ack 200 ─► Extract Event ─► Is Audio? ──┐        │
│                                                     │        │
│              text ◄────────────────────────────────┘        │
│              audio ─► Download ─► Whisper ─► Set Text       │
│                                                              │
│  Merge ─► Sheets: Get User ─► Build Context ─► AI Enabled? ─┐│
│                                                              ││
│  no ► (stop, human is handling)                             ││
│  yes ► Build Payload ─► OpenAI Chat (JSON) ─► Parse AI ─────┘│
│                                              │               │
│       ┌──────────────────────────────────────┼─────────────┐ │
│       ▼              ▼              ▼              ▼       │ │
│  Needs Human? ─► Telegram     Order? ─► Sheets    FB Reply  │ │
│                  Notify       Append Order        + Upsert  │ │
│                                                   User row  │ │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  n8n: 02-lead-followup                                       │
│                                                              │
│  Schedule (every 2h) ─► Read Users ─► Pick Leads ─►          │
│  Send Nudge (FB) ─► Update followup_count                    │
└──────────────────────────────────────────────────────────────┘
```

## Files

```
.
├── README.md
├── workflows/
│   ├── 01-messenger-chatbot.json   ← import into n8n
│   └── 02-lead-followup.json       ← import into n8n
├── docs/
│   ├── google-sheets-schema.md     ← exact column names per tab
│   └── setup.md                    ← end-to-end setup guide
└── prompts/
    └── system-prompt.txt           ← canonical AI system prompt
```

## Quick setup

1. **Google Sheet.** Create a sheet with three tabs: `Users`, `Orders`,
   `Leads`. Copy the headers from [`docs/google-sheets-schema.md`](docs/google-sheets-schema.md)
   exactly. Note the spreadsheet ID from the URL.
2. **Facebook.** Create a Facebook App, add Messenger product, get a Page
   Access Token, and pick any verify token string.
3. **OpenAI.** Get an API key (used for both Whisper and `gpt-4o-mini`).
4. **Telegram.** Create a bot via `@BotFather`, message it once, and grab your
   chat ID from `https://api.telegram.org/bot<TOKEN>/getUpdates`.
5. **n8n env vars** (Settings → Variables, or container env):
   ```
   FB_VERIFY_TOKEN=<any string, also pasted into FB webhook config>
   FB_PAGE_TOKEN=<page access token from FB>
   SHEET_ID=<google spreadsheet id>
   TELEGRAM_BOT_TOKEN=<from BotFather>
   TELEGRAM_CHAT_ID=<your chat id>
   ```
6. **n8n credentials.** Create two:
   - `Google Sheets account` — type **Google Sheets OAuth2 API**.
   - `OpenAI Bearer` — type **Header Auth**, name `Authorization`,
     value `Bearer YOUR_OPENAI_KEY`.
7. **Import workflows.** In n8n: Workflows → Import from File → select each
   JSON in `workflows/`. Re-pick the credentials in the Sheets and
   OpenAI/Whisper nodes (n8n strips credential IDs on import).
8. **Activate `01-messenger-chatbot`** and copy its production webhook URL.
9. **Wire FB.** In your FB App → Messenger → Webhooks: paste the URL, paste
   the same `FB_VERIFY_TOKEN`, subscribe to the `messages`,
   `messaging_postbacks` events, and subscribe the Page.
10. **Activate `02-lead-followup`** so nudges start running.

Full step-by-step including FB app config is in
[`docs/setup.md`](docs/setup.md).

## How "AI reads Sheets first" works

The very first thing the workflow does on each inbound message is a
`Sheets: Get User Row` lookup by `psid`. The next node, `Build Context`,
parses `ai_enabled` from that row. Then `AI Enabled?` is an IF node — its
`false` branch is empty, so the workflow simply **stops** without calling
OpenAI or replying. To re-engage the bot for that user, flip the cell back
to `TRUE` in the sheet (it's a checkbox column).

This is also how `Telegram Notify` "takes over": when `needs_human=true`,
the same row's `ai_enabled` gets written to `FALSE` in `Upsert User`, so the
next inbound from that user is silently handed to your humans.

## Speed notes

- The webhook responds 200 to Facebook **immediately** (`Ack 200 OK`) before
  any AI work, which avoids FB retry storms.
- `gpt-4o-mini` with `max_tokens: 200` keeps round-trip under ~1.5 s typical.
- Memory is bounded to 5 turns to keep the prompt small.
- Only one Sheets read and one Sheets write per message.

## Customizing

- **Brand voice / products:** edit the system prompt in
  `prompts/system-prompt.txt` and mirror the change inside the
  `Build OpenAI Payload` Code node of `01-messenger-chatbot.json`.
- **Follow-up cadence:** edit the schedule interval and the `MIN_AGE` /
  `MAX_AGE` / `NUDGES` array in the `Pick Leads` Code node of
  `02-lead-followup.json`.
- **Different LLM:** swap the URL and JSON body of the `OpenAI Chat` HTTP
  Request node; the rest of the workflow only needs the parsed JSON
  schema (`reply`, `intent`, `needs_human`, `order`).
