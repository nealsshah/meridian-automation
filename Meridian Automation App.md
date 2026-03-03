# Meridian Reddit Action Planner (Next.js) — Dev Doc (Option A, No Auth, No Reddit API)

## 0) Goal

Build a Next.js app that helps a marketer produce **actionable Reddit recommendations** from a single prompt:

* Identify which subreddits to target
* Suggest search queries and thread archetypes
* Generate draft comments/posts (not posted by app)
* Provide cadence + safety/risk notes
* Let marketer review/edit and export an “action plan” checklist

**No authentication. No Reddit API. No Meridian ingestion.**

---

## 1) MVP scope

### MVP features

1. Campaign creation

* Brand name (e.g., Nike)
* Category (e.g., running shoes)
* Goal prompts (e.g., “best running shoes”, “best marathon shoes”)
* Optional: brand voice guidelines + do-not-say list
* Optional: target subreddits (if marketer already knows them)

2. Generate plan (one LLM call)

* Subreddit targets with rationale + risk notes
* Search queries per subreddit (what marketer should type in Reddit search)
* Opportunities (comment-first / post-ok), with:

  * thread archetypes
  * example thread titles (hypothetical)
  * 2–3 draft variants (helpful/data-driven/casual)
  * follow-up replies
  * safety notes

3. Review & edit

* Marketer can edit drafts in-app

4. Export action plan

* Export as Markdown
* Export as JSON

### Out of scope (for now)

* Auth
* Team/workspace roles
* Reddit posting
* Reddit metrics tracking
* Meridian ingestion/citation analysis

---

## 2) Tech stack

* Next.js 14+ (App Router) + TypeScript
* Styling: Tailwind (optional) or basic CSS
* Persistence: **SQLite** (fast) using Prisma

  * You can switch to Postgres later without changing app logic
* LLM: OpenAI or Anthropic (abstracted behind a `llmClient`)

---

## 3) Data model (Prisma)

Keep it minimal.

### Entities

* Campaign: user inputs
* Plan: raw JSON output from LLM
* Opportunity: normalized from Plan for UI editing
* Draft: editable text blocks
* ExportSnapshot: saved exports (optional)

### Prisma schema

```prisma
datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model Campaign {
  id            String   @id @default(cuid())
  brandName     String
  category      String
  goalPrompts   String   // store newline-separated or JSON string
  targetSubs    String?  // newline-separated
  brandVoice    String?  // markdown
  doNotSay      String?  // markdown
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  plan          Plan?
  opportunities Opportunity[]
}

model Plan {
  id          String   @id @default(cuid())
  campaignId  String   @unique
  model       String
  rawJson     Json     // store the full LLM JSON output
  createdAt   DateTime @default(now())
  campaign    Campaign @relation(fields: [campaignId], references: [id])
}

model Opportunity {
  id            String   @id @default(cuid())
  campaignId    String
  type          OpportunityType
  subreddit     String
  priority      Int      @default(50)
  reason        String
  keywords      String   // JSON string array for simplicity in SQLite
  archetype     String?
  status        OpportunityStatus @default(OPEN)
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  drafts        Draft[]
  campaign      Campaign @relation(fields: [campaignId], references: [id])

  @@index([campaignId])
  @@index([status])
  @@index([priority])
}

enum OpportunityType {
  COMMENT
  POST
}

enum OpportunityStatus {
  OPEN
  REVIEWED
  ARCHIVED
}

model Draft {
  id            String   @id @default(cuid())
  opportunityId String
  variant       String   // "helpful" | "data_driven" | "casual"
  title         String?  // only if POST
  body          String
  optionalDisclosureLine String?
  followUpReplies String  // JSON array string
  safetyNotes   String?
  editedByUser  Boolean  @default(false)
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  opportunity   Opportunity @relation(fields: [opportunityId], references: [id])

  @@index([opportunityId])
}
```

---

## 4) App architecture

### Core flow

1. User creates Campaign
2. User clicks “Generate Plan”
3. App calls LLM once with the campaign inputs
4. App stores raw plan JSON + normalizes into Opportunity + Draft rows
5. User reviews/edits
6. User exports action plan (Markdown/JSON)

### File structure

