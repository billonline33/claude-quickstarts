# Tech Stack & Dependencies

## Core Framework

| Package | Version | Purpose |
|---|---|---|
| `next` | 14.2.5 | App framework — uses App Router pattern |
| `react` | ^18 | UI library |
| `react-dom` | ^18 | React DOM rendering |
| `typescript` | ^5 | Strict-mode TypeScript throughout |

Node.js >= 18.17.0 required.

---

## AI & Cloud SDKs

| Package | Version | Purpose |
|---|---|---|
| `@anthropic-ai/sdk` | ^0.27.1 | Primary Claude API client used in the API route |
| `@ai-sdk/anthropic` | ^0.0.34 | Vercel AI SDK Anthropic provider (imported but currently supplementary) |
| `ai` | ^3.2.38 | Vercel AI SDK core utilities |
| `@aws-sdk/client-bedrock-agent-runtime` | ^3.621.0 | AWS Bedrock Knowledge Base RAG retrieval |

### How each AI library is actually used

- **`@anthropic-ai/sdk`** — instantiated directly in `app/api/chat/route.ts` as `new Anthropic(...)`, calls `messages.create()` with `model`, `max_tokens`, `system`, `messages`, and `temperature`
- **`@aws-sdk/client-bedrock-agent-runtime`** — used in `app/lib/utils.ts` via `RetrieveCommand` to query a Bedrock Knowledge Base and return top-N document chunks with relevance scores
- **`ai` / `@ai-sdk/anthropic`** — imported but the main chat flow uses `@anthropic-ai/sdk` directly

---

## UI Components & Styling

| Package | Version | Purpose |
|---|---|---|
| `tailwindcss` | ^3.4.1 | Utility-first CSS framework |
| `tailwindcss-animate` | ^1.0.7 | Keyframe animation utilities (fade-in, shimmer, accordion) |
| `postcss` | ^8 | CSS processing pipeline |
| `next-themes` | ^0.3.0 | Dark/light/system theme switching with `ThemeProvider` |
| `shadcn/ui` | (config-driven) | Pre-built component wrappers around Radix UI primitives |
| `@radix-ui/react-avatar` | ^1.1.0 | Accessible avatar component |
| `@radix-ui/react-dialog` | ^1.1.1 | Accessible modal/dialog (used in FullSourceModal) |
| `@radix-ui/react-dropdown-menu` | ^2.1.1 | Accessible dropdown (model/KB selectors) |
| `@radix-ui/react-slot` | ^1.1.0 | Polymorphic slot for composable components |
| `lucide-react` | ^0.417.0 | SVG icon library (14 icons used: User, DollarSign, Zap, Wrench, etc.) |
| `class-variance-authority` | ^0.7.0 | Type-safe CSS variant composition for shadcn components |
| `clsx` | ^2.1.1 | Conditional className builder |
| `tailwind-merge` | ^2.4.0 | Merges conflicting Tailwind classes |

### Theme system
7 color themes × 2 modes (light/dark) = 14 CSS variable sets, defined in `styles/themes.js` and loaded dynamically from `TopNavBar.tsx`. Persisted in `localStorage` as `colorTheme`.

---

## Content Rendering

| Package | Version | Purpose |
|---|---|---|
| `react-markdown` | ^9.0.1 | Renders Claude's markdown responses in the chat |
| `rehype-highlight` | ^7.0.0 | Syntax highlighting for code blocks via highlight.js |
| `rehype-raw` | ^7.0.0 | Allows raw HTML within markdown content |

---

## Data Validation

| Package | Version | Purpose |
|---|---|---|
| `zod` | (transitive) | Validates Claude's JSON response shape in `app/api/chat/route.ts` before returning to client |

---

## DevDependencies

| Package | Version | Purpose |
|---|---|---|
| `@types/node` | ^20 | Node.js type definitions |
| `@types/react` | ^18 | React type definitions |
| `@types/react-dom` | ^18 | React DOM type definitions |
| `eslint` | ^8 | Linter |
| `eslint-config-next` | 14.2.5 | Next.js ESLint ruleset (`next/core-web-vitals`) |

---

## Runtime Built-ins Used

- `crypto.randomUUID()` — generates message IDs in the API route
- `performance.now()` — measures latency at multiple stages of the request pipeline
