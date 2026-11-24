# Setup guide

These steps assume you have an n8n instance (cloud or self-hosted) that supports Data Tables, a Gmail account you can test with, an OpenAI API key and a Pinecone account.

## 1. Create the Pinecone index

1. In the Pinecone console, create an index named `brightleaf-kb`.
2. It must be a classic vector index where you bring your own vectors. Name the dense vector field `_values`, set the dimension to 1536 and the metric to cosine.
3. Do not choose an integrated embedding model or add text fields. An index with a document schema will fail in n8n with the error "Inserting into a Pinecone index with a document schema is not supported yet".
4. If the console form does not offer a classic option, create the index with the API:

```
curl "https://api.pinecone.io/indexes" \
  -H "Api-Key: YOUR_PINECONE_API_KEY" \
  -H "Content-Type: application/json" \
  -H "X-Pinecone-Api-Version: 2026-07" \
  -d '{"name":"brightleaf-kb","schema":{"fields":{"_values":{"type":"dense_vector","dimension":1536,"metric":"cosine"}}}}'
```

The 1536 dimension has to match the embedding model. Workflows A and B both use the OpenAI embeddings node with its default model, so keep both on the same model if you change it.

## 2. Create the Gmail labels

Create these five labels in Gmail:

- `Brightleaf-Support`
- `AI/Replied`
- `AI/Draft-Ready`
- `AI/Needs-Human`
- `AI/Non-Support`

Then create a Gmail filter that applies `Brightleaf-Support` to the mail you want the workflow to handle. For testing, filter on the address you send test emails from.

## 3. Create the log table

In n8n, open Data Tables and create a table named `Brightleaf Support Log` with these string columns: `logged_at`, `message_id`, `thread_id`, `sender`, `subject`, `category`, `action`, `confidence`, `reason`, `execution_id`.

## 4. Import the workflows

1. In n8n, create a new workflow, open the menu and choose Import from File.
2. Import `workflows/c-error-handler.json` first, then `workflows/a-knowledge-base-ingest.json`, then `workflows/b-support-email-responder.json`.

## 5. Create credentials and attach them

Follow [credentials.md](credentials.md), then open each node listed below and select your credential.

| Workflow | Nodes that need a credential |
| --- | --- |
| A | `Store in Pinecone` (Pinecone), `OpenAI Embeddings` (OpenAI) |
| B | `New Support Email`, `Mark Email as Read`, `Get Thread`, `Save Draft for Human`, `Send Reply in Thread`, `Create Reply Draft`, every `Label ...` node, both `Alert Owner` nodes (all Gmail), `Classifier Model`, `Reply Model`, `OpenAI Chat Model`, `Retrieval Embeddings` (OpenAI), `knowledge_base` (Pinecone) |
| C | `Send Failure Alert` (Gmail) |

## 6. Replace the placeholders

| Placeholder | Where | What to put |
| --- | --- | --- |
| `YOUR_BRIGHTLEAF_SUPPORT_LABEL_ID` | B, `New Support Email`, Label IDs filter | Pick the `Brightleaf-Support` label from the dropdown |
| `YOUR_LABEL_ID_AI_REPLIED` | B, `Label AI/Replied` | Pick `AI/Replied` |
| `YOUR_LABEL_ID_AI_DRAFT_READY` | B, `Label AI/Draft-Ready` | Pick `AI/Draft-Ready` |
| `YOUR_LABEL_ID_AI_NEEDS_HUMAN` | B, `Label AI/Needs-Human (Agent)` and `(Classifier)` | Pick `AI/Needs-Human` |
| `YOUR_LABEL_ID_AI_NON_SUPPORT` | B, `Label AI/Non-Support` | Pick `AI/Non-Support` |
| `YOUR_DATA_TABLE_ID` | B, `Log Run` | Pick `Brightleaf Support Log` from the table dropdown |
| `YOUR_ALERT_EMAIL` | B, `Normalize Email`, field `ownerEmail` | The address that should receive escalation alerts. If you leave it empty, alerts go to the support inbox itself |
| `YOUR_ALERT_EMAIL` | C, `Alert Settings`, field `alertEmail` | The address that should receive failure alerts |

## 7. Load the knowledge base

Open workflow A, check that `Store in Pinecone` is set to Insert Documents, and click Execute workflow. The documents are written into the `brightleaf-kb-v1` namespace. Re-running it clears that namespace first, so you can change the documents and run it again.

## 8. Connect the error handler

Publish workflow C. Then open the settings of A and B and choose `Brightleaf C: Error Handler` as the error workflow. n8n will not accept an error workflow that has not been published.

## 9. Test, then activate

Work through [testing.md](testing.md). Leave `autoSend` on `false` until the drafts look right. When you are happy, publish workflow B.

## Troubleshooting

- **Document schema error from Pinecone:** the index was created with Pinecone's document schema or integrated embedding option. Delete it and recreate it as a classic index (step 1).
- **Dimension mismatch:** the index dimension does not match the embedding model. Use 1536 with the default OpenAI embeddings.
- **The agent says it has no information:** workflow A has not been run, it ran against a different namespace, or the index name in `knowledge_base` does not match.
- **Nothing is picked up:** the email is already read, does not carry the `Brightleaf-Support` label, or the workflow is not published. The trigger only reads unread mail.
- **Gmail credential expired:** reconnect the OAuth credential in n8n. This is the most common reason a polling workflow stops.
- **Built-in AI credits fail on the classifier:** some n8n gateway credits do not support every model operation. Use your own OpenAI credential on the model nodes.
- **Error alerts never arrive:** workflow C is not published, `alertEmail` is empty, or the failing run was a manual test. Error workflows only run for production executions.
- **Alert emails go to the support inbox:** `ownerEmail` in `Normalize Email` is empty, which is the intended fallback.
