# ApplyWorth — Case Study

> A global opportunity aggregator with context-injected AI application tools.
> Ingests from 25 heterogeneous sources, deduplicates and health-checks them on a schedule,
> and generates application material grounded in the user's real profile and the real posting.

**Live:** https://www.applyworth.com

> **Why this repository has no source code.** ApplyWorth is a live product with real users.
> The application repository is private. This is a public case study: architecture, schema,
> and the decisions behind them. Happy to walk through any part of the implementation in an interview.

---

## What it does

Seven categories of opportunity — remote jobs, scholarships, fellowships, internships, grants,
exchange programmes, and relocation ("japa") pathways — collected automatically and kept current.

On top of the feed sit **31 AI tools** that write application material: cover letters, statements
of purpose, essays, role-optimised CVs, STAR interview answers, video-essay scripts, negotiation
scripts, diversity statements, research abstracts.

The distinction that matters: every tool receives the user's **full structured profile** *and* the
**specific opportunity description** injected into the prompt. Output is conditioned on both, so it
is not a generic template with a name substituted in.

---

## Architecture at a glance

```
                 ┌──────────────── 25 sources ────────────────┐
                 │  12 API adapters      13 curated RSS feeds │
                 │  Adzuna, Jooble,      OpportunityDesk,      │
                 │  Remotive, RemoteOK,  OpportunitiesFor-     │
                 │  WeWorkRemotely,      Africans, category    │
                 │  Himalayas, Jobicy,   feeds per vertical    │
                 │  Arbeitnow, MyJobMag,                       │
                 │  Jobzilla, Firecrawl ×2                     │
                 └──────────────────┬─────────────────────────┘
                                    │  runAdapter() — per-source isolation
                                    ▼
                    ┌───────────────────────────────┐
                    │  Normalise → slug → dedupe    │
                    │  Claude categorisation pass   │
                    │  (category, deadline, country,│
                    │   funding, organisation)      │
                    └───────────────┬───────────────┘
                                    ▼
                    ┌───────────────────────────────┐
                    │  Postgres (Supabase)          │
                    │  upsert onConflict: slug      │
                    │  ignoreDuplicates: true       │
                    │  20 RLS policies              │
                    └───────────────┬───────────────┘
                                    ▼
        ┌───────────────────────────┼───────────────────────────┐
        ▼                           ▼                           ▼
┌───────────────┐         ┌──────────────────┐        ┌──────────────────┐
│ Personalised  │         │ 31 AI tools      │        │ Scheduled hygiene│
│ feed + explore│         │ Claude SSE stream│        │ link check,      │
│ ISR detail    │         │ + prompt caching │        │ expiry, freshness│
│ pages         │         │ → Document Vault │        │ report, digest   │
└───────────────┘         └──────────────────┘        └──────────────────┘
```

### Stack

| Layer | Choice |
|---|---|
| Framework | Next.js 16 — App Router, Server Components, Turbopack |
| Language | TypeScript, strict mode |
| Database | Postgres via Supabase — RLS, Auth, Storage |
| LLM | Anthropic Claude — Haiku 4.5 in test, Sonnet 4.6 in production, prompt caching on |
| Rate limiting | Upstash Redis, sliding window |
| Email | Resend |
| Payments | Paystack (Africa-first), LemonSqueezy |
| Notifications | Web Push API + VAPID + Service Worker |
| Hosting | Vercel |
| UI | Tailwind CSS v4 + shadcn/ui |

---

## Data model

Ten tables. The shape that matters is that **opportunities are global and public-read**, while
everything user-generated is row-level isolated.

| Table | Role |
|---|---|
| `users` | Identity, mirrored from Supabase Auth |
| `user_profiles` | Structured profile — the left half of every AI prompt |
| `opportunities` | Aggregated postings. `slug` is the natural key; `is_active` is the tombstone |
| `saved_opportunities` | Application tracker: Saved → Applied → Interviewing → Accepted, with notes |
| `ai_artifacts` | Document Vault — every generation persisted and editable in place |
| `ai_tool_usage` | Per-tool usage accounting, drives quota and rate limits |
| `push_subscriptions` | Web Push endpoints per device |
| `subscriptions` | Paid tier state, written by payment webhook |
| `referrals` | Referral attribution |
| `events` | Product analytics |

