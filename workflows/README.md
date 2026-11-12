# Workflows: import and setup

Three n8n workflows make up the system.

| File                  | Purpose                                                                                              | Active?                                            |
| --------------------- | ---------------------------------------------------------------------------------------------------- | -------------------------------------------------- |
| `kb-ingestion.json`   | Downloads `help-center.json`, splits it into chunks, embeds it, and stores it in ChromaDB. Run once. | No (manual)                                        |
| `support-ticket.json` | Classify, retrieve, draft, route, send for approval, handle button taps, log.                        | **Yes** (the Telegram trigger part must be active) |
| `error-alerts.json`   | Error workflow. Sends a Telegram message when another workflow fails.                                | No (n8n runs it on demand)                         |

## Prerequisites

- n8n (a recent version with **Data tables** support)
- A running **ChromaDB** instance reachable from n8n
- A running **Ollama** instance with the embedding model pulled:
  ```bash
  ollama pull nomic-embed-text
  ```
- A **Groq** API key
- A **Telegram bot** (create one with @BotFather) and your own numeric chat ID

> Telegram webhooks require a public **HTTPS** URL. If n8n runs locally, use a tunnel or set `WEBHOOK_URL` properly.

## Setup steps

### 1. Create credentials in n8n

Create these under **Credentials**:

- Groq API
- Telegram API (bot token)
- ChromaDB (self-hosted)
- Ollama

The exported JSON files contain no credentials. After import, open each node that shows a warning and select your credential.

### 2. Create the ticket log table

In **Overview, Data tables**, create a table named `ticket_log` with these columns:

```
ticket_id (number), created_at, category, priority, needs_human, should_escalate,
kb_match_weak, best_score, route, customer_message, draft_reply, decision, decided_at
```

Columns other than `ticket_id` can be text. The log nodes store booleans and scores as text.

### 3. Import the workflows

For each JSON file: create a new workflow, then **⋯ menu, Import from file**.

### 4. Replace the placeholders

Search each workflow for these values and replace them:

| Placeholder                    | Where                                              | Replace with                                                                                                                           |
| ------------------------------ | -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `YOUR_TELEGRAM_CHAT_ID`        | Telegram send nodes, and the error workflow        | Your numeric Telegram chat ID                                                                                                          |
| Owner user ID in **Is it me?** | `support-ticket.json`                              | Your numeric Telegram user ID. In a private chat with the bot, this equals your chat ID. If left unchanged, nobody can approve drafts. |
| Data table selection           | **ticket log**, **Log approved**, **Log rejected** | Select your `ticket_log` table from the list                                                                                           |

### 5. Load the knowledge base

Run `kb-ingestion.json` once with **Execute workflow**. The ChromaDB collection is named `brightboard_knowledge_base`. If the collection does not exist yet, create it first or adjust the node.

### 6. Attach the error workflow

For `support-ticket.json`, open **⋯, Settings, Error Workflow** and choose "Support Ticket - Error Alerts". Do not attach it to itself.

### 7. Activate

Activate `support-ticket.json`. The **Telegram Trigger** only receives button taps while the workflow is active, not in test mode.

## Testing

1. Set a fresh `ticket_id` and a message in the first **Edit Fields** node.
2. Click **Execute workflow**.
3. A draft arrives in Telegram with **Approve / Reject** buttons. Tap one. The message should change to APPROVED or REJECTED.
4. Open **Data tables, ticket_log** and confirm the row moved from `pending` to `approved` or `rejected`.

Suggested test messages:

| Message                                    | Expected route                                      |
| ------------------------------------------ | --------------------------------------------------- |
| "How do I update my credit card?"          | Approval (billing is always reviewed). KB match ok. |
| "Do you have a dark mode on mobile?"       | Approval, with a "No KB match" warning.             |
| A how-to question that your articles cover | Auto-handled FYI message, no buttons.               |

Use a new `ticket_id` for every test. The log uses it as the unique key, so reusing one overwrites that row.

## How the approval flow works

The ticket workflow ends after it sends the draft. A button tap starts a **new** execution, so the same file contains a second branch:

```
Telegram Trigger → Is it me? → Answer Query → Parse tap → Switch
        → Mark as approved → Next steps → Log approved
        → Mark as rejected → Next steps → Log rejected
```

- **Is it me?** ignores taps from anyone but the owner.
- **Answer Query** stops the button spinner and continues on error, because Telegram rejects answers to old taps.
- **Parse tap** reads the ticket ID from the button data and takes the reply text from everything after the `──── DRAFT ────` separator, so the context header never ends up in the customer-facing reply.
- The edit-message nodes replace the buttons with the decision so a draft cannot be tapped twice.

## Troubleshooting

| Symptom                              | Likely cause                                                                                               |
| ------------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| "can't parse entities" from Telegram | A Telegram node is missing **Parse Mode: HTML**, or text was not HTML-escaped.                             |
| "query is too old"                   | Tapped a stale button, or the same tap was processed twice. Tap a button on a fresh message.               |
| Tap does nothing                     | The workflow is not active, or the sender ID does not match **Is it me?**.                                 |
| `decision` stays `pending`           | The ticket row did not exist when the tap arrived, or the log nodes point to a different table.            |
| Everything goes to approval          | Expected when `kb_match_weak` is true or the category is billing or refund. Check the `best_score` column. |
| No error alerts                      | Error workflows only fire on automatic runs, not manual runs from the editor.                              |
| Retry On Fail not visible on a node  | AI sub-nodes (models, embeddings) do not have it. Set it on the parent node (the LLM Chain) instead.       |
