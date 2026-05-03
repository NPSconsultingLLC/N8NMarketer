# N8NMarketer

A two-workflow n8n system that turns Slack messages into approved, scheduled LinkedIn posts using a Notion content pipeline as the source of truth.

## How it works

```
Slack message/URL ──► Drafter workflow ──► Notion (status: Draft)
                                              │
                          Slack draft posted   │
                                ▼              │
                       Human edits + ✅ react  │
                                ▼              │
                       Drafter (approval) ────►│  status: Approved
                                              │
                          Every 30 min:        ▼
                       Publisher workflow ──► LinkedIn ──► Notion (status: Published)
```

Two workflows, one shared Notion database:

- **`N8N LinkedIn Drafter.json`** — listens to a Slack channel. Drafts on-brand LinkedIn posts from URLs or freeform text, posts the draft back to Slack for review, and stages a row in Notion. When a teammate reacts ✅ on a draft (or on their edited copy of a draft), the workflow saves the final text to Notion and flips status to **Approved**.
- **`N8N Publisher.json`** — runs on a 30-minute schedule. Pulls Approved rows from Notion, posts them to LinkedIn, and marks them **Published**. The LinkedIn node ships disabled — wire your credentials in before flipping it on.

---

## Prerequisites

- A running n8n instance (self-hosted or Cloud), v2.18+
- Credentials for: **Slack**, **OpenAI**, **Notion**, **Jina AI** (for URL article extraction), and eventually **LinkedIn**
- A Slack channel where drafts live
- A Notion database (schema below)

---

## 1. Set up the Notion database

Create a Notion database with these exact column names and types:

| Column | Type | Notes |
|---|---|---|
| `Topic` | Title | Set automatically from the model's output |
| `Draft` | Rich text | Original AI-generated draft |
| `Final Version` | Rich text | Human-approved version |
| `Select` | Select | Options: `Draft`, `Approved`, `Published` |
| `Source URL` | URL | Article URL when the post is link-based |
| `Slack TS` | Rich text | Slack message timestamp — used to match approval reactions to the right row |

Share the database with your Notion integration and copy the database ID — you'll paste it into both workflow JSONs (search for `35434891-7cfd-8033-be98-e1660ae14c07` and replace).

---

## 2. Set up the Slack app

In your Slack workspace, create an app with:

- **Bot token scopes:** `channels:history`, `chat:write`, `reactions:read`, `reactions:write`, `app_mentions:read`
- **Event subscriptions:** subscribe to `message.channels` and `reaction_added`
- **Request URL:** the production webhook URL from the Drafter's `Slack Trigger` node (visible after you import + activate the workflow)

Copy the channel ID for your drafting channel and replace `C0B18AR6BRC` in both workflow JSONs.

---

## 3. Import the workflows into n8n

1. In n8n, **Workflows → Import from File** for each of the two JSONs
2. For every node showing a credential warning, open it and pick the matching credential from your n8n instance
3. In `Voice Guide (Edit Me)` (Drafter workflow), the `voice_guide` system prompt is hardcoded. Edit it if you want different brand voice rules
4. Save and **activate** the Drafter workflow. Copy its production webhook URL into your Slack app's Event Subscriptions and verify
5. Leave the Publisher workflow **inactive** until LinkedIn is wired up (see step 5)

---

## 4. Day-to-day usage

### Drafting a post

In the Slack channel you configured, post one of:

- **A URL** — `https://example.com/some-article` → the Drafter fetches the article via Jina AI and writes a LinkedIn post about it
- **A topic** — `Why most B2B marketing dashboards lie` → the Drafter writes a freeform post on that topic

Within ~30 seconds the bot posts a draft back to the channel, formatted as:

```
*Topic Title*

The draft body…

_[phaedon-bot]_
```

A new row appears in Notion with status **Draft**.

### Approving a draft

Two patterns work:

1. **Approve the bot's draft as-is** — just react ✅ (white check mark) on the bot's message
2. **Edit before approving** — paste your edited version as a new top-level message in the channel (we use `EDITED ...` as a convention) and react ✅ on that

Either way, the workflow:

- Finds the Notion row by matching the bot draft's Slack timestamp
- Writes your final text into `Final Version`
- Sets status to **Approved**
- Replies in Slack: `Approved and queued for publish.`

### Publishing

Once the Publisher is enabled, every 30 minutes it:

- Finds Notion rows where Select = Approved
- Posts the `Final Version` (or falls back to `Draft` if Final Version is empty) to LinkedIn
- Sets status to **Published**
- Posts a confirmation in Slack

---

## 5. Enabling LinkedIn publishing

The `Post to LinkedIn` node ships disabled and the workflow has a guard IF that prevents anything from being marked Published while LinkedIn is offline. To turn publishing on:

1. Add a LinkedIn OAuth credential to your n8n instance
2. Open `Post to LinkedIn`, attach the credential, and set the `person` (personal URN) or `organization` URN parameter
3. Enable the node
4. Run the Publisher manually with one Approved row to verify
5. Inspect the LinkedIn node's output JSON — confirm what field it returns (`id`, `urn`, `activity`, etc.)
6. If the `Publish Succeeded?` IF guard's expression doesn't match that field name, update it
7. Activate the Publisher workflow

---

## Troubleshooting

**Slack stops triggering the workflow.** Slack disables event delivery if the request URL fails repeatedly. Check **api.slack.com/apps → your app → Event Subscriptions** for a "disabled" warning. Also confirm the workflow is active in n8n; toggling active/inactive rotates the webhook URL, so you'll need to re-paste it into Slack.

**Approval reaction doesn't update Notion.** Open the Drafter execution in n8n. Check `Extract Final Draft` — the `bot_ts` should match the `Slack TS` column on the Notion row you expect. If they differ, the bot draft and the Notion row are out of sync (usually because the row was created in an earlier debug run). Either edit the Notion row's Slack TS to match, or restart with a fresh draft.

**Publisher publishes everything regardless of status.** Check that `Filter Approved (Defensive)` is in the chain after `Find Approved Posts`, and that your Notion `Select` column options include exactly `Draft`, `Approved`, `Published` (case sensitive).

**Drafts read off-brand.** Edit the `voice_guide` system prompt in the `Voice Guide (Edit Me)` node of the Drafter.

**Notion update fails with "page not found."** The `pageId` field in `Mark Approved in Notion` and `Mark as Published` must be in resource-locator format (`{ __rl: true, value: ..., mode: "id" }`), not a plain string.

---

## Files

- `N8N LinkedIn Drafter.json` — the drafting + approval workflow
- `N8N Publisher.json` — the scheduled LinkedIn publisher
- `Phaedon - Brand Voice Guide.pdf` — the source brand voice guide that's encoded into the Drafter's `voice_guide` prompt

---

## License

See `LICENSE`.
