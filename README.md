# Lumina Smart Home — Customer Support Workflows

This folder contains two n8n workflows used by Lumina Smart Home support to process incoming support emails and generate RMAs.

## Screenshots

### Customer support workflow
![Customer support workflow](screenshots/Customer%20support%20workflow.png)

### RMA code generator workflow
![RMA code generator workflow](screenshots/RMA%20code%20generator%20workflow.png)

### Automated chunking and embedding system
![Automated chunking and embedding system](screenshots/Automated%20chunking%20and%20embedding%20system.png)

## Purpose

These workflows automate Lumina Smart Home's customer support email handling:

- The `Customer support workflow` monitors a Gmail inbox, classifies incoming messages as either customer support related or other, and ignores non-support messages.
- For support-related messages the workflow triggers an AI Agent which uses RAG via the Pinecone knowledge base and an RMA generation tool. The agent uses retrieved knowledge to answer customer questions, and when needed generates an RMA code.
- Generated RMA codes and related details are stored by the `RMA tools for customer support workflow` in a Google Sheet.
- After the AI Agent composes the response, the workflow replies to the customer via Gmail and logs a ticket row in Google Sheets with the fields: `Timestamp | Customer Email | Subject | Resolution | RMA Code`.

## Workflows included
- **Customer support workflow** — automates inbound Gmail processing: classification, RAG knowledge-base lookup, AI agent response generation, optional RMA generation (calls the RMA workflow), replies via Gmail, and logging to Google Sheets. See [workflows/Customer support workflow.json](workflows/Customer support workflow.json).
- **RMA tools for customer support workflow** — lightweight workflow that generates an RMA code (format: `LUM-####`) and saves it with customer details to Google Sheets. See [workflows/RMA tools for customer support workflow.json](workflows/RMA tools for customer support workflow.json).

## How the main flow works
- Trigger: `Gmail Trigger` polling for new messages.
- Classification: `Email classifier` decides if message is support-related.
- AI Agent: produces a strict JSON output with fields `Output`, `resolving_tool`, and `rma_code` (HTML-only `Output`). The agent either:
  - uses the knowledge base (`resolving_tool = "knowledge_base"`) and replies with helpful HTML, or
  - calls the `generate_rma` tool (`resolving_tool = "rma"`) to obtain an RMA, includes shipping/return instructions, and sets `rma_code`.
- Actions: reply to the original message (`Reply to a message`) and append a log row to Google Sheets (`Append row in sheet`).

## RMA workflow details
- Trigger: `executeWorkflowTrigger` (called from the main workflow).
- RMA generation: `RMA Generator` node creates a random code like `LUM-1234` and returns date, email, and reason.
- Storage: writes the RMA and details into a Google Sheet (`Saving customer details and RMA`).

## Credentials & external services required
- Gmail OAuth2 (for reading/replying to customer emails).
- Pinecone vector index (knowledge base) + embeddings provider (Google Gemini credential used here).
- Google Sheets / Google Drive OAuth2 (for logging and optional docs).
- Language model credentials used by LangChain nodes (e.g., Groq / other LM nodes).

Make sure these credentials are configured in your n8n instance before activating the workflows.

## Testing / Local use
- To test the RMA flow manually, open the RMA workflow and use the Execute trigger node with sample input: `Customer Email` and `Reason`.
- To test the full flow, either send a test email to the monitored Gmail account or enable the Gmail trigger and use n8n’s trigger tester.

## RMA example
- Example agent output JSON:

```
{
  "Output": "Hi Alex,<br><br>Since your item is defective, return shipping is free. Please ship the item to the address below and email us a copy of the shipping receipt for full reimbursement.<br><br><b>Return Address:</b><br>Lumina Returns, 123 Tech Blvd, Austin, TX 78701.<br><br>Warm regards,<br>Ms. Sian 🌟<br>Lumina Smart Home Support",
  "resolving_tool": "rma",
  "rma_code": "LUM-1234"
}
```

## Notes & best practices
- The AI Agent enforces a strict JSON-only output format; do not change the agent’s output parser or response schema without updating downstream mappings.
- Keep the `RMA tools` workflow active (it is active by default) and the main `Customer support workflow` enabled only after credentials are verified.
- Store sensitive credentials in n8n credential manager — do not commit secrets to this repo.

## Files
- [workflows/Customer support workflow.json](workflows/Customer support workflow.json)
- [workflows/RMA tools for customer support workflow.json](workflows/RMA tools for customer support workflow.json)



