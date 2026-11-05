# Brightboard AI Support Agent — Development Log

This document records the main technical challenges encountered while developing Brightboard AI Support Agent, the approaches used to solve them, and the issues that remain open.

The goal is to document the actual development process, including debugging decisions and lessons learned.

---

## 1. Running n8n and connecting external services

**Problem**

The project uses n8n inside Docker Desktop, while ChromaDB and Ollama run as separate services.

Using `localhost` from inside the n8n container does not refer to the Windows host machine. It refers to the container itself.

**Solution**

The service addresses were changed to use Docker's host gateway:

| Service  | Address from n8n                      |
| -------- | ------------------------------------- |
| ChromaDB | `http://host.docker.internal:8000`    |
| Ollama   | `http://host.docker.internal:11434`   |
| n8n      | `http://localhost:5678` from the host |

This allowed the n8n container to communicate with services running on the host machine.

**Lesson learned**

Container networking requires understanding the difference between a container's own localhost and the host machine.

---

## 2. ChromaDB collection configuration

**Problem**

When configuring the Chroma Vector Store node, the collection field was required. The node could not execute without a valid collection.

There was also uncertainty about whether another retriever or vector-store node was necessary.

**Solution**

A collection named:

`brightboard_knowledge_base`

was created and selected in the ChromaDB node.

The workflow uses:

- Insert mode for knowledge-base ingestion.
- Load mode for ticket-time retrieval.
- An Ollama embedding connection for vector operations.

The existing ChromaDB Load node was sufficient for retrieval, so an additional vector-store retriever was unnecessary.

**Lesson learned**

Understanding the different vector-store operations prevented unnecessary nodes and duplicate retrieval components.

---

## 3. Incorrect document ingestion

**Problem**

Early ingestion attempts did not produce the expected full article content. Some indexed records contained article identifiers and titles without the complete body.

This meant retrieval could identify an article without providing enough information to generate a useful answer.

**Solution**

The ingestion workflow was changed to explicitly format every article before loading it.

A JavaScript Code node now transforms the article data into text:

```javascript
const articles = $input.all().map((item) => item.json);

return articles.map((article) => ({
  json: {
    text: `Article ID: ${article.id}
Title: ${article.title}
Content: ${article.body}`,
    article_id: article.id,
    title: article.title,
  },
}));
```

The Default Data Loader uses the formatted `text` field as its document content.

Metadata such as `article_id` and `title` is retained separately.

The corrected ingestion workflow was tested with the help-center articles.

**Lesson learned**

A vector database can successfully store embeddings while still containing incomplete or poorly formatted source documents. Successful execution alone does not guarantee useful retrieval.

---

## 4. Document loading and text splitting

**Problem**

Configuring the Default Data Loader and Recursive Character Text Splitter involved several confusing options, including data type and input mode.

There was uncertainty about whether the incoming article data should be treated as binary data or text.

**Solution**

The workflow was configured to pass the formatted article text as JSON data to the Default Data Loader.

The Recursive Character Text Splitter was connected through the document loader's text-splitter input.

Final configuration:

| Parameter     |                  Value |
| ------------- | ---------------------: |
| Chunk size    |                    700 |
| Chunk overlap |                    100 |
| Input content | Formatted article text |

**Lesson learned**

The document loader, text splitter, and embedding model are separate parts of the ingestion pipeline. Each connection has a specific purpose.

---

## 5. Duplicate retrieval results

**Problem**

Some retrieval tests returned repeated articles.

This made the results less useful because the limited retrieval slots could be occupied by duplicate content.

**Approach**

The collection was inspected and re-ingestion was treated carefully.

The development workflow was kept separate from ticket processing so that inserting documents would not happen every time a support ticket arrived.

Duplicate-ingestion protection and collection cleanup remain important operational considerations.

**Lesson learned**

A repeatable ingestion process needs to account for existing records. Re-running an insert operation is not necessarily equivalent to updating a collection.

---

## 6. Building the classification pipeline

**Problem**

The LLM needed to return predictable ticket information rather than unrestricted natural-language output.

Without structured output, downstream nodes could receive inconsistent field names or values.

**Solution**

A Basic LLM Chain was connected to:

- Groq Chat Model
- Structured Output Parser

The classifier was given explicit category and priority definitions, along with escalation rules.

A JavaScript Code node then combined the classification result with the original ticket information.

This preserved fields such as:

- Ticket ID
- Original customer message
- Classification status

**Lesson learned**

Structured output makes LLM-based workflows easier to validate and connect to deterministic automation logic.

---

## 7. Connecting ChromaDB retrieval to response generation

**Problem**

The retrieved documents were returned as multiple separate items.

The response-generation model needed the original ticket information and all relevant retrieved content in one input.

**Solution**

A Code node named `Prepare Response Context` was added after ChromaDB retrieval.

It collects the retrieved documents and combines their page content into a single knowledge string.

The resulting item contains:

- Ticket ID
- Category
- Priority
- Human-handling flags
- Original message
- Retrieved knowledge

This gives the response-generation model a single, consistent context.

**Lesson learned**

Retrieval and generation are separate stages. A context-preparation step makes the handoff between them explicit and easier to debug.

---

## 8. Preventing unsupported AI responses

**Problem**

The response-generation model sometimes included actions that the workflow had not actually performed, such as claiming that a ticket had been forwarded or escalated.

It could also introduce troubleshooting instructions that were not supported by the retrieved documents.

**Solution**

The response-generation prompt was refined with grounding rules:

- Use the supplied knowledge base for company policies.
- Do not invent refund exceptions.
- Do not claim that a refund has been approved.
- Acknowledge issues not covered by the retrieved articles.
- Do not claim that a ticket has been forwarded, escalated, or assigned unless the workflow has performed that action.
- Generate a draft for human review.

