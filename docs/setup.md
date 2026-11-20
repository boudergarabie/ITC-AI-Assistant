# Setup Guide

This guide assumes that you already have **n8n installed on a VPS** and can access it through a public HTTPS domain. The repository contains workflow exports; it does not install n8n or provision the external services.

## 1. Prepare your VPS-hosted n8n

Before importing the workflows:

- Make sure the n8n editor is reachable through your domain.
- Configure HTTPS with a valid TLS certificate.
- Make sure n8n's public webhook URL points to that HTTPS domain. For a typical reverse-proxy deployment, review the n8n settings such as `N8N_HOST`, `N8N_PROTOCOL`, `N8N_PORT`, `WEBHOOK_URL` and `N8N_EDITOR_BASE_URL` and set them to match your own domain and proxy.
- Restart n8n after changing its environment configuration.

Do not copy example domain values literally. Use your own VPS domain and the deployment method you already use.

## 2. Prepare Supabase

1. Create or open your Supabase project.
2. Enable the `vector` extension (pgvector) in the database.
3. Create the `documents` table expected by the Supabase Vector Store nodes.

A common schema for the default OpenAI embedding configuration is:

```sql
create extension if not exists vector;

create table if not exists public.documents (
  id bigserial primary key,
  content text,
  metadata jsonb,
  embedding vector(1536)
);
```

**Important:** `1536` is the commonly used dimension for `text-embedding-3-small`. Confirm the embedding model selected in your OpenAI Embeddings nodes and make the vector column dimension match that model. Do not use this example dimension if you select a model with a different output dimension.

4. In n8n, create a **Supabase** credential using your project URL and the appropriate Supabase API key.
5. Assign that credential to both Supabase Vector Store nodes in the main workflow.

Use the least-privileged key that supports the operations required by your setup. Do not publish the key.

## 3. Prepare PostgreSQL

The workflows use PostgreSQL for AI Agent chat memory and error logging.

1. Make sure your PostgreSQL server is reachable from the n8n VPS.
2. Create a database or choose an existing database.
3. Create the error log table:

```sql
create table if not exists public.error_logs (
  id bigserial primary key,
  workflow_name text,
  failed_node text,
  error_message text,
  ai_solution text,
  execution_url text,
  created_at timestamptz
);
```

4. In n8n, create a **Postgres** credential with the host, port, database, username and password for your database.
5. Assign it to the `Postgres Chat Memory` node and the `Log Error to Database` node in the error workflow.
6. Ensure the database user has permission to create/use the chat-memory table and to insert rows into `public.error_logs`.

The Chat Memory node uses its configured/default table behavior. If you change its table settings later, keep the database permissions and setup consistent.

## 4. Create the required n8n credentials

Create credentials in **n8n → Credentials**. The `.env.example` file is a reference checklist only; the workflow nodes do not automatically load values from it.

| Credential type | Used by | What you need |
|---|---|---|
| Google Gemini (Google PaLM) | Main AI Agent and error diagnosis model | Gemini API key |
| OpenAI API | Both OpenAI Embeddings nodes | OpenAI API key |
| Supabase API | Both Supabase Vector Store nodes | Supabase project URL and API key |
| Postgres | Chat memory and error logging | PostgreSQL host, database, user, password and port |
| Telegram API | Telegram Trigger, Get a file and Send Telegram Alert | Telegram bot token |

After importing, open each relevant node and select the credential created in your own n8n instance. The credential IDs and names in the export are retained from the source environment; they do not transfer the secret credential data to your instance.

## 5. Import the workflows

1. In n8n, open **Workflows**.
2. Choose **Import from File**.
3. Import `workflows/01-itc-assistant.json`.
4. Import `workflows/02-error-handler.json`.
5. Open the nodes that use credentials and select your own credentials:
   - `Telegram Trigger`
   - `Get a file`
   - `Supabase Vector Store`
   - `Supabase Vector Store1`
   - `Embeddings OpenAI`
   - `Embeddings OpenAI1`
   - `Google Gemini Chat Model`
   - `Postgres Chat Memory`
   - `Send Telegram Alert`
   - `Log Error to Database`
   - `Brain`
6. Save both workflows.

Do not assume that an imported credential reference is connected just because its old name appears in the JSON. Confirm each credential selection in the editor.

## 6. Configure Telegram ingestion

1. Create a Telegram bot with BotFather and copy its bot token.
2. Add the token as a Telegram API credential in n8n.
3. Select that credential in the `Telegram Trigger`, `Get a file` and `Send Telegram Alert` nodes.
4. Activate the main workflow so n8n registers the Telegram webhook.
5. Send each file in `data/` to the bot as a **document attachment**.

The current `If` node only allows ingestion when all three conditions match:

- The sender's Telegram user ID matches the administrator ID already stored in the workflow.
- The incoming message contains a document.
- The document MIME type is exactly `text/markdown`.

The error notification node also has a Telegram chat ID stored in its parameters. Both values are intentionally retained from the original workflow. If you are deploying for a different administrator, update the sender ID in the `If` node and the destination chat ID in `Send Telegram Alert`.

If Telegram does not pass a Markdown attachment through the current MIME-type condition, inspect the incoming Telegram Trigger execution and confirm the actual `message.document.mime_type` value before changing the condition.

## 7. Configure the error workflow

1. Open the main `ITC-Assistant` workflow's settings.
2. Set its **Error Workflow** to the imported error-handling workflow.
3. Save the main workflow.
4. Activate the error-handling workflow.
5. Confirm that the Telegram bot can send a message to the configured administrator chat.

The error workflow uses Gemini to diagnose failures and writes the diagnosis to `public.error_logs`. Make sure the database table and credentials are configured before testing it.

## 8. Activate and test

Test in this order:

1. **Credentials:** confirm every credential-backed node has a valid credential selected.
2. **Ingestion:** send one Markdown file from `data/` to Telegram and check the main workflow execution.
3. **Supabase:** confirm that rows were inserted into `public.documents`.
4. **Chat:** open the Chat Trigger's chat interface and ask a question whose answer is present in the uploaded files.
5. **Memory:** send a follow-up question that depends on the previous message.
6. **Error handling:** perform a controlled test in a safe test workflow and confirm the error is logged and a Telegram alert is received.

Do not test by deliberately breaking a production workflow.

## 9. Security checklist

- Keep API keys, Telegram bot tokens and database passwords out of Git.
- Do not commit a real `.env` file. Only `.env.example` belongs in the repository.
- Use HTTPS for the public n8n URL.
- Restrict access to the n8n editor and secure your VPS and database.
- Review the Telegram administrator ID and alert destination before enabling the workflows.
- Test the assistant with non-sensitive sample data before public use.
