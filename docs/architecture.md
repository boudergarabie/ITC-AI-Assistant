# Architecture

## Overview

ITC AI Assistant is organized around two n8n workflows. The main workflow handles both knowledge-base ingestion and user conversations. The second workflow handles failures and sends diagnostic notifications.

## 1. Knowledge-base ingestion

1. An administrator sends a Markdown file to the Telegram bot as a document attachment.
2. The Telegram Trigger receives the message.
3. The `If` node checks the sender ID, the presence of a document and the MIME type.
4. `Get a file` downloads the document.
5. `Code in JavaScript` reads the original filename and parses the Markdown sections. It extracts the text and metadata fields: `source`, `id`, `title` and `category`.
6. `Recursive Character Text Splitter` splits the extracted content.
7. `Default Data Loader` packages each chunk and its metadata.
8. `Embeddings OpenAI` creates embeddings.
9. `Supabase Vector Store` inserts the documents into the `documents` table.

The four source documents are stored in `data/`. Send them individually through Telegram to ingest them.

## 2. Chat and retrieval

1. A visitor sends a message through the n8n Chat Trigger.
2. The AI Agent uses Google Gemini as its language model.
3. For ITC-related factual questions, the agent queries `Supabase Vector Store1`.
4. OpenAI embeddings are used to search the Supabase vector store.
5. PostgreSQL Chat Memory provides persistent conversation context.
6. The agent responds using the retrieved ITC content and the behavior rules in its system message.

The agent's instructions require it to stay within ITC topics, use retrieved information for factual answers and avoid inventing unsupported details.

## 3. Error handling

1. The Error Trigger receives an execution failure.
2. `Parse Error Payload` extracts workflow, execution, node and error details.
3. `AI Error Diagnosis` asks Google Gemini for a short root-cause explanation and corrective steps.
4. `Log Error to Database` writes the diagnostic information to PostgreSQL's `error_logs` table.
5. `Send Telegram Alert` sends the error and suggested solution to the configured Telegram chat.

Assign the error workflow in the main workflow's settings so that failures are routed to it.

## External dependencies

- A publicly reachable n8n instance with HTTPS.
- Telegram Bot API access.
- Google Gemini API access.
- OpenAI API access for embeddings.
- Supabase with pgvector and a compatible `documents` table.
- PostgreSQL access for chat memory and error logging.

## Important implementation details

- The current Telegram ingestion filter is restricted to the administrator ID stored in the `If` node.
- The Telegram alert node has a configured chat ID in its parameters.
- The workflows retain their original credential references. These references must be mapped to credentials in the importing n8n instance.
- The vector table's embedding dimension must match the embedding model configured in the OpenAI Embeddings nodes.