**Remaining limitation**

Prompt instructions reduce unsupported claims but do not guarantee their complete elimination. Systematic response evaluation is still required.

**Lesson learned**

An AI support system should distinguish between what the model recommends, what the company policy states, and what the automation has actually done.

---

## 9. Telegram message parsing errors

**Problem**

Telegram returned errors similar to:

```text
Bad Request: can't parse entities:
Can't find end of the entity
```

The generated response contained Markdown formatting, while Telegram was configured to interpret message formatting using its entity-parsing rules.

Characters such as `*`, `<`, `>`, and `&` could cause parsing problems when the formatting was not valid for the selected parse mode.

**Approach**

The message expression was adjusted to escape HTML-sensitive characters and remove Markdown asterisks.

For example:

```javascript
{
  {
    $json.draft_reply
      .replace(/&/g, "&amp;")
      .replace(/</g, "&lt;")
      .replace(/>/g, "&gt;")
      .replace(/\*/g, "");
  }
}
```

**Status**

Message formatting still requires final validation. The goal is reliable plain-text delivery rather than spending unnecessary time reproducing complex formatting.

**Lesson learned**

LLM-generated text is not guaranteed to be valid HTML or Markdown. External messaging APIs have their own parsing requirements.

---

## 10. Implementing human approval in Telegram

**Problem**

A normal Telegram Send Message operation could deliver a draft, but it did not provide the required wait-for-decision behavior.

The workflow needed to pause until a human approved or rejected the draft.

**Solution**

The Telegram node was changed to:

`Send and Wait for Response`

The response type was configured for approval.

The approver restriction was set to the authorized Telegram user ID.

Initially, only one approval button appeared. The approval configuration was then changed to:

`Approve and Disapprove`

with custom button labels:

- Approve
- Decline

The Telegram node successfully returned a decision payload after an approval interaction.

Example:

```json
{
  "data": {
    "approved": true,
    "responder": {
      "id": "TELEGRAM_USER_ID",
      "source": "telegram"
    }
  }
}
```

**Lesson learned**

A human approval step should return structured decision data that can be used by the rest of the workflow, rather than relying on a human's free-text reply.

---

## 11. Routing approval decisions

**Problem**

The workflow needed to distinguish approved drafts from rejected drafts.

**Solution**

An n8n Switch node was configured using:

```javascript
{
  {
    $json.data.approved;
  }
}
```

Two Boolean conditions were added:

| Condition | Output   |
| --------- | -------- |
| `true`    | Approved |
| `false`   | Rejected |

This creates separate paths for the two possible decisions.

**Status**

Both routing rules are configured. End-to-end testing of both outcomes remains part of workflow validation.

---

## 12. Telegram approval buttons opening an n8n link

**Problem**

When attempting to approve or decline a Telegram message, the buttons redirected to an n8n URL instead of completing the interaction directly inside Telegram.

**Investigation**

The n8n instance is running locally in Docker and is exposed through an ngrok HTTPS tunnel.

Current tunnel configuration:

```text
https://trousers-custodian-enviably.ngrok-free.dev
    -> http://localhost:5678
```

The ngrok session was confirmed online.

**Status**

The public tunnel is running, but the n8n public URL and webhook-related environment configuration still need verification.

Relevant configuration variables to inspect:

- `WEBHOOK_URL`
- `N8N_EDITOR_BASE_URL`

This issue is not yet confirmed as resolved.

**Lesson learned**

Having a public HTTPS tunnel is not by itself proof that every n8n URL and callback is configured correctly.

---

## 13. Preserving ticket data after approval

**Problem**

The Telegram approval node returns a decision payload. The original generated draft and ticket information should not be assumed to remain available in that output.

For example, the approval response contains:

```javascript
$json.data.approved;
```

but the original draft is not automatically guaranteed to be present.

**Planned solution**

Preserve the important ticket fields and generated reply before the Telegram wait operation.

Proposed fields:

- `ticket_id`
- `category`
- `customer_message`
- `draft_reply`

The downstream workflow can then use the approval decision alongside the original ticket data.

**Status**

This data-preservation step still needs to be added and tested in the final workflow.

**Lesson learned**

Long-running approval steps create a data-flow boundary. Important information should be deliberately preserved before waiting for an external response.

---

## Summary of Development Status

| Component                        | Status                                    |
| -------------------------------- | ----------------------------------------- |
| Docker-based n8n setup           | Working                                   |
| ChromaDB connectivity            | Working                                   |
| Ollama embedding connection      | Working                                   |
| Help-center ingestion            | Tested                                    |
| Ticket classification            | Tested                                    |
| Structured output                | Tested                                    |
| Semantic retrieval               | Tested                                    |
| Response context preparation     | Tested                                    |
| Groq response generation         | Tested                                    |
| Telegram draft delivery          | Tested, formatting needs validation       |
| Telegram approval configuration  | Configured and approval tested            |
| Approve / Decline routing        | Configured, both outcomes need validation |
| Telegram callback navigation     | Unresolved                                |
| Data preservation after approval | Pending                                   |
| Customer delivery                | Not implemented                           |
| Evaluation on 75 labeled tickets | Planned                                   |

---

## Final Reflection

One of the most important lessons from this project has been that building an AI automation system is not simply a matter of connecting an LLM to a few workflow nodes.

The reliability of the system depends on the entire pipeline:

- Data must be loaded correctly.
- Documents must be searchable and relevant.
- LLM outputs must follow predictable structures.
- Generated responses must be grounded in available information.
- External integrations must handle formatting and callbacks correctly.
- Human decisions must be represented explicitly in the workflow.
- Information must survive transitions between nodes.

Brightboard is being developed with these considerations in mind, with a focus on reliability, traceability, and human oversight.
