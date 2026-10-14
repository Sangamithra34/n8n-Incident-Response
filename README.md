**NGP-049 Incident Ticket Bot**

**Domain: Incident Response** 
A Telegram bot built in n8n. It turns a plain-text incident report into a structured, tracked ticket and keeps a tamper-evident audit log of every decision.


**COMMANDS:**
/incident <description>
An AI Agent extracts the affected asset, category, severity and suggested owner as JSON. The ticket is saved with the ID NGP-049-<sequence>, and the bot replies with a ticket card.

/close <ticket-id>
Sets the ticket status to CLOSED.

Anything else
Sends a help message.


**WORKFLOW OVERVIEW:**
Telegram Trigger -> Switch (command router)
  /incident -> Triage incident -> Record audit trail -> Notify reporter
  /close    -> Close ticket    -> Record audit trail -> Notify reporter
  fallback  -> Send Help

1. Telegram Trigger receives every message sent to the bot.
2. Switch routes by command (/incident, /close, fallback).
3. Triage incident uses an AI Agent with a Structured Output Parser. The output is JSON with affected_asset, category, severity (red, orange or green), suggested_owner and summary. A ticket ID is generated and the row is inserted into the tickets table with status OPEN.
4. Close ticket looks up the ticket ID and updates its status.
5. Record audit trail (Challenge S6) is described below.
6. Notify reporter (Challenge F3) is described below.


**SEVERITY ROUTING:**
A "Route By Severity" Switch node sends each result to one of three Telegram templates:
- RED (Notify Red): CRITICAL, full ticket card, "Escalate immediately to <owner>"
- ORANGE (Notify Orange): MODERATE or MEDIUM
- GREEN (Notify Green): LOW, "Handle in normal queue"


**AUDIT TRAIL:**
Every decision writes one row to the audit_log table:
- ticket_id: ticket the decision belongs to
- action: INCIDENT or CLOSE
- event_ts: timestamp of the decision
- input_hash: SHA-256 of the original input text
- output_hash: SHA-256 of the model output
- prev_hash: row_hash of the previous audit row
- row_hash: SHA-256 of input_hash + output_hash + event_ts + prev_hash
Hashes are computed with n8n's Crypto node (SHA-256). Each row includes the previous row's hash, so editing any old record breaks every later row_hash. That is what makes the log tamper-evident.


**MESSAGE FORMAT:**
Every Telegram reply starts with:
NGP-049 | exec #<execution id> | <dd Mon yyyy HH:mm> IST | <verdict/severity> | <one-line summary>
The execution ID comes from $execution.id, and the time is converted to Asia/Kolkata.


**DATA TABLES:**
tickets: ticket_id, description, affected_asset, category, severity, suggested_owner, status, created_at, closed_at, chat_id

audit_log: ticket_id, action, event_ts, input_hash, output_hash, prev_hash, row_hash


**SETUP:**
1. Create a bot with @BotFather and copy the token.
2. In n8n, add a Telegram API credential and an LLM credential for the AI Agent.
3. Create the tickets and audit_log Data tables with the columns above.
4. Import workflow.json (three dots menu -> Import from file), select your credentials, then Publish the workflow.
5. Send /incident <description> to your bot.
