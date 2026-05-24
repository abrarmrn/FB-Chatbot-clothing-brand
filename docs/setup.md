# Setup guide

End-to-end steps from a blank n8n install to a live Facebook Messenger bot.

## 0. Prerequisites

- An n8n instance reachable from the public internet (n8n Cloud, or
  self-hosted with HTTPS — Facebook will not call HTTP).
- A Facebook Page you own.
- A Google account.
- An OpenAI API key with access to `gpt-4o-mini` and `whisper-1`.
- A Telegram account.

## 1. Google Sheet

1. Create a new spreadsheet. Name it e.g. `FB-Chatbot-DB`.
2. Create three tabs: `Users`, `Orders`, `Leads`.
3. In each tab, paste the exact header row from
   [`google-sheets-schema.md`](google-sheets-schema.md).
4. In the `Users` tab, format the `ai_enabled` column as a checkbox
   (Insert → Checkbox).
5. Copy the spreadsheet ID from the URL
   (`https://docs.google.com/spreadsheets/d/<THIS_PART>/edit`).

## 2. Facebook App + Page Token

1. Go to <https://developers.facebook.com/apps>, create a new app
   (type: **Business**).
2. Add the **Messenger** product.
3. Under Messenger → Settings → Access Tokens, select your Page and
   generate a **Page Access Token**. Copy it — this is `FB_PAGE_TOKEN`.
4. Pick any random string for `FB_VERIFY_TOKEN`
   (e.g. `bangla-clothing-2026-x9k`). You will paste this in two places.

## 3. OpenAI

1. Get an API key at <https://platform.openai.com/api-keys>.
2. Confirm your account has Whisper + gpt-4o-mini access (default for
   funded accounts).

## 4. Telegram

1. In Telegram, message `@BotFather`, send `/newbot`, follow the prompts.
   Save the **bot token**.
2. Send any message to your new bot.
3. Open
   `https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates` in a browser.
   Find `"chat":{"id": <NUMBER> ...}`. That number is `TELEGRAM_CHAT_ID`.

## 5. n8n env vars

Set these as environment variables on your n8n instance (n8n Cloud:
Settings → Variables). The workflows read them via `$env.NAME`.

```
FB_VERIFY_TOKEN=bangla-clothing-2026-x9k
FB_PAGE_TOKEN=EAAB...your-long-page-token...
SHEET_ID=1AbCdEfGhIjKlMnOpQrStUvWxYz1234567890
TELEGRAM_BOT_TOKEN=1234567890:ABC...
TELEGRAM_CHAT_ID=123456789
```

## 6. n8n credentials

In n8n → Credentials → New:

1. **Google Sheets OAuth2 API**
   - Name: `Google Sheets account`
   - Click Connect, authorize the Google account that owns the spreadsheet.

2. **Header Auth**
   - Name: `OpenAI Bearer`
   - Header name: `Authorization`
   - Header value: `Bearer sk-...your-openai-key...`

## 7. Import workflows

1. n8n → Workflows → Import from File → select
   `workflows/01-messenger-chatbot.json`.
2. Open every Google Sheets node and re-select the `Google Sheets account`
   credential (n8n strips credential IDs on import).
3. Open `Whisper Transcribe` and `OpenAI Chat` HTTP Request nodes and
   re-select the `OpenAI Bearer` Header Auth credential.
4. Save. Repeat steps 1–3 for `workflows/02-lead-followup.json`.

## 8. Activate the messenger workflow

1. Open `01-messenger-chatbot`.
2. Toggle **Active** in the top-right.
3. Click the `FB Webhook (POST)` node and copy the **Production URL**.
   It looks like `https://your-n8n.example.com/webhook/fb-webhook`.

## 9. Wire Facebook to n8n

1. In the FB App → Messenger → Webhooks → click **Edit Callback URL**.
2. Callback URL: paste the production URL from step 8.
3. Verify Token: paste the same `FB_VERIFY_TOKEN`.
4. Click **Verify and Save**. Facebook will hit the GET endpoint; the
   workflow's `Verify Token` Code node returns the challenge.
5. Subscribe the webhook to: `messages`, `messaging_postbacks`.
6. Under "Generate token", select your Page and click **Add Subscriptions**.

## 10. Test

- Open Messenger, message your Page. You should see a Banglish reply
  within ~1–2 s. The user row appears in `Users` with memory.
- Send a voice note. The audio is transcribed via Whisper, then replied.
- Type "ami human / agent chai" — Telegram fires, `ai_enabled` flips to
  FALSE in the sheet, the bot goes quiet for that user.
- To resume the bot for that user, flip `ai_enabled` back to TRUE.

## 11. Activate follow-ups

1. Open `02-lead-followup`.
2. Toggle **Active**. The schedule runs every 2 hours.
3. Leave a test conversation idle for ~4 hours; the bot will send a
   Banglish nudge and increment `followup_count`.

## Troubleshooting

| Symptom                              | Likely cause                                                                                                    |
| ------------------------------------ | --------------------------------------------------------------------------------------------------------------- |
| FB webhook verification fails        | `FB_VERIFY_TOKEN` env value differs from what you typed in FB. They must match exactly.                         |
| 403 from FB Send API                 | `FB_PAGE_TOKEN` expired or wrong Page. Regenerate.                                                              |
| `Get User Row` returns no item       | First-time user — the workflow handles this in `Build Context` (ai_enabled defaults to true, memory to []).     |
| Bot answers in Bangla script         | The system prompt drift — re-paste it from `prompts/system-prompt.txt` into `Build OpenAI Payload` Code node.   |
| AI never replies for one user        | Their `ai_enabled` cell is FALSE. Flip it to TRUE in the sheet.                                                 |
| Whisper returns 415                  | The FB audio URL expired (they live ~minutes). Make sure `Download Audio` runs immediately, no long sleeps before. |
| Follow-up not sending                | User is outside 4–22h window, status is `closed`/`customer`, or `followup_count` already 2.                     |