**20 RLS policies** across nine migrations. Reads of `opportunities` are open; every other table
is scoped to `auth.uid()`. Migrations are numbered and append-only — an existing migration is
never edited, only superseded.

---

## Technical decisions

*Chose X over Y because Z.*

- **Chose `slug` as the dedup key with `ignoreDuplicates: true`, over upserting on a content hash.**
  Twenty-five sources publish the same fellowship with different titles, tracking parameters, and
  formatting. A content hash treats those as distinct rows; a normalised slug collapses them. And
  `ignoreDuplicates` rather than overwrite means a row a human has since corrected is never
  clobbered by a later scrape of the same posting. Dedup runs twice — in memory before the batch,
  then again at the database via the unique constraint — because a single sync run can pull the
  same slug from two adapters simultaneously.

- **Chose two consolidated cron routes over eleven individual ones, while keeping all eleven callable.**
  Vercel's Hobby tier allows two scheduled crons. Rather than pay for the tier or drop jobs, the
  `daily` and `weekly` routes are thin aggregators that call the underlying job modules in sequence.
  Each `/api/cron/<job>` route still exists and is individually invokable, so a single failing job
  can be debugged in isolation without running the whole chain. Every job in `lib/jobs/` exports a
  pure `run()` and knows nothing about HTTP, so scheduling is a deployment concern rather than an
  architectural one.

- **Chose a dead-link tombstone (`is_active = false`) over deleting expired rows.**
  Users save opportunities to their tracker. Hard-deleting a posting that 404s would orphan their
  saved records and destroy their application history. Instead a batched `HEAD` sweep marks dead
  links inactive, a separate job deactivates anything past its deadline, and a weekly freshness
  report flags any source that returned zero rows — which is how a silently broken adapter gets
  caught in days rather than months.

- **Chose lazy client construction over module-level instantiation.**
  The Anthropic, Redis, and Upstash Ratelimit clients are built on first call, not at import. A
  client that throws at module load fails *before* the route handler's `try/catch` exists, so the
  user gets Next.js's default HTML error page instead of a structured JSON error — and on a
  streaming endpoint that means an HTML document arriving where the client expects an SSE stream.

- **Chose to flush a `: warmup` SSE comment before the first real chunk.**
  Errors thrown after the stream has opened surface as structured `data: {"error": "..."}` frames
  the client can render. Errors thrown before it opens surface as an HTTP 500 with an HTML body.
  Flushing a no-op comment immediately moves the boundary, so almost every failure becomes a
  recoverable in-stream error rather than a dead connection. Related: only `Content-Type` and
  `Cache-Control` are set on SSE responses — `Connection: keep-alive` is forbidden under HTTP/2
  and was the root cause of a long-running class of intermittent 500s on Vercel.

---

## Engineering details worth naming

- **Per-source isolation.** Each adapter runs inside `runAdapter()`. One source timing out, changing
  its schema, or rate-limiting the crawler degrades that source only — the sync completes with the
  remaining 24.
- **LLM as a normaliser, not just a generator.** Incoming postings are heterogeneous free text.
  A batched Claude pass extracts category, deadline, country, funding and organisation into a strict
  JSON contract, with date validation on the way out. Cheap model, structured output, validated.
- **Cost control.** Prompt caching on the profile/opportunity prefix, Haiku for non-production paths,
  and per-user sliding-window rate limits backed by Redis.
- **Readiness scoring.** A 0–100 per-opportunity score composed from eligibility match, materials
  already prepared, profile completeness, and engagement — computed server-side.
- **CV ingestion.** PDF and DOCX parsing (`unpdf`, `mammoth`) into the structured profile; export
  back out to PDF and DOCX.

---

## Screenshots

<!-- TODO: add 3–4 screenshots. Suggested: personalised feed, an AI tool mid-stream,
     the Document Vault, the application tracker. Drop the files in /screenshots and
     reference them here. A recruiter scrolls before they read. -->

---

## Author

**Temitope Alabi** — MSc Computer Science (AI), University of Lincoln
[GitHub](https://github.com/toptech5419) · [LinkedIn](https://www.linkedin.com/in/toptech5419/) · alabitemitope51@gmail.com
