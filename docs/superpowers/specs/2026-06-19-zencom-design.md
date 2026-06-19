# Zencom -- Intercom-Style B2B SaaS Customer Support Platform

## Overview

Zencom is a multi-tenant B2B SaaS application modeled after Intercom/Zendesk. It provides embeddable chat widgets, AI-powered support agents, a shared team inbox, knowledge base management, lead capture, and billing -- all scoped to isolated workspaces mapped to Clerk organizations.

**Stack:** Next.js 16 + Tailwind CSS 4 + shadcn/ui + Convex (database/backend) + Clerk (auth/billing) + OpenAI (LLM)

**Architecture:** Hybrid -- the `@convex-dev/agent` component handles AI threads, message history, vector search, and streaming. All other application logic lives in the main `convex/` directory with a monolithic schema.

---

## 1. Multi-Tenancy Model

### Workspace = Clerk Organization

Every Clerk organization maps 1:1 to a Zencom workspace. The Clerk `orgId` is the universal tenant scoping key.

- **Provisioning:** When a user creates an org via Clerk, an `organization.created` webhook fires. A Convex HTTP action creates a `workspaces` record with default free-tier settings.
- **Isolation:** Every tenant-scoped table includes an `orgId: v.string()` field with a `by_orgId` index. All queries filter by `orgId` derived from the authenticated user's active organization.
- **Roles:** Two Clerk roles -- `org:admin` (full access) and `org:member` (inbox access). Admins control KB, widget customizer, billing, and team settings.

### Auth Wrapper Pattern

Custom `authedQuery` and `authedMutation` wrappers (using `convex-helpers/server/customFunctions`) extract the user identity via `ctx.auth.getUserIdentity()`, look up the user's workspace by `tokenIdentifier`, and inject `{ user, workspace }` into the handler context. Every public Convex function uses these wrappers.

### Widget Visitors (Unauthenticated)

