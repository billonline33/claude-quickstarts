# AWS Setup Guide

## Overview

The app uses **AWS Bedrock Knowledge Bases** for RAG (Retrieval-Augmented Generation). This requires:
1. An IAM user with Bedrock permissions
2. An S3 bucket containing your support documents
3. A Bedrock Knowledge Base that indexes the S3 content
4. AWS credentials configured in `.env.local`

AWS region is hardcoded to `us-east-1` in `app/lib/utils.ts`. Your Knowledge Base must be in that region.

---

## Step 1: Create an IAM User

1. Open AWS Console → IAM → Users → **Create user**
2. Attach policy: **`AmazonBedrockFullAccess`**
3. Create **Access Key** (type: *Application running outside AWS*)
4. Save the **Access Key ID** and **Secret Access Key**

---

## Step 2: Create an S3 Bucket and Upload Documents

1. Create an S3 bucket in **us-east-1**
2. Upload your support documents (PDF, TXT, HTML, DOCX, CSV, etc.)
3. The S3 URIs will appear as source references in the RightSidebar after retrieval

---

## Step 3: Create a Bedrock Knowledge Base

1. Open AWS Console → Amazon Bedrock → **Knowledge Bases** → Create
2. **Data source**: select your S3 bucket
3. **Embeddings model**: Amazon Titan Embeddings (recommended)
4. **Vector store**: Amazon OpenSearch Serverless (auto-created) or your own
5. After creation, copy the **Knowledge Base ID** (format: `XXXXXXXXXX`)
6. Run a **sync** to index your documents

---

## Step 4: Configure Environment Variables

Create or update `.env.local` in the project root:

```bash
ANTHROPIC_API_KEY=sk-ant-...          # From console.anthropic.com
BAWS_ACCESS_KEY_ID=AKIA...            # IAM Access Key ID (B-prefix — see note below)
BAWS_SECRET_ACCESS_KEY=...            # IAM Secret Access Key (B-prefix — see note below)
```

### Why the `B` prefix?

AWS Amplify (and some CI systems) **block environment variables starting with `AWS_`** to prevent accidental credential exposure. The app prefixes credentials with `B` to work around this restriction.

In `app/lib/utils.ts`, the client is initialized as:
```typescript
new BedrockAgentRuntimeClient({
  region: "us-east-1",
  credentials: {
    accessKeyId: process.env.BAWS_ACCESS_KEY_ID!,
    secretAccessKey: process.env.BAWS_SECRET_ACCESS_KEY!,
  },
});
```

Do NOT rename the env vars to `AWS_ACCESS_KEY_ID` — this will break Amplify deployments.

---

## Step 5: Add Your Knowledge Base to the UI

In `components/ChatArea.tsx`, find the `knowledgeBases` array and add your KB:

```typescript
const knowledgeBases = [
  { id: "NONE", name: "No Knowledge Base" },
  { id: "YOUR_KB_ID_HERE", name: "My Support Docs" },
];
```

Users select the active Knowledge Base from the dropdown in the chat UI.

---

## AWS Amplify Deployment

The project includes `amplify.yml` for Amplify hosting.

**Required Amplify environment variables** (set in Amplify Console → App settings → Environment variables):
```
ANTHROPIC_API_KEY
BAWS_ACCESS_KEY_ID
BAWS_SECRET_ACCESS_KEY
KNOWLEDGE_BASE_ID      # Optional — baked into build if set
```

**Service role**: The Amplify service role also needs **`AmazonBedrockFullAccess`** attached if you want server-side KB access during SSR.

**Build variants**: The `amplify.yml` `build` phase writes env vars into `.env` at build time, then runs `npm run build`. You can change which variant is built by modifying the build command in Amplify Console.

---

## Troubleshooting

| Issue | Likely cause |
|---|---|
| RAG returns no results | KB not synced, wrong KB ID, or documents not indexed |
| `CredentialsProviderError` | `BAWS_ACCESS_KEY_ID` / `BAWS_SECRET_ACCESS_KEY` not set |
| `ResourceNotFoundException` | KB ID doesn't exist or is in wrong region |
| Empty `x-rag-sources` header | `retrieveContext()` caught an error silently — check server logs |
| Amplify build fails on env vars | Don't use `AWS_` prefix — use `BAWS_` prefix |