```
/src
  /app
    /page.tsx                     // landing: campaigns list + create
    /campaigns
      /new/page.tsx               // campaign form
      /[id]/page.tsx              // campaign dashboard
      /[id]/opportunities/[oppId]/page.tsx // review/edit drafts
      /[id]/export/page.tsx       // export view
    /api
      /campaigns/route.ts          // CRUD
      /campaigns/[id]/route.ts     // read/update/delete
      /plans/generate/route.ts     // POST generate plan for campaign
      /export/[id]/route.ts        // GET markdown/json export
  /lib
    /db/prisma.ts
    /llm/llmClient.ts
    /llm/prompts.ts
    /plans/normalizePlan.ts
    /plans/validators.ts
    /export/renderMarkdown.ts
    /utils/safeJson.ts
```

---

## 5) LLM “one-shot plan” output contract

You need a strict schema so the UI is stable.

### Plan JSON schema (target)

```ts
type PlanOutput = {
  subreddits: Array<{
    name: string;              // "r/RunningShoeGeeks"
    why: string;
    riskNotes: string;
    postingStyle: "comment-first" | "post-ok";
    ruleWarnings: string[];
  }>;
  searchQueries: Array<{
    subreddit: string;         // "r/RunningShoeGeeks"
    queries: string[];         // ["best daily trainer", ...]
    threadArchetypes: string[];// ["Beginner asking for recs", ...]
  }>;
  opportunities: Array<{
    type: "COMMENT" | "POST";
    subreddit: string;
    priority: number;          // 0-100
    reason: string;
    targeting: {
      keywords: string[];
      archetype: string;
    };
    threadExamples: Array<{
      exampleTitle: string;
      whyThisThread: string;
      mustAvoid: string[];
    }>;
    drafts: Array<{
      variant: "helpful" | "data_driven" | "casual";
      title?: string;
      body: string;
      optionalDisclosureLine?: string;
      followUpReplies: string[];
      safetyNotes?: string;
    }>;
  }>;
  dailyCadence: {
    maxCommentsPerDay: number;
    maxPostsPerWeek: number;
    rotationStrategy: string;
  };
};
```

---

## 6) The single LLM prompt (use as your MVP engine)

Put this in `src/lib/llm/prompts.ts`.

```text
You are a growth strategist optimizing a brand’s presence in LLM answers by participating on Reddit in a helpful, non-spammy way.

Brand: {{brandName}}
Category: {{category}}

Goal prompts (user intent queries):
{{goalPrompts}}

Optional target subreddits (if provided):
{{targetSubs}}

Brand voice guidelines (if provided):
{{brandVoice}}

Do-not-say / forbidden phrases (if provided):
{{doNotSay}}

Hard rules:
- Do NOT be salesy or promotional.
- Do NOT attack competitors.
- Do NOT claim things you can’t verify.
- Prefer value-first, practical advice.
- Provide comment-first suggestions; only include posts where appropriate.
- Output MUST be valid JSON and MUST match the exact schema requested (no extra keys).
- Do not include real Reddit URLs (assume you cannot browse). Use hypothetical thread examples only.

Task:
Create a Reddit execution plan that a marketer can manually execute to increase authentic brand mentions and helpful presence for the goal prompts.

Return JSON in this exact schema:
{ ...PlanOutput schema exactly... }
```

**Important:** explicitly forbid real URLs since you’re not browsing.

---

## 7) Plan generation endpoint

### `POST /api/plans/generate`

Body: `{ campaignId: string }`

Steps:

1. Load campaign
2. Build prompt with inputs
3. Call LLM
4. Parse JSON safely
5. Validate against schema (Zod)
6. Store `Plan.rawJson`
7. Normalize:

   * Upsert Opportunities
   * Create Drafts for each opportunity
8. Return `{ ok: true, campaignId }`

### Zod validator

Create `src/lib/plans/validators.ts` using `zod` to enforce the plan schema.

If validation fails:

* Store the raw text response in logs
* Return a friendly error with “Regenerate” option

---

## 8) Normalization logic

`src/lib/plans/normalizePlan.ts`

* Delete existing opportunities/drafts for campaign (or archive them) before inserting new ones.
* For each `opportunity`:

  * Create Opportunity with `priority`, `reason`, `subreddit`, `type`, `keywords`, `archetype`
  * Create Draft rows for each draft variant

---

## 9) UI requirements