Widget visitors do not authenticate through Clerk. They receive a `visitorId` (UUID generated client-side, stored in localStorage on the customer's site). Widget Convex functions validate the `workspaceId` parameter against existing workspaces and rate-limit by visitor session.

---

## 2. Data Architecture

### Schema Tables

All tables use flat, relational design with ID references and indexed foreign keys.

#### Core Tables

**workspaces**
- `orgId: v.string()` -- Clerk organization ID (unique)
- `name: v.string()`
- `plan: v.union(v.literal("free"), v.literal("pro"), v.literal("enterprise"))`
- `subscriptionId: v.optional(v.string())` -- Clerk subscription ID
- `subscriptionStatus: v.optional(v.string())`
- `createdAt: v.number()`
- Indexes: `by_orgId` (unique lookup)

**users**
- `tokenIdentifier: v.string()` -- from Clerk identity
- `clerkUserId: v.string()`
- `orgId: v.string()`
- `name: v.string()`
- `email: v.string()`
- `imageUrl: v.optional(v.string())`
- `role: v.union(v.literal("admin"), v.literal("member"))`
- Indexes: `by_tokenIdentifier`, `by_orgId`, `by_clerkUserId`

#### Chat Tables

**conversations**
- `orgId: v.string()`
- `visitorId: v.id("visitors")`
- `status: v.union(v.literal("open"), v.literal("closed"))`
- `assignedTo: v.optional(v.id("users"))` -- human agent assignment
- `assignedType: v.union(v.literal("ai"), v.literal("human"))`
- `agentThreadId: v.optional(v.string())` -- @convex-dev/agent thread ID
- `lastMessageAt: v.number()`
- `lastMessagePreview: v.string()`
- `unreadCount: v.number()` -- unread by agents
- `createdAt: v.number()`
- Indexes: `by_orgId`, `by_orgId_and_status`, `by_orgId_and_assignedTo`, `by_orgId_and_lastMessageAt`

**messages**
- `orgId: v.string()`
- `conversationId: v.id("conversations")`
- `senderType: v.union(v.literal("visitor"), v.literal("agent"), v.literal("ai"))`
- `senderId: v.optional(v.string())` -- visitorId or userId depending on senderType
- `content: v.string()`
- `citations: v.optional(v.array(v.object({ title: v.string(), articleId: v.optional(v.id("articles")), documentId: v.optional(v.id("documents")), snippet: v.string() })))`
- `isStreaming: v.optional(v.boolean())` -- true while AI is streaming
- `createdAt: v.number()`
- Indexes: `by_conversationId`, `by_orgId`

**visitors**
- `orgId: v.string()`
- `visitorId: v.string()` -- UUID from client
- `name: v.optional(v.string())`
- `email: v.optional(v.string())`
- `phone: v.optional(v.string())`
- `lastSeenAt: v.number()`
- `metadata: v.optional(v.object({ userAgent: v.optional(v.string()), referrer: v.optional(v.string()), pageUrl: v.optional(v.string()) }))`
- Indexes: `by_orgId`, `by_orgId_and_visitorId`

#### Knowledge Base Tables

**articles**
- `orgId: v.string()`
- `title: v.string()`
- `slug: v.string()`
- `content: v.string()` -- markdown
- `category: v.string()`
- `coverImageId: v.optional(v.id("_storage"))`
- `isPublished: v.boolean()`
- `isPopular: v.boolean()`
- `createdAt: v.number()`
- `updatedAt: v.number()`
- Indexes: `by_orgId`, `by_orgId_and_isPublished`, `by_orgId_and_category`, `by_orgId_and_slug`
- Search index: `search_title_content` on `title` (filter by `orgId`)

**documents**
- `orgId: v.string()`
- `fileName: v.string()`
- `fileId: v.id("_storage")`
- `contentType: v.string()`
- `status: v.union(v.literal("processing"), v.literal("ready"), v.literal("error"))`
- `errorMessage: v.optional(v.string())`
- `chunkCount: v.optional(v.number())`
- `createdAt: v.number()`
- Indexes: `by_orgId`, `by_orgId_and_status`

**documentChunks**
- `orgId: v.string()`
- `sourceType: v.union(v.literal("article"), v.literal("document"))`
- `sourceId: v.string()` -- article ID or document ID
- `chunkText: v.string()`
- `chunkIndex: v.number()`
- `createdAt: v.number()`
- Indexes: `by_orgId`, `by_sourceId`

(Vector embeddings are managed internally by the `@convex-dev/agent` component.)

#### Lead Management Tables

**leads**
- `orgId: v.string()`
- `visitorId: v.id("visitors")`
- `conversationId: v.optional(v.id("conversations"))`
- `name: v.string()`
- `email: v.string()`
- `phone: v.optional(v.string())`
- `status: v.union(v.literal("new"), v.literal("contacted"), v.literal("closed"))`
- `source: v.union(v.literal("widget_form"), v.literal("chat"))`
- `createdAt: v.number()`
- `updatedAt: v.number()`
- Indexes: `by_orgId`, `by_orgId_and_status`, `by_orgId_and_createdAt`

#### Widget Configuration Tables

**widgetConfigs**
- `orgId: v.string()` -- one per workspace
- `appearance: v.object({ primaryColor: v.string(), textColor: v.string(), backgroundColor: v.string(), borderRadius: v.number(), launcherPosition: v.union(v.literal("bottom-right"), v.literal("bottom-left")), title: v.string(), subtitle: v.optional(v.string()), logoId: v.optional(v.id("_storage")), marginBottom: v.number(), marginSide: v.number() })`
- `behavior: v.object({ proactiveEnabled: v.boolean(), proactiveDelaySeconds: v.number(), proactiveMessage: v.optional(v.string()), leadCaptureEnabled: v.boolean(), leadCaptureFields: v.array(v.union(v.literal("name"), v.literal("email"), v.literal("phone"))), soundEnabled: v.boolean() })`
- `faqArticleIds: v.optional(v.array(v.id("articles")))` -- pinned FAQ articles
- `createdAt: v.number()`
- `updatedAt: v.number()`
- Indexes: `by_orgId` (unique lookup)

#### Operational Tables

**usageCounters**
- `orgId: v.string()`
- `period: v.string()` -- "2026-06" format (monthly)
- `aiMessages: v.number()`
- `kbDocuments: v.number()`
- `kbArticles: v.number()`
- Indexes: `by_orgId_and_period`

**presenceState** (high-churn, separated from stable data)
- `orgId: v.string()`
- `userId: v.id("users")`
- `conversationId: v.optional(v.id("conversations"))` -- which conversation they're viewing
- `isTyping: v.boolean()`
- `lastHeartbeat: v.number()`
- Indexes: `by_orgId`, `by_conversationId`

**conversationReads** (high-churn, per-agent read tracking)
- `userId: v.id("users")`
- `conversationId: v.id("conversations")`
- `lastSeenMessageId: v.optional(v.id("messages"))`
- `lastSeenAt: v.number()`
- Indexes: `by_userId_and_conversationId`, `by_conversationId`

---

## 3. Billing, Plans, and Feature Gating

### Plan Structure (Clerk Organization Plans with Seat Limits)

| Attribute | Free (`org:free`) | Pro (`org:pro`, $49/mo) | Enterprise (`org:enterprise`, $199/mo) |
|---|---|---|---|
| Seats | 1 | 5 | 25 |
| AI messages/mo | 100 | 2,000 | Unlimited |
| KB articles | 10 | 100 | Unlimited |
| KB documents | 5 | 50 | 500 |
| Custom branding | No | Yes | Yes |
| Proactive messages | No | Yes | Yes |
| Lead capture | Basic (name+email) | Full + CSV export | Full + CSV export |
| Priority support | No | No | Yes |

### Feature Slugs (Clerk Features, attached to plans)

- `custom_branding` -- Pro, Enterprise
- `proactive_messages` -- Pro, Enterprise
- `csv_export` -- Pro, Enterprise
- `full_lead_capture` -- Pro, Enterprise (phone field + advanced)
- `priority_support` -- Enterprise
- `advanced_analytics` -- Enterprise (future)

### Implementation

1. **Plan creation:** Via `clerk config patch` or Clerk CLI, creating all three as Organization Plans with seat caps in the Organization Plans tab.

2. **Feature gating (frontend):** `has({ feature: 'custom_branding' })` via `auth()` (server) or `useAuth()` (client). Plan-level checks use `has({ plan: 'org:pro' })`.

3. **Usage metering (Convex):** The `usageCounters` table tracks monthly AI message count, KB document count, and KB article count per workspace. Before each AI message or document upload, a mutation checks the counter against plan-specific limits defined in a shared config.

4. **Billing webhooks:** Convex HTTP actions handle Clerk billing events (`subscription.created`, `subscription.updated`, `subscriptionItem.canceled`). These update the `workspaces` table's `plan`, `subscriptionId`, and `subscriptionStatus` fields.

5. **Billing UI:** `<PricingTable for="organization" />` on the pricing page. `<OrganizationProfile />` for org-level billing management (admin-only).

### Workspace Provisioning Flow

```
User signs up -> Creates Clerk org -> organization.created webhook fires
  -> Convex HTTP action creates workspace record (plan: "free")
  -> Creates default widgetConfig
  -> Creates initial usageCounters for current month
  -> User redirected to onboarding/dashboard
```

---

## 4. Chat System and AI Agent

### Architecture

```
Visitor (widget iframe on customer site)
    |
    v
Convex WebSocket (direct, unauthenticated, validated by workspaceId)
    |
    v
conversations + messages tables (real-time subscriptions)
    |
    v
@convex-dev/agent component
    - AI thread management
    - Vector search (RAG)
    - Token-by-token streaming over WebSocket
    - Usage tracking
    - Rate limiting
    |
    v
Agent inbox (authenticated Clerk session, real-time subscriptions)
```

### Conversation Lifecycle

1. **Start:** Visitor opens widget, optionally fills lead capture form, sends first message.
2. **Create:** Mutation creates `visitors` record (if new), `conversations` record (status: open, assignedType: ai), `messages` record, and `leads` record (if lead capture is on).
3. **AI response:** A Convex action invokes the `@convex-dev/agent`:
   - Creates/continues a thread for this conversation
   - Retrieves relevant KB chunks via vector similarity search
   - Generates streaming response with source citations
   - Writes AI response to `messages` table
4. **Ongoing chat:** Each visitor message triggers the AI agent (if assignedType is "ai").
5. **Human takeover:** Agent clicks "Take over" in inbox -> mutation sets `assignedTo` to agent's userId, `assignedType` to "human". AI stops auto-responding.
6. **Close:** Agent marks conversation as closed. Visitor can reopen by sending a new message.

### @convex-dev/agent Component Configuration

```typescript
const supportAgent = new Agent(components.agent, {
  name: "Support Agent",
  chat: openai.chat("gpt-4o-mini"),
  instructions: `You are a helpful customer support agent for {workspaceName}.
    Answer questions using the provided knowledge base context.
    Always cite your sources. If you don't know the answer, say so
    and offer to connect the visitor with a human agent.`,
  tools: { searchKB, escalateToHuman },
});
```

### Real-Time Features

- **Typing indicators:** `presenceState` table with `isTyping` flag, TTL-based cleanup via scheduled function.
- **Presence (who's viewing):** `presenceState` tracks which conversation each agent is viewing.
- **Live badges:** Reactive queries on `conversations` with unread counts.
- **Read tracking:** `conversationReads` table, updated when agent views a conversation.

---

## 5. Knowledge Base and RAG Pipeline

### Sources

1. **Helpdesk articles:** CRUD in dashboard. Markdown, categories, cover images. Published articles appear in widget help center and are embedded for RAG.
2. **Document uploads:** .md, .txt, .pdf files. Ingested and embedded for RAG only (not directly browsable).

### Ingestion Pipeline

```
Article published / Document uploaded
  -> Convex action (Node.js runtime for PDF parsing)
    -> Extract text
    -> Chunk into ~500-token segments with 50-token overlap
    -> Generate embeddings via OpenAI text-embedding-3-small
    -> Store chunks in documentChunks table
    -> Store embeddings via @convex-dev/agent vector search
```

### Retrieval (during AI response generation)

```
Visitor message received
  -> Embed the query
  -> Vector similarity search across workspace's chunks
  -> Return top-k (5-10) relevant chunks
  -> Include in LLM prompt as context
  -> AI generates response with inline citations
```

### Help Center (Widget)

- Searchable article list filtered by category
- Full-text search on article titles (Convex search index)
- Clean markdown reader view
- Only published articles visible

---

## 6. Widget System

### Embed Model

A one-line script tag that creates an iframe:

```html
<script src="https://widget.zencom.dev/loader.js" data-workspace-id="WORKSPACE_CONVEX_ID"></script>
```

The `data-workspace-id` is the Convex document `_id` from the `workspaces` table. The loader creates an iframe pointing to `/widget/[workspaceId]` on the Zencom domain. CSS/JS isolation, no conflicts with host site.

### Widget Pages (Next.js routes, outside auth protection)

- `/widget/[workspaceId]` -- Main widget container
  - Chat tab: real-time messaging
  - Help center tab: article search + reader
  - Lead capture form (configurable)
  - Launcher button with proactive message bubble

### Widget-Convex Connection

The widget loads a lightweight Convex React client pointing to the public Convex deployment URL. No Clerk auth. Queries/mutations validate `workspaceId`.

### Security

- `workspaceId` validation on every query/mutation
- Rate limiting on message sends (per visitor session, configurable per plan)
- No access to draft articles, internal data, or admin functions
- Content sanitization on incoming messages

### Theming

All appearance properties from `widgetConfigs` applied as CSS custom properties in the widget iframe. Dynamic updates via reactive Convex subscriptions.

---

## 7. Agent Inbox

### Layout

Two-pane design: conversation list (left) + thread view (right).

### Conversation List

- Real-time subscription to `conversations` filtered by `orgId`
- Filters: All, Unread, Unassigned, Assigned to me, Closed
- Displays: visitor name/email, last message preview, timestamp, status badges
- Sorted by `lastMessageAt` descending

### Thread View

- Real-time subscription to `messages` for selected conversation
- Message bubbles differentiated by sender type (visitor, AI, human agent)
- AI messages show source citations as expandable references
- Streaming indicator while AI is generating

### Composer

- Text input with markdown support
- Send button
- "Take over from AI" button (when conversation is AI-assigned)

### Presence

- Avatars of agents currently viewing the conversation
- Typing indicators (both sides)
- Online/offline agent status

### Read Tracking

- `conversationReads` table updated on conversation view
- Unread badge on conversation list items
- Global unread count in sidebar navigation

---

## 8. Lead Management

### Lead Capture

Visitors fill out a configurable form in the widget before (or during) chat. Fields: name (required), email (required), phone (optional, Pro+ plans).

### Dashboard

- Sortable, filterable table of leads
- Filters: status (new/contacted/closed), source (widget_form/chat), text search
- Inline status updates (dropdown to change lifecycle stage)
- Click through to the associated conversation
- CSV export (Pro+ plans)
- Cursor-based pagination

### Lifecycle

`new` -> `contacted` -> `closed`

Status transitions are logged. Agents can move leads through stages with one click.

---

## 9. Widget Customizer

### Panels

**Appearance:**
- Primary color, text color, background color (color pickers)
- Border radius (slider)
- Launcher position (bottom-right / bottom-left)
- Title and subtitle text
- Logo upload (Convex file storage)
- Margin controls (bottom, side)

**Behavior:**
- Proactive messages toggle + delay (seconds)
- Proactive message text
- Lead capture toggle + field selection
- Sound on/off
- FAQ article selection (multi-select from published articles)

**Install:**
- Copy-paste embed snippet with workspace ID
- Preview of the snippet

### Live Preview

An iframe in the customizer showing the widget with current settings. Updates in real-time as settings change (before saving).

---

## 10. Marketing Site

### Landing Page

- Animated hero section demonstrating the chat widget
- Feature sections highlighting key capabilities
- Social proof / testimonials placeholder
- CTA to sign up

### Pricing Page

- Plan comparison table (Free / Pro / Enterprise)
- Feature checklist per plan
- FAQ section
- `<PricingTable for="organization" />` for checkout

### Install Page

- Step-by-step setup instructions
- Embed snippet with workspace ID
- Framework-specific integration guides

---

## 11. Phase Breakdown

### Phase 1: Foundation (Auth + Multi-tenancy + Core Schema)

- Install and configure shadcn/ui
- Wire ConvexProvider with Clerk auth in root layout
- Define complete Convex schema (all tables from Section 2)
- Clerk organizations: enable via CLI, configure membership required mode
- Webhook handlers: `organization.created` (provisions workspace), `user.created`/`user.updated` (syncs users)
- Auth wrapper functions (`authedQuery`, `authedMutation` via convex-helpers)
- `convex/auth.config.ts` for Clerk JWT integration
- Dashboard layout: sidebar with navigation, org switcher, header
- Onboarding: sign up -> create org -> redirect to dashboard
- **Exit criteria:** Sign up, create org, workspace provisioned, navigate empty dashboard sections

### Phase 2: Chat Engine + Widget MVP

- Install and configure `@convex-dev/agent` component
- Conversation CRUD: create, list, update status, assign
- Message CRUD: send, list (real-time), mark read
- Visitor management: create/update visitor records
- Widget: React micro-app loaded in iframe
  - Launcher button
  - Chat interface (send/receive messages)
  - Visitor session management (localStorage visitorId)
- Widget hosted route: `/widget/[workspaceId]` (public, no auth)
- Widget loader script
- AI agent: basic OpenAI GPT-4o-mini responses (no RAG yet)
- Streaming responses via @convex-dev/agent
- Inbox: conversation list + thread view + composer
- Human takeover flow
- Typing indicators (presenceState table)
- Basic presence (who's viewing)
- **Exit criteria:** Embed widget, send message, see in inbox, AI responds, human takes over

### Phase 3: Knowledge Base + RAG

- Articles CRUD: create, edit, publish/unpublish, categories, cover images
- Document upload: .md, .txt, .pdf with file storage
- Text extraction (Node.js action for PDF parsing)
- Chunking pipeline (500-token chunks with overlap)
- Embedding generation (OpenAI text-embedding-3-small)
- Vector storage via @convex-dev/agent component
- RAG retrieval integrated into AI agent
- Source citations in AI responses
- Help center tab in widget (search + reader)
- Dashboard KB management page
- **Exit criteria:** Upload doc, create article, ask question in widget, get cited answer

### Phase 4: Lead Management

- Lead capture form in widget (configurable fields)
- Leads table in dashboard (columns: name, email, phone, status, source, date)
- Sorting by any column
- Filters: status, source, text search
- Inline status dropdown (new/contacted/closed)
- Click-through to conversation
- CSV export (with feature gating for Pro+)
- Cursor-based pagination
- **Exit criteria:** Fill form in widget, see lead in dashboard, filter, export CSV

### Phase 5: Widget Customizer

- Widget config CRUD (Convex functions)
- Customizer page: appearance panel, behavior panel, install panel
- Color pickers, sliders, toggles
- Live preview iframe
- Proactive message system (dwell delay trigger in widget)
- Per-workspace theming (CSS custom properties in widget)
- Install snippet generator
- Logo upload to Convex file storage
- **Exit criteria:** Customize widget, see live preview, copy snippet, see themed widget on test site

### Phase 6: Billing + Plans

- Enable Clerk Billing via CLI
- Create Free/Pro/Enterprise organization plans with seat limits
- Create feature slugs, attach to plans
- Feature gating throughout app (server + client checks)
- Usage metering: AI message counter, KB doc/article counters
- Usage limit enforcement (reject when over quota)
- Pricing page with `<PricingTable for="organization" />`
- Billing management via `<OrganizationProfile />`
- Billing webhooks: subscription lifecycle events -> update workspace plan
- Rate limiting on AI calls via @convex-dev/agent rate limiter
- Plan-specific UI (upgrade prompts, limit warnings)
- **Exit criteria:** Subscribe to Pro, features unlock, usage limits enforced on Free

### Phase 7: Polish + Marketing

- Landing page with animated demo widget
- Pricing page with comparison table + FAQ
- Install/setup page
- Browser notifications for agents
- Sound notifications (configurable)
- Team management improvements
- Performance optimization (pagination review, index tuning)
- Error boundaries and loading states
- Mobile responsiveness
- **Exit criteria:** Production-ready marketing site and application

### Phase 8: Advanced (Future)

- Website crawling for KB ingestion (URL input, page crawler, quota per plan)
- Conversation tagging and notes
- Canned responses / macros
- Advanced analytics dashboard
- Multi-channel: email integration
- Custom AI personality settings per workspace
- Conversation satisfaction ratings
- SLA tracking

---

## 12. File Organization

### Convex Backend

```
convex/
  schema.ts                    -- Complete schema definition
  auth.config.ts               -- Clerk JWT provider config
  convex.config.ts             -- App config with @convex-dev/agent component
  http.ts                      -- HTTP router (webhooks)
  lib/
    auth.ts                    -- getCurrentUser, authedQuery, authedMutation
    plans.ts                   -- Plan limits config, usage checking helpers
    errors.ts                  -- Custom error types
  workspaces.ts                -- Workspace CRUD, provisioning
  users.ts                     -- User sync, lookup
  conversations.ts             -- Conversation CRUD, assignment, status
  messages.ts                  -- Message CRUD, streaming
  visitors.ts                  -- Visitor management
  articles.ts                  -- KB article CRUD
  documents.ts                 -- Document upload, processing
  leads.ts                     -- Lead CRUD, lifecycle
  widgetConfigs.ts             -- Widget configuration CRUD
  usage.ts                     -- Usage metering, limit checking
  presence.ts                  -- Typing indicators, online status
  agent.ts                     -- AI agent definition, RAG tools
  webhooks/
    clerk.ts                   -- Clerk webhook handlers (org, user, billing)
```

### Next.js Frontend

```
app/
  layout.tsx                   -- Root layout (ClerkProvider + ConvexProvider)
  page.tsx                     -- Landing page (public)
  pricing/page.tsx             -- Pricing page (public)
  sign-in/[[...sign-in]]/page.tsx
  sign-up/[[...sign-up]]/page.tsx
  onboarding/page.tsx          -- Post-org-creation setup
  dashboard/
    layout.tsx                 -- Authenticated dashboard shell (sidebar, header, org switcher)
    page.tsx                   -- Dashboard home / overview
    inbox/
      page.tsx                 -- Conversation inbox (two-pane)
      [conversationId]/page.tsx -- Deep link to specific conversation
    kb/
      page.tsx                 -- Article list
      new/page.tsx             -- Create article
      [articleId]/edit/page.tsx -- Edit article
      documents/page.tsx       -- Document uploads
    leads/page.tsx             -- Lead management
    customizer/page.tsx        -- Widget customizer
    billing/page.tsx           -- Billing/plan management
    settings/page.tsx          -- Workspace settings, team management
  widget/
    [workspaceId]/page.tsx     -- Widget host page (public, no auth)
  api/
    webhooks/
      clerk/route.ts           -- Clerk webhook receiver (forwards to Convex)
```

### Component Organization

```
components/
  ui/                          -- shadcn/ui components
  dashboard/
    sidebar.tsx
    header.tsx
    org-switcher.tsx
  inbox/
    conversation-list.tsx
    conversation-item.tsx
    thread-view.tsx
    message-bubble.tsx
    composer.tsx
    typing-indicator.tsx
    presence-dots.tsx
  kb/
    article-editor.tsx
    article-list.tsx
    document-uploader.tsx
  leads/
    leads-table.tsx
    lead-status-badge.tsx
  customizer/
    appearance-panel.tsx
    behavior-panel.tsx
    install-panel.tsx
    live-preview.tsx
  widget/
    widget-app.tsx
    launcher.tsx
    chat-view.tsx
    help-center.tsx
    lead-form.tsx
    proactive-message.tsx
  marketing/
    hero.tsx
    feature-section.tsx
    plan-comparison.tsx
```

---

## 13. Key Technical Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Database | Convex | Real-time subscriptions, integrated vector search, TypeScript-native |
| Auth + Billing | Clerk | B2B orgs, seat-based billing, role/permission system built in |
| AI Agent | @convex-dev/agent | Built-in threads, RAG, streaming over WebSocket, usage tracking |
| LLM | OpenAI GPT-4o-mini | Best cost/quality ratio for support. Upgradeable per workspace later. |
| Embeddings | OpenAI text-embedding-3-small | Fast, affordable, good quality for KB retrieval |
| Widget delivery | iframe with direct Convex connection | CSS isolation, real-time without proxy, simple embed snippet |
| Styling | Tailwind CSS 4 + shadcn/ui | Consistent design system, accessible components |
| Visitor identity | Client-side UUID in localStorage | No auth overhead, simple session tracking |

---

## 14. Security Considerations

- All tenant data queries filter by `orgId` derived server-side from auth (never from client args)
- Widget functions validate `workspaceId` existence and workspace active status
- Rate limiting on widget message sends (per visitor session)
- Rate limiting on AI calls (per workspace, via @convex-dev/agent rate limiter)
- Content sanitization on user-submitted messages and article content
- Clerk webhook signatures verified before processing
- Internal functions use `internalMutation`/`internalAction` (not exposed publicly)
- Scheduled functions use `internal.*` references only
- No `userId` accepted as function argument for authorization
- CORS restricted on widget endpoints
