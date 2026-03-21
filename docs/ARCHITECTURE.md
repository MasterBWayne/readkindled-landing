# ReadKindled — Technical Architecture Spec
**Version:** 1.0
**Author:** Selina (Technical Lead)
**Date:** 2026-03-21
**Status:** Draft — awaiting Bruce review

---

## Table of Contents
1. [System Overview](#1-system-overview)
2. [Tech Stack](#2-tech-stack)
3. [Data Model](#3-data-model)
4. [Layer 1: Reader App (B2C)](#4-layer-1-reader-app)
5. [Layer 2: Author Dashboard (B2B SaaS)](#5-layer-2-author-dashboard)
6. [Layer 3: Recommendation Engine](#6-layer-3-recommendation-engine)
7. [Privacy Architecture](#7-privacy-architecture)
8. [MVP Scope](#8-mvp-scope)
9. [Supabase Schema](#9-supabase-schema)

---

## 1. System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      ReadKindled Platform                    │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐ │
│  │  Reader App   │  │   Author     │  │  Recommendation   │ │
│  │  (B2C)        │  │   Dashboard  │  │  Engine           │ │
│  │              │  │   (B2B SaaS)  │  │  (Netflix Layer)  │ │
│  │  • Intake     │  │              │  │                   │ │
│  │  • Reading    │  │  • Heatmaps  │  │  • Outcome-based  │ │
│  │  • Coaching   │  │  • Scores    │  │  • Profile match  │ │
│  │  • Check-ins  │  │  • Rewrites  │  │  • Affiliates     │ │
│  │  • Exercises  │  │  • Segments  │  │                   │ │
│  └──────┬───────┘  └──────┬───────┘  └────────┬──────────┘ │
│         │                 │                    │            │
│  ┌──────┴─────────────────┴────────────────────┴──────────┐ │
│  │                   Supabase Backend                      │ │
│  │  Auth │ Postgres │ Edge Functions │ Realtime │ Storage  │ │
│  └──────────────────────┬──────────────────────────────────┘ │
│                         │                                    │
│  ┌──────────────────────┴──────────────────────────────────┐ │
│  │                    AI Layer                              │ │
│  │  Gemini (coaching) │ Claude (empathy) │ Embeddings (rec) │ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Tech Stack

### Decision: React Native (Expo) — not Flutter

| Factor | React Native (Expo) | Flutter |
|---|---|---|
| **Existing codebase** | Loop App is React + TS. Direct code sharing. | Full rewrite. |
| **Web support** | First-class via Expo Web / React Native Web | Possible but weaker ecosystem |
| **AI SDK ecosystem** | All AI SDKs are JS/TS-first | Dart wrappers lag behind |
| **Hiring** | Easier to find RN devs | Smaller pool |
| **Supabase** | Official JS SDK, battle-tested | Community Dart SDK |
| **Bruce's stack** | Already knows React + TS | Would need to learn Dart |

**Verdict:** React Native with Expo. Share components with existing Loop App. Ship to iOS, Android, and Web from one codebase.

### Full Stack

| Layer | Technology | Why |
|---|---|---|
| **Mobile + Web** | React Native (Expo) + TypeScript | Code sharing with Loop, single codebase |
| **Backend** | Supabase (Postgres + Auth + Edge Functions + Realtime) | Already using it, zero ops |
| **AI Coaching** | Gemini 2.5 Flash (fast/cheap) + Claude Sonnet (empathy-critical) | Same dual-model pattern as Loop |
| **Embeddings** | Gemini text-embedding-005 or OpenAI text-embedding-3-small | For recommendation engine similarity search |
| **Vector Store** | Supabase pgvector extension | No additional infra |
| **Push Notifications** | Expo Notifications + Supabase Edge Functions (cron) | Built into Expo |
| **Author Dashboard** | Next.js (separate app) or Expo Web route group | Depends on MVP scope |
| **Payments** | RevenueCat (mobile) + Stripe (web/author SaaS) | Industry standard |
| **Analytics** | PostHog (self-hostable, privacy-first) | Better than Mixpanel for this use case |
| **CDN/Hosting** | Vercel (web) + EAS (mobile builds) | Already using Vercel |

---

## 3. Data Model

### Entity Relationship Overview

```
users ─────────────┬── reading_sessions ── chapters ── books
                   ├── intake_profiles
                   ├── check_ins
                   ├── exercises
                   ├── outcomes
                   ├── emotional_snapshots
                   └── recommendations

books ─────────────┬── chapters
                   ├── exercises (templates)
                   └── author_accounts (FK)

author_accounts ───┬── author_book_access
                   └── (reads aggregated views only)
```

### Core Principle
Every user interaction generates a data point. The reader gets coaching. The author gets intelligence. The engine gets training signal. Same event, three consumers.

---

## 4. Layer 1: Reader App (B2C)

### 4.1 Intake Flow

Purpose: Build a rich user profile BEFORE they start reading. This is the data that powers personalization AND the recommendation engine.

**Intake steps:**
1. **Life areas** — multi-select: career, relationships, health, money, identity, family, habits, purpose
2. **Current struggle** — free text: "What's the #1 thing you want to change?"
3. **Past attempts** — "Have you tried to fix this before? What happened?"
4. **Emotional baseline** — 5-point scale on: anxiety, clarity, motivation, confidence, peace
5. **Reading style** — "How do you prefer to learn?" (direct/reflective/mix — reused from Loop)
6. **Goal** — "In 30 days, what would 'better' look like for you?"

Stored in `intake_profiles`. Referenced by AI coaching for every interaction.

### 4.2 Reading Session Tracking

Every chapter open/close generates a `reading_session`:
- `started_at`, `ended_at` (duration)
- `completion_pct` (scroll depth or explicit "done" tap)
- `emotional_state_before` (quick 1-5 check pre-chapter)
- `emotional_state_after` (quick 1-5 check post-chapter)
- `highlights` (user-marked passages, stored as text + position)
- `ai_interactions` (count of coaching messages during this session)

### 4.3 AI Coaching Companion

Same architecture as Loop's Gemini service but with book context:

```
System prompt:
- User's intake profile (struggles, goals, style)
- Current book + chapter content (key concepts)
- User's reading history (what they've completed)
- Exercise responses (what they've written)
- Emotional trajectory (are they improving?)

Rules:
- Translate book concepts to user's specific life
- Reference their intake answers by name
- Never generic. Always specific to THEIR situation.
- If emotional state drops 2+ points between sessions, proactively check in
```

### 4.4 Proactive Check-ins (Push Notifications)

Triggered by Supabase Edge Function cron:

| Trigger | Notification |
|---|---|
| 24h since last session | "You left off at [chapter]. 5 minutes to keep the momentum?" |
| Exercise due | "[Book] asked you to try [action] this week. How did it go?" |
| Emotional dip detected | "Noticed things felt heavier last session. Want to talk about it?" |
| 7-day milestone | "One week with [Book]. Here's what's shifted so far." |
| 30-day post-completion | "It's been a month since you finished [Book]. Quick check-in?" |

### 4.5 Exercise Tracking

Each book chapter can have 0-N exercises defined by the content team:
- `exercise_templates` — the prompt/question (defined per chapter)
- `exercise_responses` — user's answer, timestamp, time spent
- `exercise_followups` — AI-generated follow-up based on their response

### 4.6 Before/After Emotional State Tracking

Two measurement points:
1. **Per-session:** Quick 1-5 emotional pulse before and after each chapter
2. **Per-book:** Full emotional baseline at intake, re-measured at book completion and 30-day follow-up

Stored in `emotional_snapshots` with `snapshot_type` = `SESSION_PRE` | `SESSION_POST` | `INTAKE` | `COMPLETION` | `FOLLOWUP_30D`

This creates the **Transformation Effectiveness Score** for Layer 2.

---

## 5. Layer 2: Author Dashboard (B2B SaaS)

### 5.1 What Authors See

Authors NEVER see individual user data. Everything is aggregated with minimum cohort size of 10.

#### Reader Engagement Heatmap
```sql
-- Chapter-by-chapter completion funnel
SELECT 
  c.chapter_number,
  c.title,
  COUNT(DISTINCT rs.user_id) as readers_started,
  COUNT(DISTINCT rs.user_id) FILTER (WHERE rs.completion_pct >= 0.8) as readers_completed,
  AVG(rs.duration_seconds) as avg_time_spent,
  AVG(rs.emotional_state_after - rs.emotional_state_before) as avg_emotional_shift
FROM reading_sessions rs
JOIN chapters c ON c.id = rs.chapter_id
WHERE c.book_id = :book_id
GROUP BY c.chapter_number, c.title
ORDER BY c.chapter_number;
```

Visual: horizontal bar chart per chapter. Green = high completion. Red = abandonment cliff. Size = time spent.

#### Transformation Effectiveness Score (TES)

```
TES = (completion_rate × 0.3) + (exercise_completion_rate × 0.3) + (emotional_improvement × 0.4)

Where:
- completion_rate = % of readers who finished all chapters
- exercise_completion_rate = % of exercises completed by those who finished
- emotional_improvement = avg(completion_snapshot - intake_snapshot) normalized to 0-100
```

Displayed as a single number 0-100 with trend line over time.

#### Audience Segmentation

Authors see which reader PROFILES respond best to their book:
- "Readers struggling with **relationships** had 85% completion and +22 emotional improvement"
- "Readers struggling with **career** had 45% completion — Chapter 6 is where they drop off"

Built from cross-referencing `intake_profiles.life_areas` with `outcomes`.

#### AI-Generated Rewrite Suggestions

When a chapter has:
- Completion rate < 50%, OR
- Emotional state drops (post < pre), OR
- Exercise skip rate > 70%

The system generates a suggestion:

```
System prompt to Claude:
"Chapter 6 of [Book] has a 42% completion rate. 
Readers who quit cited these in their last exercise responses: [aggregated themes].
The chapter covers: [chapter summary].
Average time before abandonment: 4.2 minutes.
Suggest 3 specific changes the author could make to improve engagement and completion."
```

#### "ReadKindled Certified" Badge

Awarded when a book achieves:
- 70%+ readers complete all chapters
- 60%+ complete at least one exercise per chapter
- Average emotional improvement score > 15 points (out of 100)
- Minimum 50 readers in cohort

Badge is displayable on Amazon listings, author websites, social media.

### 5.2 Author Dashboard Backend

```
┌─────────────────────────────────────────┐
│           Author Dashboard              │
│                                         │
│  Next.js app (or Expo Web route group)  │
│         ↓ API calls ↓                   │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │  Supabase Edge Functions          │  │
│  │  /author/engagement-heatmap      │  │
│  │  /author/transformation-score    │  │
│  │  /author/audience-segments       │  │
│  │  /author/rewrite-suggestions     │  │
│  │  /author/certification-status    │  │
│  └───────────────┬───────────────────┘  │
│                  │                       │
│  ┌───────────────┴───────────────────┐  │
│  │  Materialized Views (Postgres)    │  │
│  │  • mv_chapter_engagement         │  │
│  │  • mv_book_outcomes              │  │
│  │  • mv_audience_segments          │  │
│  │  Refreshed every 6 hours         │  │
│  │  Minimum cohort: 10 readers      │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

Key privacy mechanism: **Materialized views**. Authors query pre-aggregated views, never raw tables. Row-Level Security ensures `author_accounts` can only access `mv_*` views filtered to their own `book_id`.

---

## 6. Layer 3: Recommendation Engine

### 6.1 Core Concept

Traditional book recommendations: "People who bought X also bought Y" (purchase correlation).

ReadKindled recommendations: "People with YOUR exact profile who completed X reported the highest transformation scores. You should read Y next." (outcome correlation).

This is the moat. Nobody else has outcome data tied to reader profiles.

### 6.2 How It Works

#### Step 1: Build User Embeddings

When a user completes intake + finishes a book, generate a composite embedding:

```typescript
const userVector = await embed([
  intake.life_areas.join(', '),
  intake.current_struggle,
  intake.goal,
  `completed: ${book.title}`,
  `emotional_change: ${outcome.emotional_improvement}`,
  `top_exercises: ${outcome.most_impactful_exercises.join(', ')}`,
].join(' | '));
```

Store in `user_embeddings` table (pgvector).

#### Step 2: Build Book Outcome Profiles

For each book, compute an aggregate "outcome profile" — what kind of transformation does this book actually deliver?

```typescript
const bookOutcomeVector = await embed([
  `book: ${book.title}`,
  `avg_completion: ${stats.completion_rate}`,
  `strongest_areas: ${stats.top_improving_life_areas.join(', ')}`,
  `avg_emotional_lift: ${stats.avg_emotional_improvement}`,
  `reader_profile_fit: ${stats.best_responding_profiles.join(', ')}`,
].join(' | '));
```

Store in `book_outcome_embeddings`.

#### Step 3: Match

```sql
-- Find books whose outcome profile is most similar to what this user needs
SELECT 
  b.id, b.title, b.author,
  1 - (boe.embedding <=> :user_embedding) as match_score,
  boe.avg_emotional_improvement,
  boe.completion_rate
FROM book_outcome_embeddings boe
JOIN books b ON b.id = boe.book_id
WHERE b.id NOT IN (SELECT book_id FROM reading_sessions WHERE user_id = :user_id)
ORDER BY match_score DESC
LIMIT 5;
```

#### Step 4: Present

```
"Based on your profile and what worked for 847 readers like you:

📖 Recommended next: "The Untethered Soul" by Michael Singer
   → 89% of readers with your profile completed it
   → Average emotional improvement: +28 points
   → Strongest area: Identity & Purpose (your #1 struggle)

[Start Reading]  [Why this book?]"
```

### 6.3 Affiliate Revenue

When a recommendation leads to a book purchase (Amazon affiliate link or in-app purchase):
- Track click → conversion in `recommendation_events`
- Revenue share: ReadKindled takes affiliate commission
- Authors on the platform get preferred recommendation placement
- Data flywheel: more readers → better recommendations → more conversions → more authors join

---

## 7. Privacy Architecture

### Principles
1. **Authors never see individual data.** All views are aggregated (min cohort 10).
2. **Users own their data.** Export and delete at any time (GDPR/CCPA compliant).
3. **Embeddings are one-way.** User embeddings cannot be reverse-engineered to reconstruct intake text.
4. **AI coaching conversations are ephemeral.** Stored for 90 days, then deleted. Only structured data (emotional scores, exercise responses) is permanent.

### Implementation

```sql
-- Row Level Security on all tables
-- Users can only see their own data
CREATE POLICY "users_own_data" ON reading_sessions
  FOR ALL USING (auth.uid() = user_id);

-- Authors can only see materialized views for their books
CREATE POLICY "authors_see_aggregates" ON mv_chapter_engagement
  FOR SELECT USING (
    book_id IN (
      SELECT book_id FROM author_book_access 
      WHERE author_id = auth.uid()
    )
  );

-- Materialized views enforce minimum cohort size
CREATE MATERIALIZED VIEW mv_chapter_engagement AS
SELECT 
  chapter_id, book_id,
  COUNT(DISTINCT user_id) as reader_count,
  -- Only show data when cohort >= 10
  CASE WHEN COUNT(DISTINCT user_id) >= 10 
    THEN AVG(completion_pct) ELSE NULL END as avg_completion,
  CASE WHEN COUNT(DISTINCT user_id) >= 10 
    THEN AVG(emotional_state_after - emotional_state_before) ELSE NULL END as avg_emotional_shift
FROM reading_sessions
GROUP BY chapter_id, book_id;
```

---

## 8. MVP Scope

### Goal: First 100 readers + First 1 author on the platform

### What to Build (8-week sprint)

#### Weeks 1-2: Core Reader App
- [ ] Expo project setup (React Native + TypeScript)
- [ ] Supabase auth (email + Apple Sign-In)
- [ ] Intake flow (6 screens)
- [ ] Book library (start with 3-5 curated books — reuse existing Loop Reader content)
- [ ] Chapter reader with scroll tracking
- [ ] Per-session emotional pulse (before/after)

#### Weeks 3-4: AI Coaching + Exercises
- [ ] AI coaching companion (Gemini Flash, context-aware prompts)
- [ ] Exercise templates per chapter
- [ ] Exercise response capture + AI follow-up
- [ ] Reading session persistence (duration, completion %)

#### Weeks 5-6: Check-ins + Outcomes
- [ ] Push notification system (Expo Notifications)
- [ ] Proactive check-in cron (Supabase Edge Functions)
- [ ] Book completion flow + emotional re-measurement
- [ ] 30-day follow-up scheduling
- [ ] Basic outcome tracking

#### Weeks 7-8: Author Dashboard MVP
- [ ] Author signup + book claim flow
- [ ] Chapter engagement heatmap (basic bar chart)
- [ ] Transformation Effectiveness Score (single number)
- [ ] Simple audience segmentation (which life areas respond best)

#### NOT in MVP (Phase 2)
- Recommendation engine (needs data volume first — min 500 completed readers)
- AI rewrite suggestions (needs enough data to be meaningful)
- ReadKindled Certified badge (needs 50+ readers per book)
- Affiliate integration
- Payment/subscription (free during beta)
- Advanced analytics in author dashboard

### MVP Success Metrics
| Metric | Target |
|---|---|
| Readers signed up | 100 |
| Books completed | 30 |
| Avg completion rate | > 60% |
| Exercise completion | > 40% |
| 30-day follow-up response rate | > 25% |
| Authors onboarded | 1 (+ your own curated books) |
| Emotional improvement (avg) | Measurable positive delta |

---

## 9. Supabase Schema

```sql
-- ============================================================
-- ReadKindled — Full Supabase Schema
-- ============================================================

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "vector";  -- pgvector for recommendations

-- ─────────────────────────────────────────────
-- Users (extends Supabase auth.users)
-- ─────────────────────────────────────────────
CREATE TABLE user_profiles (
  id              UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  display_name    TEXT,
  reading_style   TEXT CHECK (reading_style IN ('direct', 'warm', 'balanced')),
  timezone        TEXT DEFAULT 'UTC',
  notifications   BOOLEAN DEFAULT true,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ─────────────────────────────────────────────
-- Intake Profiles (one per user, updated on re-take)
-- ─────────────────────────────────────────────
CREATE TABLE intake_profiles (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id         UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  life_areas      TEXT[] NOT NULL DEFAULT '{}',          -- ['career', 'relationships', ...]
  current_struggle TEXT,                                  -- free text
  past_attempts   TEXT,                                   -- free text
  goal_30day      TEXT,                                   -- free text
  emotional_baseline JSONB NOT NULL DEFAULT '{}',         -- {anxiety: 3, clarity: 2, motivation: 4, confidence: 2, peace: 3}
  version         INTEGER NOT NULL DEFAULT 1,             -- increments on re-take
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE(user_id, version)
);

CREATE INDEX intake_user_idx ON intake_profiles(user_id);

-- ─────────────────────────────────────────────
-- Books
-- ─────────────────────────────────────────────
CREATE TABLE books (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  title           TEXT NOT NULL,
  author_name     TEXT NOT NULL,
  author_id       UUID REFERENCES auth.users(id),         -- NULL if not on platform
  description     TEXT,
  cover_url       TEXT,
  category        TEXT,                                    -- 'personal-development', 'relationships', etc.
  total_chapters  INTEGER NOT NULL DEFAULT 1,
  is_published    BOOLEAN DEFAULT false,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ─────────────────────────────────────────────
-- Chapters
-- ─────────────────────────────────────────────
CREATE TABLE chapters (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  book_id         UUID NOT NULL REFERENCES books(id) ON DELETE CASCADE,
  chapter_number  INTEGER NOT NULL,
  title           TEXT NOT NULL,
  content         TEXT,                                    -- full chapter text or structured JSON
  key_concepts    TEXT[],                                  -- AI-extracted key ideas
  estimated_minutes INTEGER DEFAULT 10,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE(book_id, chapter_number)
);

CREATE INDEX chapters_book_idx ON chapters(book_id);

-- ─────────────────────────────────────────────
-- Exercise Templates (per chapter)
-- ─────────────────────────────────────────────
CREATE TABLE exercise_templates (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  chapter_id      UUID NOT NULL REFERENCES chapters(id) ON DELETE CASCADE,
  prompt          TEXT NOT NULL,                           -- "Write about a time when..."
  exercise_type   TEXT DEFAULT 'reflection',               -- 'reflection', 'action', 'tracking'
  sort_order      INTEGER DEFAULT 0,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX exercises_chapter_idx ON exercise_templates(chapter_id);

-- ─────────────────────────────────────────────
-- Reading Sessions
-- ─────────────────────────────────────────────
CREATE TABLE reading_sessions (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id         UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  book_id         UUID NOT NULL REFERENCES books(id),
  chapter_id      UUID NOT NULL REFERENCES chapters(id),
  started_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  ended_at        TIMESTAMPTZ,
  duration_seconds INTEGER,
  completion_pct  REAL DEFAULT 0,                          -- 0.0 to 1.0
  highlights      JSONB DEFAULT '[]',                      -- [{text, position, created_at}]
  ai_interaction_count INTEGER DEFAULT 0,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX rs_user_idx ON reading_sessions(user_id);
CREATE INDEX rs_book_idx ON reading_sessions(book_id);
CREATE INDEX rs_chapter_idx ON reading_sessions(chapter_id);

-- ─────────────────────────────────────────────
-- Emotional Snapshots
-- ─────────────────────────────────────────────
CREATE TABLE emotional_snapshots (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id         UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  book_id         UUID REFERENCES books(id),
  chapter_id      UUID REFERENCES chapters(id),
  session_id      UUID REFERENCES reading_sessions(id),
  snapshot_type   TEXT NOT NULL CHECK (snapshot_type IN (
                    'SESSION_PRE', 'SESSION_POST',
                    'INTAKE', 'COMPLETION', 'FOLLOWUP_30D'
                  )),
  scores          JSONB NOT NULL,                          -- {anxiety: 3, clarity: 4, ...}
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX es_user_idx ON emotional_snapshots(user_id);
CREATE INDEX es_book_idx ON emotional_snapshots(book_id);
CREATE INDEX es_type_idx ON emotional_snapshots(snapshot_type);

-- ─────────────────────────────────────────────
-- Exercise Responses
-- ─────────────────────────────────────────────
CREATE TABLE exercise_responses (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id         UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  exercise_id     UUID NOT NULL REFERENCES exercise_templates(id),
  chapter_id      UUID NOT NULL REFERENCES chapters(id),
  book_id         UUID NOT NULL REFERENCES books(id),
  response_text   TEXT,
  time_spent_seconds INTEGER,
  ai_followup     TEXT,                                    -- AI-generated response to their answer
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX er_user_idx ON exercise_responses(user_id);
CREATE INDEX er_book_idx ON exercise_responses(book_id);

-- ─────────────────────────────────────────────
-- Check-ins (proactive push notification responses)
-- ─────────────────────────────────────────────
CREATE TABLE check_ins (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id         UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  book_id         UUID REFERENCES books(id),
  check_in_type   TEXT NOT NULL CHECK (check_in_type IN (
                    'DAILY', 'EXERCISE_FOLLOWUP', 'EMOTIONAL_DIP',
                    'MILESTONE', 'POST_COMPLETION_30D'
                  )),
  prompt          TEXT,                                    -- what we asked
  response        TEXT,                                    -- what they said
  emotional_score JSONB,                                   -- optional quick pulse
  sent_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  responded_at    TIMESTAMPTZ,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX ci_user_idx ON check_ins(user_id);

-- ─────────────────────────────────────────────
-- Outcomes (per user per book — computed on completion)
-- ─────────────────────────────────────────────
CREATE TABLE outcomes (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id         UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  book_id         UUID NOT NULL REFERENCES books(id),
  completed_at    TIMESTAMPTZ,
  total_sessions  INTEGER,
  total_duration_minutes INTEGER,
  chapters_completed INTEGER,
  exercises_completed INTEGER,
  exercises_total INTEGER,
  emotional_before JSONB,                                  -- intake snapshot scores
  emotional_after  JSONB,                                  -- completion snapshot scores
  emotional_30day  JSONB,                                  -- 30-day followup scores (nullable until followup)
  emotional_improvement REAL,                              -- computed: avg(after) - avg(before), normalized 0-100
  transformation_sustained BOOLEAN,                        -- true if 30-day scores >= completion scores
  most_impactful_chapters INTEGER[],                       -- chapters with highest emotional lift
  most_impactful_exercises UUID[],                         -- exercises user rated as most helpful
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE(user_id, book_id)
);

CREATE INDEX outcomes_book_idx ON outcomes(book_id);
CREATE INDEX outcomes_user_idx ON outcomes(user_id);

-- ─────────────────────────────────────────────
-- Author Accounts
-- ─────────────────────────────────────────────
CREATE TABLE author_accounts (
  id              UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  author_name     TEXT NOT NULL,
  bio             TEXT,
  website_url     TEXT,
  plan            TEXT DEFAULT 'free' CHECK (plan IN ('free', 'pro', 'enterprise')),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE author_book_access (
  author_id       UUID NOT NULL REFERENCES author_accounts(id) ON DELETE CASCADE,
  book_id         UUID NOT NULL REFERENCES books(id) ON DELETE CASCADE,
  role            TEXT DEFAULT 'owner' CHECK (role IN ('owner', 'collaborator', 'viewer')),
  granted_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (author_id, book_id)
);

-- ─────────────────────────────────────────────
-- Recommendation Engine
-- ─────────────────────────────────────────────
CREATE TABLE user_embeddings (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id         UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  embedding       vector(768),                             -- dimension matches embedding model
  context         TEXT,                                    -- what was embedded (for debugging)
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE(user_id)
);

CREATE TABLE book_outcome_embeddings (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  book_id         UUID NOT NULL REFERENCES books(id) ON DELETE CASCADE,
  embedding       vector(768),
  reader_count    INTEGER DEFAULT 0,                       -- how many outcomes this is based on
  avg_emotional_improvement REAL,
  completion_rate REAL,
  top_life_areas  TEXT[],                                  -- areas where this book performs best
  context         TEXT,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE(book_id)
);

CREATE TABLE recommendation_events (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id         UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  book_id         UUID NOT NULL REFERENCES books(id),
  match_score     REAL,
  action          TEXT CHECK (action IN ('shown', 'clicked', 'started', 'purchased')),
  affiliate_url   TEXT,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX re_user_idx ON recommendation_events(user_id);

-- ─────────────────────────────────────────────
-- AI Coaching Conversations (ephemeral — 90 day TTL)
-- ─────────────────────────────────────────────
CREATE TABLE coaching_messages (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id         UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  book_id         UUID REFERENCES books(id),
  chapter_id      UUID REFERENCES chapters(id),
  role            TEXT NOT NULL CHECK (role IN ('user', 'assistant')),
  content         TEXT NOT NULL,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX cm_user_idx ON coaching_messages(user_id);
-- TTL cleanup via Supabase Edge Function cron:
-- DELETE FROM coaching_messages WHERE created_at < NOW() - INTERVAL '90 days';

-- ─────────────────────────────────────────────
-- Row Level Security
-- ─────────────────────────────────────────────
ALTER TABLE user_profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE intake_profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE reading_sessions ENABLE ROW LEVEL SECURITY;
ALTER TABLE emotional_snapshots ENABLE ROW LEVEL SECURITY;
ALTER TABLE exercise_responses ENABLE ROW LEVEL SECURITY;
ALTER TABLE check_ins ENABLE ROW LEVEL SECURITY;
ALTER TABLE outcomes ENABLE ROW LEVEL SECURITY;
ALTER TABLE coaching_messages ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_embeddings ENABLE ROW LEVEL SECURITY;
ALTER TABLE recommendation_events ENABLE ROW LEVEL SECURITY;
ALTER TABLE author_accounts ENABLE ROW LEVEL SECURITY;
ALTER TABLE author_book_access ENABLE ROW LEVEL SECURITY;

-- Users see only their own data
CREATE POLICY "own_data" ON user_profiles FOR ALL USING (auth.uid() = id);
CREATE POLICY "own_data" ON intake_profiles FOR ALL USING (auth.uid() = user_id);
CREATE POLICY "own_data" ON reading_sessions FOR ALL USING (auth.uid() = user_id);
CREATE POLICY "own_data" ON emotional_snapshots FOR ALL USING (auth.uid() = user_id);
CREATE POLICY "own_data" ON exercise_responses FOR ALL USING (auth.uid() = user_id);
CREATE POLICY "own_data" ON check_ins FOR ALL USING (auth.uid() = user_id);
CREATE POLICY "own_data" ON outcomes FOR ALL USING (auth.uid() = user_id);
CREATE POLICY "own_data" ON coaching_messages FOR ALL USING (auth.uid() = user_id);
CREATE POLICY "own_data" ON user_embeddings FOR ALL USING (auth.uid() = user_id);
CREATE POLICY "own_data" ON recommendation_events FOR ALL USING (auth.uid() = user_id);
CREATE POLICY "own_data" ON author_accounts FOR ALL USING (auth.uid() = id);
CREATE POLICY "own_data" ON author_book_access FOR ALL USING (auth.uid() = author_id);

-- Books and chapters are publicly readable
ALTER TABLE books ENABLE ROW LEVEL SECURITY;
CREATE POLICY "public_read" ON books FOR SELECT USING (is_published = true);
ALTER TABLE chapters ENABLE ROW LEVEL SECURITY;
CREATE POLICY "public_read" ON chapters FOR SELECT USING (
  book_id IN (SELECT id FROM books WHERE is_published = true)
);
ALTER TABLE exercise_templates ENABLE ROW LEVEL SECURITY;
CREATE POLICY "public_read" ON exercise_templates FOR SELECT USING (
  chapter_id IN (SELECT id FROM chapters WHERE book_id IN (SELECT id FROM books WHERE is_published = true))
);

-- ─────────────────────────────────────────────
-- Materialized Views (Author Dashboard)
-- ─────────────────────────────────────────────
CREATE MATERIALIZED VIEW mv_chapter_engagement AS
SELECT 
  c.id as chapter_id,
  c.book_id,
  c.chapter_number,
  c.title,
  COUNT(DISTINCT rs.user_id) as reader_count,
  CASE WHEN COUNT(DISTINCT rs.user_id) >= 10 
    THEN ROUND(AVG(rs.completion_pct)::numeric, 2) ELSE NULL END as avg_completion,
  CASE WHEN COUNT(DISTINCT rs.user_id) >= 10 
    THEN ROUND(AVG(rs.duration_seconds)::numeric, 0) ELSE NULL END as avg_duration_seconds,
  CASE WHEN COUNT(DISTINCT rs.user_id) >= 10 
    THEN ROUND(AVG(
      (es_post.scores->>'clarity')::numeric - (es_pre.scores->>'clarity')::numeric
    ), 2) ELSE NULL END as avg_clarity_shift
FROM chapters c
LEFT JOIN reading_sessions rs ON rs.chapter_id = c.id
LEFT JOIN emotional_snapshots es_pre ON es_pre.session_id = rs.id AND es_pre.snapshot_type = 'SESSION_PRE'
LEFT JOIN emotional_snapshots es_post ON es_post.session_id = rs.id AND es_post.snapshot_type = 'SESSION_POST'
GROUP BY c.id, c.book_id, c.chapter_number, c.title;

CREATE MATERIALIZED VIEW mv_book_outcomes AS
SELECT
  book_id,
  COUNT(*) as total_outcomes,
  ROUND(AVG(emotional_improvement)::numeric, 1) as avg_emotional_improvement,
  ROUND(AVG(exercises_completed::numeric / NULLIF(exercises_total, 0))::numeric, 2) as avg_exercise_rate,
  ROUND(AVG(chapters_completed::numeric / NULLIF((SELECT total_chapters FROM books WHERE id = outcomes.book_id), 0))::numeric, 2) as avg_completion_rate,
  COUNT(*) FILTER (WHERE transformation_sustained = true) as sustained_count
FROM outcomes
GROUP BY book_id;

-- Auto-refresh triggers
-- ─────────────────────────────────────────────
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN NEW.updated_at = NOW(); RETURN NEW; END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_user_profiles BEFORE UPDATE ON user_profiles FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER trg_intake BEFORE UPDATE ON intake_profiles FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER trg_outcomes BEFORE UPDATE ON outcomes FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER trg_exercise_responses BEFORE UPDATE ON exercise_responses FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER trg_author_accounts BEFORE UPDATE ON author_accounts FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER trg_user_embeddings BEFORE UPDATE ON user_embeddings FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER trg_book_outcome_embeddings BEFORE UPDATE ON book_outcome_embeddings FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

---

## Cost Estimates (MVP)

| Service | Monthly Cost |
|---|---|
| Supabase Pro | $25 |
| Vercel Pro | $20 |
| Gemini API (coaching) | ~$5-15 (Flash is cheap) |
| Claude API (empathy moments) | ~$10-20 |
| Expo EAS builds | $0 (free tier) |
| PostHog | $0 (free tier < 1M events) |
| **Total** | **~$60-80/mo** |

Scales to 10,000 readers without changing anything. After that, Supabase Team plan ($599/mo) and dedicated Gemini/Claude tiers.

---

## Next Steps

1. Bruce reviews and approves this spec
2. Selina sets up Expo project + Supabase schema
3. Start Week 1-2 sprint: auth + intake + book library
4. Steve Rogers delivers app visual identity + design tokens
5. First book content loaded (reuse existing Loop Reader Carnegie/Voss chapters)
