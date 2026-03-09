# Project Structure

## Directory Tree

```
customer-support-agent/
├── app/                              # Next.js App Router root
│   ├── api/
│   │   └── chat/
│   │       └── route.ts             # POST /api/chat — core AI logic
│   ├── lib/
│   │   ├── customer_support_categories.json  # 8 categories with keyword lists
│   │   └── utils.ts                 # AWS Bedrock RAG helper (retrieveContext)
│   ├── favicon.ico
│   ├── globals.css                  # CSS custom properties, base styles
│   ├── layout.tsx                   # Root layout: font, metadata, ThemeProvider
│   └── page.tsx                     # Main page: assembles TopNavBar + sidebars + ChatArea
│
├── components/
│   ├── ui/                          # shadcn/ui primitives (do not modify directly)
│   │   ├── avatar.tsx
│   │   ├── button.tsx
│   │   ├── card.tsx
│   │   ├── dialog.tsx
│   │   ├── dropdown-menu.tsx
│   │   ├── input.tsx
│   │   └── textarea.tsx
│   ├── ChatArea.tsx                 # Primary chat UI (model/KB selection, messages, input)
│   ├── FullSourceModal.tsx          # Dialog showing full RAG source content
│   ├── LeftSidebar.tsx              # "Assistant Thinking" panel (mood, categories, reasoning)
│   ├── RightSidebar.tsx             # "Knowledge Base History" panel (RAG sources + scores)
│   ├── TopNavBar.tsx                # Logo, theme selector, dark mode toggle
│   └── theme-provider.tsx           # next-themes ThemeProvider wrapper
│
├── docs/                            # Developer documentation (this folder)
├── lib/
│   └── utils.ts                     # cn() helper (clsx + tailwind-merge)
├── public/                          # Static assets (SVG logos)
├── styles/
│   └── themes.js                    # 7 color themes × 2 modes as CSS variable maps
├── tutorial/                        # Screenshots for README setup guide
│
├── config.ts                        # Feature flags: includeLeftSidebar, includeRightSidebar
├── next.config.mjs                  # Webpack AWS SDK externals, serverComponentsExternalPackages
├── tailwind.config.ts               # Custom animations, CSS variable color scheme
├── components.json                  # shadcn/ui CLI configuration
├── amplify.yml                      # AWS Amplify build config
└── .env.local                       # Local secrets (not committed)
```

---

## Component Responsibilities

### `app/page.tsx`
Assembles the full-page layout. Conditionally renders `LeftSidebar` and `RightSidebar` based on `config.ts` flags. Both sidebars are loaded with `dynamic(..., { ssr: false })` to prevent SSR hydration issues.

### `components/ChatArea.tsx` (725 lines — the main component)
- Maintains `messages[]` state and `conversationHistory[]` for API calls
- Provides model selector (Claude 3 Haiku, Claude Haiku 4.5, Claude 3.5 Sonnet) and Knowledge Base selector
- Sends `POST /api/chat` and handles the JSON response
- Dispatches browser custom events to update sidebars (see Event Bus below)
- Renders suggested questions and "Talk to a human" redirect button
- Tracks perceived latency, network duration, and JSON parse time via `performance.now()`

### `components/LeftSidebar.tsx`
Listens for `updateSidebar` events. Displays:
- Internal reasoning (`thinking`) text
- Detected user mood (positive / negative / curious / frustrated / confused / neutral)
- Matched support categories with icons
- `context_used` debug flag

### `components/RightSidebar.tsx`
Listens for `updateRagSources` events. Displays:
- RAG source file names and snippets
- Relevance score with color coding: >60% green, 40–60% yellow, <40% red
- Opens `FullSourceModal` on click

### `components/TopNavBar.tsx`
- Renders theme-aware logo (light/dark variant)
- 7-color theme switcher that injects CSS variables from `styles/themes.js` into `:root` and persists to `localStorage`

---

## Event Bus (inter-component communication)

Components communicate via browser custom events dispatched on `window`:

| Event | Dispatcher | Listener | Payload |
|---|---|---|---|
| `updateSidebar` | `ChatArea` | `LeftSidebar` | `{ thinking, user_mood, matched_categories, debug }` |
| `updateRagSources` | `ChatArea` | `RightSidebar` | `RAGSource[]` |
| `agentRedirectRequested` | `ChatArea` | `LeftSidebar` | `{ reason }` |
| `humanAgentRequested` | Button in `ChatArea` | (external integrations) | `{ reason }` |

---

## Data Flow

```
User types message
        │
        ▼
ChatArea.tsx
  ├─ builds messages[] + conversationHistory[]
  ├─ POST /api/chat  { messages, model, knowledgeBaseId }
  │
  ▼
app/api/chat/route.ts
  ├─ app/lib/utils.ts → retrieveContext()
  │     └─ AWS Bedrock RetrieveCommand → top-3 KB chunks
  │
  ├─ builds systemPrompt with RAG context injected
  ├─ calls anthropic.messages.create() with pre-seeded "{" for JSON forcing
  ├─ validates response with Zod schema
  └─ returns JSON response + x-rag-sources header
  │
  ▼
ChatArea.tsx (response handling)
  ├─ appends assistant message to messages[]
  ├─ dispatches updateSidebar event  →  LeftSidebar updates
  ├─ dispatches updateRagSources event  →  RightSidebar updates
  └─ triggers redirect button if redirect_to_agent.should_redirect === true
```

---

## Key Configuration Points

### Changing the Claude model
Edit the `models` array in `components/ChatArea.tsx`.

### Changing Knowledge Bases
Edit the `knowledgeBases` array in `components/ChatArea.tsx` — add/remove `{ id, name }` objects.

### Changing the system prompt
Edit `systemPrompt` construction in `app/api/chat/route.ts`.

### Adding/removing support categories
Edit `app/lib/customer_support_categories.json` and update the category icons map in `components/LeftSidebar.tsx`.

### Toggling sidebars at build time
Set `NEXT_PUBLIC_INCLUDE_LEFT_SIDEBAR` / `NEXT_PUBLIC_INCLUDE_RIGHT_SIDEBAR` env vars, or use the npm script variants (`dev:left`, `dev:right`, `dev:chat`).
