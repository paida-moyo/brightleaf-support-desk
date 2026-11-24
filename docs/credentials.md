# Credentials

No secrets live in this repository. You create each credential in your own n8n instance and attach it to the nodes listed in [setup.md](setup.md).

## Gmail (OAuth2)

- **Used by:** every Gmail node in B and C.
- **What it needs:** permission to read mail, modify labels, create drafts and send mail on the account that receives support email.
- **How to create it:** in n8n, add a Gmail OAuth2 credential and sign in with the account. On n8n Cloud the one-click sign-in handles the Google setup. On a self-hosted instance you create an OAuth client in Google Cloud Console, enable the Gmail API, and paste the client ID and secret into n8n.
- **Note:** use a test account or a dedicated support mailbox while you are building. The trigger only reads mail with the `Brightleaf-Support` label, but the credential itself can see the whole mailbox.

## OpenAI

- **Used by:** `Classifier Model`, `Reply Model`, `OpenAI Chat Model` and `Retrieval Embeddings` in B, and `OpenAI Embeddings` in A.
- **What it needs:** an API key with access to a chat model (the workflows use `gpt-4o-mini`) and to an embeddings model.
- **How to create it:** create a key in the OpenAI platform dashboard, then add an OpenAI credential in n8n and paste it in.
- **Note:** use your own key on the model nodes. The workflows were built with n8n's built-in AI credits at first, and the classifier did not run on them.

## Pinecone

- **Used by:** `Store in Pinecone` in A and `knowledge_base` in B.
- **What it needs:** an API key for the project that holds the `brightleaf-kb` index.
- **How to create it:** in the Pinecone console, open API Keys, create a key, then add a Pinecone credential in n8n and paste it in.
- **Note:** the index must be a classic vector index (1536 dimensions, cosine). See step 1 of the setup guide.

## n8n Data Table

The `Log Run` node writes to an n8n Data Table, which needs no separate credential. You only select the table in the node.
