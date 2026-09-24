# Facebook Comment Automation (n8n)

An n8n workflow that automatically moderates and replies to Facebook Page comments for **Kasri Oil**, a pain-relief-oil brand page. Positive/neutral comments get an AI-generated public reply plus a private inbox message with offer details; negative, complaint, or abusive comments are automatically deleted.

## What it does

1. Listens for new Facebook Page comment events via a webhook (Graph API `feed` subscription)
2. Filters out non-comment events (reactions, shares, edits) so only new comments (`item: comment`, `verb: add`) are processed
3. Sends the comment text to an AI model (Groq — `gpt-oss-120b`) with a system prompt containing the brand's reply patterns, offers, and classification rules
4. Classifies each comment as `normal` or `negative`
   - **normal** → AI drafts a short public reply (redirects to inbox, includes contact info) + sends a private reply to the commenter with full offer details
   - **negative** → comment is deleted via the Graph API (no reply)
5. Logs every processed comment (person, comment, AI reply/action) to a Google Sheet for review
6. Acknowledges Facebook's webhook immediately (separate from the AI/reply branch) to avoid duplicate-delivery retries

## Architecture

```
Facebook Page (kasrioil)
        │  webhook (feed field)
        ▼
   n8n Webhook node
        │
        ├── GET  → verify hub.challenge (webhook handshake)
        │
        └── POST → Respond to Webhook (immediate ack, prevents FB retries)
                 │
                 └── Filter (item == "comment" AND verb == "add")
                          │
                          └── Edit Fields (extract comment_id, comment_text, commenter name)
                                   │
                                   └── AI Agent (Groq gpt-oss-120b)
                                       — classifies + drafts reply using system prompt
                                            │
                                            ├── normal   → Reply to Comment (Graph API)
                                            │            → Send Private Reply (Graph API messages)
                                            │            → Append row to Google Sheet
                                            │
                                            └── negative → Delete Comment (Graph API)
                                                         → Append row to Google Sheet
```

## Setup

### 1. Facebook App
- Create a Facebook Developer App (Business type), connect the target Page
- Add the **Webhooks** product, subscribe the Page object to the **feed** field
- Generate a Page Access Token with the following permissions:
  - `pages_manage_engagement`
  - `pages_read_engagement`
  - `pages_read_user_content`
  - `pages_messaging` (required for private replies)
- Set the app to **Live mode** to receive comments from the general public (may require App Review for `pages_messaging` in production)

### 2. n8n
- Import the workflow JSON
- Configure credentials:
  - **Groq API** (Chat Model node) — free tier, no card required
  - **Google Sheets** (for logging)
- Replace placeholders in HTTP Request nodes:
  - `PAGE_ID` → your Facebook Page ID
  - `PAGE_ACCESS_TOKEN` → the token generated above
- Set the Webhook node's **Respond** setting to *"Using Respond to Webhook Node"*, and wire a `Respond to Webhook` node directly off the webhook (parallel to the processing branch) so Facebook gets an immediate `200 OK`

### 3. Facebook Webhook subscription
- Callback URL: your n8n **production** webhook URL (not `/webhook-test/`)
- Verify token: must match the value checked in the workflow's `If` node
- After saving, confirm the Page itself is subscribed to the app's feed field:
  ```
  POST /{page-id}/subscribed_apps?subscribed_fields=feed&access_token={page-token}
  ```

## Notes & limitations

- **Private replies** can only be sent once per comment, and only within 7 days of the comment being made
- **Groq free tier**: ~1,000 requests/day, ~200,000 tokens/day (model-dependent) — sufficient for moderate comment volume; monitor usage if traffic grows
- The AI classifies negative/harmful comments using **pattern-based reasoning** (not a fixed keyword list) to catch creative spellings, Banglish, and symbol substitution
- Access tokens should be stored as n8n credentials rather than hardcoded in node parameters where possible, and rotated if ever exposed

## Tech stack

- **n8n** (self-hosted) — workflow orchestration
- **Facebook Graph API** (v21.0) — webhook events, comment reply/delete, private messaging
- **Groq API** (`gpt-oss-120b`) — comment classification and reply generation
- **Google Sheets** — execution/audit log