### Campaign list (`/`)

* List existing campaigns
* Button “New campaign”

### New campaign form (`/campaigns/new`)

Fields:

* Brand name (required)
* Category (required)
* Goal prompts (textarea, required) — one per line
* Target subreddits (optional textarea) — one per line
* Brand voice (optional)
* Do-not-say (optional)

Submit → create campaign → redirect to campaign page

### Campaign dashboard (`/campaigns/[id]`)

* Summary of inputs
* Button: **Generate Plan**
* Show:

  * Subreddit recommendations
  * Cadence suggestions
  * Opportunities table:

    * priority, type, subreddit, archetype, status
* Click an opportunity to review drafts

### Opportunity review (`/campaigns/[id]/opportunities/[oppId]`)

* Opportunity details (reason, keywords, must-avoid bullets from threadExamples)
* Draft tabs (helpful/data_driven/casual)
* Editable fields:

  * title (if exists)
  * body
  * disclosure line
  * follow-up replies
* Button: “Mark reviewed” or “Archive”

### Export (`/campaigns/[id]/export`)

* Export Markdown preview
* Buttons:

  * Copy Markdown
  * Download JSON
  * Copy JSON

---

## 10) Export format (Markdown)

`src/lib/export/renderMarkdown.ts`

Markdown sections:

* Campaign info (brand, category, prompts)
* Cadence plan
* Subreddits list (why + risks)
* Search queries per subreddit
* Opportunities (sorted by priority desc)

  * For each: suggested action + archetype + keywords
  * Drafts (with variant headings)
  * Safety notes

Example structure:

```md
# Reddit Action Plan — Nike (Running Shoes)

## Goal prompts
- best running shoes
- best marathon shoes

## Cadence
- Max comments/day: 2
- Max posts/week: 1
- Rotation: ...

## Subreddits to target
### r/RunningShoeGeeks
Why: ...
Risks: ...
Rule warnings: ...

## Search queries
- r/RunningShoeGeeks:
  - “best daily trainer”
  - “marathon shoe recs”
  Archetypes: ...

## Opportunities
### (Priority 87) COMMENT — r/RunningShoeGeeks — “Beginner asking for recs”
Reason: ...
Keywords: ...

#### Draft (helpful)
Body: ...
Optional disclosure: ...
Follow-ups:
- ...
Safety notes: ...
```

---

## 11) LLM client abstraction

`src/lib/llm/llmClient.ts`

Implement as:

* `generatePlan(inputs): Promise<PlanOutput>`

Support provider selection via env:

* `LLM_PROVIDER=openai|anthropic`
* Keep implementation minimal and swappable.

---

## 12) Environment variables

For SQLite + local dev:

```
DATABASE_URL="file:./dev.db"

LLM_PROVIDER="openai"
OPENAI_API_KEY="..."
OPENAI_MODEL="gpt-4.1-mini"  // example

# or Anthropic
ANTHROPIC_API_KEY="..."
ANTHROPIC_MODEL="claude-3-7-sonnet-latest"
```

---

## 13) Build steps (from scratch)

1. Create app

* `npx create-next-app@latest meridian-reddit-planner --ts --app`

2. Install deps

* `npm i prisma @prisma/client zod`
* `npm i -D prisma`

3. Prisma init + migrate

* `npx prisma init`
* set SQLite DATABASE_URL
* add schema above
* `npx prisma migrate dev --name init`

4. Implement DB client

* `src/lib/db/prisma.ts`

5. Build CRUD routes

* create/read/update campaign endpoints

6. Build Generate Plan route

* call LLM
* validate JSON
* store plan
* normalize into opportunities/drafts

7. Build UI pages

* list → new → dashboard → opportunity edit → export

---

## 14) “Fast demo” defaults (so it looks good immediately)

When user creates a campaign, prefill:

* Brand voice: “helpful, practical, no marketing tone”
* Do-not-say: “buy now”, “limited time”, “best on the market”, “#1 guaranteed”

Also create a “Generate sample plan” dev button that loads a fixture JSON (for offline demo).

---

## 15) Definition of Done (MVP)

* User can create a campaign
* Click “Generate Plan” and see:

  * subreddits, queries, opportunities
  * 2–3 draft variants per opportunity
* User can edit drafts
* User can export Markdown + JSON action plan