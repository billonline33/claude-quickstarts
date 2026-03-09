# Customer Support Agent — Developer Guide

AI-powered customer support chat UI built with Next.js 14, Claude, and AWS Bedrock Knowledge Bases (RAG).

> **Status:** Prototype — provided as-is, not for production use without modification.

---

## Quick Start

```bash
npm install
cp .env.local.example .env.local   # then fill in credentials
npm run dev                         # http://localhost:3000
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 14 (App Router), React 18, TypeScript (strict) |
| AI | Anthropic Claude via `@anthropic-ai/sdk` |
| RAG | AWS Bedrock Knowledge Bases via `@aws-sdk/client-bedrock-agent-runtime` |
| UI | shadcn/ui + Radix UI + Tailwind CSS + lucide-react |
| Themes | 7 color themes × light/dark via `next-themes` + CSS variables |

See [docs/tech-stack.md](docs/tech-stack.md) for full dependency table with versions and usage notes.

---

## Project Structure

```
app/
  api/chat/route.ts          # Core AI logic — RAG + Claude inference
  lib/utils.ts               # AWS Bedrock retrieveContext() helper
  lib/customer_support_categories.json
components/
  ChatArea.tsx               # Main chat UI (model/KB selection, messages)
  LeftSidebar.tsx            # Thinking, mood, category display
  RightSidebar.tsx           # RAG source history with scores
  TopNavBar.tsx              # Logo, theme switcher
config.ts                    # Sidebar feature flags
styles/themes.js             # Theme CSS variable definitions
```

See [docs/project-structure.md](docs/project-structure.md) for the full annotated tree and data flow diagram.

---

## Build / Run Commands

```bash
npm run dev           # Full UI — both sidebars
npm run dev:left      # Left sidebar only
npm run dev:right     # Right sidebar only
npm run dev:chat      # Chat area only (no sidebars)

npm run build         # Production build (full UI)
npm run build:left    # Build — left sidebar only
npm run build:right   # Build — right sidebar only
npm run build:chat    # Build — chat area only

npm run lint          # ESLint (next/core-web-vitals)
```

No automated test suite. Node.js >= 18.17.0 required.

---

## Environment Variables

```bash
# .env.local
ANTHROPIC_API_KEY=sk-ant-...          # From console.anthropic.com

# AWS credentials — note the B-prefix (required for AWS Amplify compatibility)
BAWS_ACCESS_KEY_ID=AKIA...
BAWS_SECRET_ACCESS_KEY=...

# Optional: sidebar feature flags (set automatically by npm scripts)
NEXT_PUBLIC_INCLUDE_LEFT_SIDEBAR=true
NEXT_PUBLIC_INCLUDE_RIGHT_SIDEBAR=true
```

**Note:** `BAWS_` prefix is intentional — AWS Amplify blocks variables starting with `AWS_`.
See [docs/aws-setup.md](docs/aws-setup.md) for full explanation.

---

## AWS Resources Required

| Resource | Purpose |
|---|---|
| **IAM User** | Credentials with `AmazonBedrockFullAccess` policy |
| **S3 Bucket** (us-east-1) | Storage backend for support documents |
| **Bedrock Knowledge Base** (us-east-1) | Indexes S3 docs; ID entered in UI dropdown |

AWS region is hardcoded to `us-east-1` in `app/lib/utils.ts`.

See [docs/aws-setup.md](docs/aws-setup.md) for step-by-step IAM, S3, and Bedrock KB setup, plus AWS Amplify deployment instructions.

---

## Key Architecture Notes

- **JSON forcing:** The API route pre-seeds the assistant turn with `{` to guarantee Claude returns structured JSON without preamble (`app/api/chat/route.ts`)
- **Inter-component events:** Sidebars receive data via browser custom events (`updateSidebar`, `updateRagSources`) dispatched by `ChatArea.tsx` — no prop drilling or shared state
- **SSR safety:** Sidebars use `dynamic(..., { ssr: false })` to prevent hydration mismatches
- **RAG flow:** Top-3 Bedrock chunks → injected into system prompt → only highest-score chunk returned in `x-rag-sources` response header
- **Response validation:** Claude's JSON output is validated with Zod before returning to the client

---

## Customization Reference

| What to change | Where |
|---|---|
| Claude model options | `models` array in `components/ChatArea.tsx` |
| Knowledge Base options | `knowledgeBases` array in `components/ChatArea.tsx` |
| System prompt | `systemPrompt` in `app/api/chat/route.ts` |
| Support categories | `app/lib/customer_support_categories.json` + icons in `components/LeftSidebar.tsx` |
| Color themes | `styles/themes.js` |

---

## Further Reading

- [docs/tech-stack.md](docs/tech-stack.md) — All dependencies with versions and usage details
- [docs/project-structure.md](docs/project-structure.md) — Full directory tree, component responsibilities, data flow
- [docs/aws-setup.md](docs/aws-setup.md) — IAM setup, Bedrock KB creation, Amplify deployment
- [docs/api-reference.md](docs/api-reference.md) — POST /api/chat request/response schema and headers
