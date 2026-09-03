# emailPlus — Inbound Enquiry Triage (n8n)

An n8n workflow that watches a mailbox for inbound enquiries, uses an LLM to classify each one and draft a reply, routes low-risk enquiries to auto-send while escalating anything sensitive to a human, and logs every enquiry to a Google Sheet.

Built and validated via the [n8n MCP server](https://github.com/czlonkowski/n8n-mcp) tools — the workflow JSON in this repo (`workflow.json`) is a direct export of what's running in n8n.

## What it does

1. **Watches an IMAP mailbox** for new enquiries.
2. **Classifies intent** (Sales / Support / Billing / Complaint / Other) and **drafts a personalised reply** with an LLM (OpenAI `gpt-4o-mini` by default).
3. **Decides whether a human needs to look at it first.** The LLM also outputs a `requiresReview` flag — true for anything it judges to be a Complaint, a Billing dispute, legally/financially sensitive, or ambiguous. False for routine Sales/Support questions.
4. **Routes accordingly:**
   - `requiresReview = false` → the drafted reply is **sent automatically** to the sender.
   - `requiresReview = true` → a **notification email** goes to a human reviewer instead, with the original message and the AI's draft attached for them to edit and send manually.
5. **Logs every enquiry** to a Google Sheet, regardless of which path it took, including whether it was auto-sent or escalated.
6. **Skips duplicates** — each email's Message-ID is tracked so a re-triggered or re-polled email isn't processed (and replied to) twice.
7. **Notes attachments** — attachments are downloaded during processing and their filenames are logged, but the files themselves are not persisted anywhere (see Limitations).

## Architecture

```
Enquiry Received (IMAP trigger)
        |
        v
      Config  ──────────────────────────────────────────┐  (holds fromAddress / reviewerEmail,
        |                                                │   passed through to every later node)
        v                                                │
 Normalize Fields  (senderEmail, senderName, subject,    │
   body, messageId, attachment info)                     │
        |                                                │
        v                                                │
 Skip Duplicate Emails  (Code node; drops already-seen    │
   Message-IDs using workflow static data)               │
        |                                                │
        v                                                │
 Classify & Draft Reply  <──── OpenAI Chat Model          │
   (Information Extractor: intent, draftReply,            │
    requiresReview)                                       │
        |                                                │
        v                                                │
 Consolidate Fields  (flattens the AI output + earlier ───┘
   fields into one flat object)
        |
        v
 Needs Human Review? (If node, branches on requiresReview)
        |
   ┌────┴─────┐
   │ true      │ false
   v           v
Notify        Send Reply
Reviewer      (auto-send to sender)
   |           |
   v           v
Prepare Row   Prepare Row
(Escalated)   (Auto-sent)
   |           |
   v           v
Log to Sheet  Log to Sheet
(Escalated)   (Auto-sent)
```

Both branches write to the same Google Sheet tab; the `Status` column records which path an enquiry took.

## Node-by-node

| Node | Type | Role |
|---|---|---|
| Enquiry Received | Email Trigger (IMAP) | Polls the mailbox, marks messages read after fetching, downloads attachments |
| Config | Set | Single place to hold your support "from" address and the reviewer's email |
| Normalize Fields | Set | Extracts sender name/email, subject, body, Message-ID, and attachment metadata into flat fields |
| Skip Duplicate Emails | Code | Tracks processed Message-IDs in workflow static data; drops repeats |
| OpenAI Chat Model | OpenAI Chat Model | The LLM backing the classifier below |
| Classify & Draft Reply | Information Extractor | One LLM call returns `intent`, `draftReply`, and `requiresReview` |
| Consolidate Fields | Set | Flattens the extractor's output together with the earlier normalized fields, so every later node reads plain `$json.x` |
| Needs Human Review? | If | Branches on `requiresReview` |
| Notify Reviewer | Send Email (SMTP) | Emails the reviewer with the enquiry + AI draft, for manual editing/sending |
| Send Reply | Send Email (SMTP) | Sends the AI draft straight to the enquirer |
| Prepare Row (Escalated / Auto-sent) | Set | Builds the row that gets logged, including a `Status` column |
| Log to Sheet (Escalated / Auto-sent) | Google Sheets — Append Row | Writes the row to the `Enquiries` tab |

## Setup

The MCP tools that built this workflow can create/update workflow *structure*, but credentials have to go through n8n's own UI (by design — API keys and OAuth shouldn't pass through a workflow-management tool). Three credentials are needed:

1. **IMAP** — for the "Enquiry Received" trigger. Host, username, password/app-password, TLS settings for your mailbox provider.
2. **OpenAI API key** — for "OpenAI Chat Model".
3. **SMTP** — for "Notify Reviewer" and "Send Reply" (can be the same mailbox's SMTP, or a separate sending account).
4. **Google Sheets OAuth2** — for both "Log to Sheet" nodes.

Then:

1. **Edit the "Config" node**: replace `fromAddress` with the address replies should be sent from, and `reviewerEmail` with the address that should get escalation notifications.
2. **Create the Google Sheet**: a spreadsheet with a tab named `Enquiries` and this header row:
   `Timestamp | Sender Email | Sender Name | Subject | Intent | Original Message | Drafted Reply | Attachments | Message ID | Status`
   Put its spreadsheet ID into both "Log to Sheet" nodes' `documentId` field (currently `REPLACE_WITH_SPREADSHEET_ID`).
3. **Assign credentials** to each node listed above.
4. **Test**: send a real enquiry to the mailbox, run the workflow (or wait for the poll), and check:
   - The Google Sheet gets a new row with sensible `Intent` and `Drafted Reply` values.
   - A routine question triggers "Send Reply" (check the sender's inbox / your sent folder).
   - Something you'd expect to be a Complaint/Billing issue triggers "Notify Reviewer" instead.
5. **Activate** the workflow once you're happy with the classification and drafting quality.

### Tuning the AI behaviour

The classification categories, the `requiresReview` rule, and the reply's tone are all defined in the **attribute descriptions** on the "Classify & Draft Reply" node — edit those descriptions directly in n8n to change behaviour; no code changes needed.

## Known limitations

- **No RFC threading.** n8n's built-in SMTP "Send Email" node doesn't support setting custom `In-Reply-To`/`References` headers, so replies aren't linked to the original message at the protocol level. The workflow keeps replies visually grouped by re-using `Re: <original subject>`, which most mail clients group by, but it isn't true thread-linking. A custom HTTP-based SMTP integration would be needed for full threading.
- **`messageId` field name is best-effort.** It's read from the IMAP trigger's output as `$json.messageId`, which is the common field name for this in n8n's IMAP node, but hasn't been confirmed against a live test email in this mailbox. If deduplication or the Message ID column comes back empty, check the actual field name on a real execution and adjust "Normalize Fields" accordingly.
- **Attachments aren't stored anywhere durable.** They're downloaded during the run (so the AI/log step can see filenames) but not uploaded to Drive or similar — add a Google Drive (or similar) node if you need to keep the files themselves.
- **Auto-send has no second safety net beyond the `requiresReview` flag.** If the LLM misjudges a routine-looking enquiry, the drafted reply goes out with no human in the loop. Tighten the `requiresReview` description if you want a wider safety margin.
- Workflow-level static data (used for dedup) resides in n8n's own database; it isn't visible in this repo and isn't reset by re-importing `workflow.json`.

## Importing into n8n

In n8n: **Workflows → Import from File** and select `workflow.json`, or paste its contents via **Import from URL/Clipboard**. Credentials are never included in exports — you'll need to reassign them as described in Setup above.
