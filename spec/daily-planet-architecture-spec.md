# The Daily Planet — Architecture Spec

**Product:** Your own agentic newspaper delivering news and stories from people and topics you care about.
**Stack:** Glove (agentic runtime) × Station (background jobs infrastructure)
**By:** dterminal
**Version:** v0.1 — March 2026

---

## Overview

The Daily Planet is a personalized, multi-agent newsroom that runs autonomously on a schedule. Users define topics of interest, people to follow, and agentic writers with distinct voices. The system indexes sources, researches stories, verifies claims, edits and ranks articles, then publishes formatted editions to web and Telegram — with optional audio briefings.

The product is built on two dterminal primitives:

- **Glove** handles the _who_ and _what_ — agent personas, tool use, user-facing screens, and voice.
- **Station** handles the _when_ and _how_ — scheduling, DAG orchestration, retries, deduplication, and observability.

---

## System Layers

The architecture has four distinct layers:

1. **Indexing Pipeline** — Fetching, deduplicating, and storing raw source material.
2. **Editorial Workflow** — The multi-agent DAG that turns raw material into publishable articles.
3. **Delivery & Presentation** — Web publications, Telegram delivery, audio briefings, and the interactive assistant.
4. **User Dashboard** — Configuration surface for topics, writers, delivery preferences, and commissioned reports.

---

## 1. Indexing Pipeline

### Purpose

Continuously collect raw source material from the web, Twitter/X accounts, RSS feeds, and any user-defined sources. Store it in a normalized format for the editorial agents to work from.

### Data Sources

- **Web search** — Periodic keyword searches based on user-defined topics and their associated keywords.
- **Twitter/X accounts** — Poll posts from followed accounts at a configurable interval.
- **RSS feeds** — Subscribe to feeds from publications, blogs, and news outlets.
- **Custom URLs** — User-provided links to specific pages, newsletters, or feeds.

### Indexing Flow

```
[Source Polling] → [Fetch Content] → [Deduplication Check] → [Normalize & Store] → [Index by Topic]
```

#### Source Polling (Station: scheduled job)

A recurring Station job that runs on the user's configured frequency (hourly, every 4 hours, etc.). For each configured source type, it dispatches fetch tasks.

- One job per source type (web search, Twitter, RSS) to allow independent scheduling and retry logic.
- Each job fans out into individual fetch tasks per source/keyword.

#### Fetch Content

For each source URL or search result:

- Retrieve the full page content (not just snippets).
- Extract the article body, title, author, publish date, and any structured data (numbers, quotes, entities).
- Store raw content alongside metadata.

#### Deduplication

Before storing, check against existing indexed stories:

- Content-hash based dedup (exact match).
- Semantic similarity check against recent stories in the same topic (prevent covering the same event from multiple angles as separate stories — flag them as related instead).
- Maintain a `story_cluster` concept — multiple sources about the same event get grouped.

#### Normalize & Store

Each indexed item gets stored with:

- `id` — Unique identifier.
- `source_url` — Original URL.
- `source_type` — web_search | twitter | rss | custom.
- `source_name` — Human-readable source name (e.g., "Reuters", "@karpathy").
- `title` — Extracted or generated title.
- `body` — Full extracted text.
- `entities` — People, companies, technologies mentioned.
- `publish_date` — Original publication date.
- `indexed_at` — When we fetched it.
- `topic_ids` — Which user topics this maps to (can be multiple).
- `story_cluster_id` — Which cluster of related stories this belongs to.
- `relevance_score` — How relevant to the user's interests (based on keyword match, topic priority, source trustworthiness).
- `used_in_edition` — Which edition(s) this source was used in (prevents re-use).

#### Breaking News Detection

A separate Station job runs at a higher frequency (every 5–15 minutes) specifically to detect breaking news:

- Monitors high-priority sources only (based on user's topic priority settings).
- Looks for sudden spikes in coverage of a topic across multiple sources.
- If a breaking story is detected, it triggers the editorial workflow immediately (bypasses the normal schedule) and sends a notification if the user has breaking news alerts enabled.

### Station Mapping

| Job | Schedule | Fan-out | Retry |
|-----|----------|---------|-------|
| `index:web-search` | User-configured (default: every 4h) | Per keyword per topic | 3 retries, exponential backoff |
| `index:twitter` | Every 30 min | Per followed account | 3 retries |
| `index:rss` | Every 1h | Per feed | 3 retries |
| `index:breaking-detect` | Every 10 min | Per priority-1 topic | 2 retries, fast |

---

## 2. Editorial Workflow

### Purpose

Transform indexed source material into polished, verified articles organized into a publishable edition. This is the core DAG that exercises Station's orchestration (station-broadcast) and Glove's agent runtime.

### Agents

There are two categories of agents:

#### System Agents (not user-configurable)

- **Lead Researcher** — Takes indexed sources for a topic, investigates deeper. Follows links, finds primary sources, pulls supporting data. Spins up sub-research tasks for specific claims or entities. Outputs a research brief for each story cluster.
- **Lead Editor** — Receives all research briefs and writer drafts. Verifies claims against sources. Decides which stories make the edition, where they rank (based on user topic priority + story importance), and which template each story gets (headline, short report, full-page piece). Handles dedup at the editorial level — if two writers produce overlapping stories, the editor merges or kills one.

#### Writer Agents (user-configurable)

Each writer has:

- `id` — Unique identifier.
- `name` — Display name (e.g., "Mara Kessler").
- `initials` — For avatar display.
- `role` — E.g., "Senior Correspondent", "Markets Desk", "Africa Bureau".
- `beat` — Topics they cover (maps to user topic IDs).
- `voice_description` — Free-text description of how they write. This becomes the system prompt personality for the Glove agent.
- `style_tags` — Enumerated style descriptors (e.g., "Analytical", "Evidence-First", "Narrative", "Concise").
- `preferences` — Structured preferences: avg article length, sources-per-piece target, use of quotes, use of data visualizations, etc.

### Editorial DAG

This is the main Station broadcast DAG that runs for each edition:

```
[Gather Indexed Sources]
        │
        ▼
[Lead Researcher] ──fan-out──→ [Research Brief per Story Cluster]
        │                              │
        │                              ▼
        │                    [Assign to Writers] (Editor decides)
        │                              │
        │                    ┌─────────┼─────────┐
        │                    ▼         ▼         ▼
        │              [Writer A] [Writer B] [Writer C]  ← fan-out, parallel
        │                    │         │         │
        │                    └─────────┼─────────┘
        │                              │
        │                              ▼
        │                    [Drafts Submitted]
        │                              │
        ▼                              ▼
[Lead Editor] ◄──────────── [All Drafts + Research Briefs]
        │
        ├── Verify claims against sources
        ├── Rank stories (user priority × story importance)
        ├── Assign templates (headline, short, full-page)
        ├── Kill duplicates
        ├── Write editor's notes / analysis blocks
        │
        ▼
[Edition Manifest]
        │
        ├──→ [Generate Web Publication]
        ├──→ [Generate Audio Briefing] (conditional: if user has audio enabled)
        └──→ [Send Notifications] (Telegram, email, etc.)
```

### Edition Manifest

The editor's output is a structured manifest that defines the complete edition:

- `edition_id` — Unique identifier.
- `edition_type` — morning | afternoon | evening | breaking | hourly.
- `published_at` — Timestamp.
- `articles` — Ordered list of articles, each containing:
  - `article_id`
  - `headline`
  - `deck` — Subtitle/summary.
  - `body` — Full article content (structured: paragraphs, quotes, data points, subheadings).
  - `template` — Which presentation template to use: `broadsheet_lead` | `broadsheet_secondary` | `dispatch_item` | `editorial_full_page` | `short_report`.
  - `writer_id` — Which writer produced it.
  - `topic_ids` — Which topics it covers.
  - `sources` — List of source references with verification status.
  - `editorial_notes` — Editor's annotations (editor's note, analysis, context).
  - `data_visualizations` — Any inline charts, data rows, or figures (structured data the template renderer will turn into visual elements).
  - `priority_rank` — Position in the edition.
  - `read_time_minutes` — Estimated read time.
- `breaking` — Boolean, whether this is a breaking news edition.
- `audio_script` — If audio is enabled, the script for the audio briefing.
- `summary` — Short text summary of the edition for Telegram/notification delivery.

### Research Brief Structure

What the Lead Researcher outputs per story cluster:

- `story_cluster_id`
- `topic_ids`
- `headline_suggestion` — Proposed headline.
- `summary` — 2–3 sentence summary of the story.
- `key_claims` — List of specific claims with source references.
- `key_quotes` — Notable quotes from sources.
- `key_data` — Numbers, statistics, percentages worth highlighting.
- `entities` — People, companies, technologies involved.
- `source_list` — All sources consulted, with reliability notes.
- `suggested_priority` — Researcher's recommendation on importance.
- `suggested_template` — Whether this warrants a full-page piece, short report, or headline.
- `follow_up_angles` — Suggestions for deeper investigation.

### Writer Draft Structure

What each writer outputs:

- `article_id`
- `headline`
- `deck`
- `body` — Structured content with:
  - Paragraphs (with optional drop-cap flag on first).
  - Block quotes (with attribution).
  - Subheadings (h3 level).
  - Inline data references (that the editor/renderer can turn into data-row visualizations).
  - Callouts / analysis blocks.
- `sources_cited` — Which sources from the research brief they used.
- `word_count`
- `suggested_data_visualizations` — If the writer thinks a chart or data row would help.

### Deduplication Rules

Dedup happens at two levels:

1. **Indexing level** — Content-hash and semantic similarity prevent storing the same raw source twice.
2. **Editorial level** — The editor checks if two story clusters are covering the same underlying event. If so, they merge the research briefs and assign to a single writer. The editor also checks against previous editions — a story that was covered in edition N should not appear again in edition N+1 unless there's a material update.

### Station Mapping

| Job | Type | Notes |
|-----|------|-------|
| `editorial:gather-sources` | Step 1 of DAG | Pulls all unprocessed indexed items since last edition |
| `editorial:research` | Fan-out | One task per story cluster, parallel |
| `editorial:assign-writers` | Conditional | Editor reviews briefs, assigns to writers based on beat |
| `editorial:write` | Fan-out | One task per assignment, parallel |
| `editorial:edit` | Fan-in | Waits for all drafts, then editor processes |
| `editorial:publish` | Fan-out | Parallel: web generation, audio generation, notifications |

### Glove Mapping

| Agent | Glove Role | Tools Available |
|-------|-----------|-----------------|
| Lead Researcher | System agent, no user screen | Web search, web fetch, source database read |
| Lead Editor | System agent, no user screen | Source database read, edition database write, dedup checker |
| Writer (each) | User-defined persona via voice_description | Research brief reader, source database read |
| Audio Narrator | System agent | Text-to-speech (ElevenLabs), edition manifest reader |

---

## 3. Delivery & Presentation

### Purpose

Render the edition manifest into user-facing outputs: web publications, Telegram messages, audio briefings, and the interactive assistant.

### Web Publication

The primary output. Each edition becomes a standalone web page with a unique URL.

#### Templates

Based on the design explorations, the web publication uses these template modes:

- **Broadsheet** — The main edition view. Front-page layout with lead story, sidebar trending/ranked topics, breaking ticker, secondary stories grid. Used for the default "open your newspaper" experience.
- **Dispatch** — Dark, timeline-based format. Used for breaking news editions and hourly digests. Dense, priority-tagged, fast to scan.
- **Editorial** — Magazine-style cover with TOC bar. Used when the edition has a clear lead story that deserves the spotlight. Cover page → TOC → individual articles.
- **Full-Page Piece** — Individual article deep-dive. Cinematic hero, three-column prose layout (section markers | body | annotations), inline data visualizations, verified source cards, writer bio. Every article can be opened in this view.

#### Template Selection Logic

The editor assigns templates per-article, but the overall edition template is determined by:

- **Breaking edition** → Dispatch template.
- **Scheduled edition with a clear lead story** → Editorial template (cover + TOC).
- **Scheduled edition, balanced stories** → Broadsheet template.
- **Individual article click-through** → Always Full-Page Piece.

#### Rendering

The web publication renderer takes the edition manifest and produces static HTML:

- Apply Signal brand system (fonts, colors, spacing, motion, grain).
- Populate template slots with article content.
- Generate inline data visualizations from structured data in the manifest (bar charts, data rows, stat strips).
- Render source cards with verification status.
- Embed audio player if audio briefing is available.
- Include the interactive assistant (see below).

### Telegram Delivery

A Telegram bot that:

- Sends a message when a new edition is published, containing:
  - Edition type and time.
  - Headline summary (2–3 sentences).
  - List of article headlines with topic tags.
  - Link to open the full web publication.
- Sends breaking news alerts immediately with:
  - Breaking label.
  - Headline and one-paragraph summary.
  - Link to the dispatch view.
- Supports commands:
  - `/latest` — Link to most recent edition.
  - `/breaking` — Toggle breaking news notifications.
  - `/schedule` — Show current publication schedule.
  - `/report [topic]` — Commission a report via Telegram.

### Audio Briefing

Generated after the editor finalizes each edition:

- The editor produces an `audio_script` in the edition manifest — a narration-optimized version of the edition's key stories.
- A Station job sends this script to ElevenLabs for text-to-speech rendering (via glove-11 integration).
- The resulting audio file is attached to the edition and playable from the web publication and Telegram.
- Duration target: 3–5 minutes for standard editions, 1–2 minutes for breaking.

### Interactive Assistant

A Glove agent embedded in the web publication as a slide-out panel. Capabilities:

- **Ask questions** about any story in the current edition or across past editions.
- **Get source links** — "Where did this claim come from?" returns the specific source with a link.
- **Pull up related stories** — "What else have we covered on this topic?" searches the edition archive.
- **Play audio** — "Read me the summary" triggers the audio briefing or a per-article narration.
- **Navigate** — "Show me the full report on the EU regulation" opens the full-page piece view.

The assistant has read access to:

- Current edition manifest.
- All indexed sources for the current edition.
- Edition archive (past editions and their articles).
- User's topic and writer configuration (to personalize responses).

### Station Mapping

| Job | Trigger | Notes |
|-----|---------|-------|
| `delivery:render-web` | After editorial:publish | Generates static HTML from manifest + templates |
| `delivery:audio-briefing` | After editorial:publish (conditional) | Sends script to ElevenLabs, stores audio file |
| `delivery:telegram-notify` | After render-web completes | Sends message with link |
| `delivery:email-digest` | After render-web completes (conditional) | Sends email summary |
| `delivery:breaking-alert` | Triggered by breaking detection | Immediate, bypasses normal schedule |

---

## 4. User Dashboard

### Purpose

The configuration surface where users set up their newsroom. This is separate from the publication output — it's the control room.

### Sections

#### Topics & Sources

**Topics tab:**

- List of user-defined topics, displayed as ranked cards.
- Each topic has:
  - `id`
  - `name` — Display name (e.g., "AI & Machine Learning").
  - `keywords` — Search terms and phrases used by the indexing pipeline.
  - `priority` — Rank order (determines story placement in editions and breaking news filtering).
  - `sources_indexed` — Count of raw sources collected for this topic.
  - `articles_published` — Count of articles published on this topic.
- Users can: add, edit, reorder (drag), delete topics.
- Reordering changes priority, which affects: where stories appear in the edition, which breaking news the user is notified about, and how the editor weighs stories.

**People & Accounts tab:**

- List of Twitter/X accounts, RSS feeds, and specific people to track.
- Each source has:
  - `id`
  - `name` — Display name.
  - `handle` — @handle or URL.
  - `type` — twitter | rss | website | person.
  - `topic_ids` — Which topics this source maps to.
- Posts/content from these sources are fed into the indexing pipeline and associated with relevant topics.

#### Writers

- Cards for each user-created writer agent showing: name, role, beat, voice description, style tags, and stats (articles written, avg read time, sources per piece).
- **Create Writer** flow:
  - Name and initials.
  - Role title (free text).
  - Beat — select which topics they cover.
  - Voice description — free-text description of writing style. This is the core personality prompt for the Glove agent.
  - Style tags — select from a set (Analytical, Narrative, Concise, Data-Driven, Evidence-First, People-Focused, Understated, Fast-Paced, etc.) or create custom.
  - Preferences — target article length, sources per piece, use of quotes, data visualization tendency.
- **Edit Writer** — same fields, editable at any time.
- **System Agents** section (non-editable) shows the Lead Researcher and Lead Editor with their current activity status.

#### Delivery Settings

- **Channels** — Toggle on/off: Web Publication, Telegram Bot, Audio Briefing, Email Digest.
- **Notifications:**
  - Breaking News — on/off, with scope filter (all topics, priority 1–2 only, priority 1 only).
  - Edition Ready — notify when a new edition is published.
  - Report Complete — notify when a commissioned report finishes.
- **Publication Schedule:**
  - Frequency selector: hourly, every 4 hours, 2× daily, daily.
  - Day-of-week grid with delivery times per day.
  - Weekend schedule can differ from weekday.

#### Commissioned Reports

- List of user-requested deep-dive research pieces.
- Each report has:
  - `id`
  - `title` — What the user wants investigated.
  - `brief` — User's description of what they want to know.
  - `assigned_writer_id` — Which writer agent is producing it.
  - `status` — requested | researching | writing | in_review | complete.
  - `sources_collected` — Count.
  - `requested_at` — Timestamp.
  - `completed_at` — Timestamp (if complete).
  - `edition_id` — Which edition it was published in (if complete).
- **Commission Report** flow:
  - Title / topic.
  - Brief / what specifically to investigate.
  - Assign to a writer (or let the editor auto-assign based on beat).
  - Estimated scope (quick report vs. deep-dive).
- Commissioned reports bypass the normal editorial pipeline — they get their own research → write → edit cycle and can be published as standalone pieces or included in the next scheduled edition.

#### Activity Feed

Visible at the bottom of every dashboard tab. Shows a live log of what the newsroom agents are doing:

- Lead Researcher scanning sources.
- Writers drafting or idle.
- Editor verifying or publishing.
- Audio briefing generated.
- Breaking news detected.

This maps to Station's observability layer — each log entry corresponds to a Station job execution or state change.

---

## Data Model Summary

### Core Entities

| Entity | Description |
|--------|-------------|
| `User` | The person configuring and consuming the newspaper. |
| `Topic` | A subject area with keywords and priority ranking. |
| `Source` | A Twitter account, RSS feed, website, or person being tracked. |
| `Writer` | A user-defined agentic writer with a persona and style. |
| `IndexedItem` | A raw piece of source material fetched from the web. |
| `StoryCluster` | A group of related indexed items about the same event/topic. |
| `ResearchBrief` | The Lead Researcher's output per story cluster. |
| `Article` | A finished piece of writing produced by a writer and reviewed by the editor. |
| `Edition` | A published collection of articles with metadata and template assignments. |
| `CommissionedReport` | A user-requested deep-dive report. |
| `AudioBriefing` | A generated audio file for an edition. |
| `Notification` | A delivery event (Telegram message, email, etc.). |

### Key Relationships

```
User
├── has many Topics (ordered by priority)
├── has many Sources (each linked to Topics)
├── has many Writers
├── has many CommissionedReports
└── has many Editions
    ├── has many Articles
    │   ├── belongs to Writer
    │   ├── references Sources (verified)
    │   └── belongs to StoryCluster
    ├── has one AudioBriefing (optional)
    └── has many Notifications
```

---

## Station DAG Overview

The full system is composed of two Station DAGs plus standalone scheduled jobs:

### DAG 1: Indexing

```
index:web-search ──┐
index:twitter ─────┤──→ index:normalize-store ──→ index:dedup ──→ index:topic-assign
index:rss ─────────┘
```

Runs on the user's source polling schedule. Each source type is independent.

### DAG 2: Editorial + Delivery

```
editorial:gather ──→ editorial:research (fan-out)
                          │
                          ▼
                   editorial:assign (editor)
                          │
                          ▼
                   editorial:write (fan-out)
                          │
                          ▼
                   editorial:edit (fan-in)
                          │
                          ▼
                   editorial:publish
                     ┌────┼────┐
                     ▼    ▼    ▼
              render  audio  notify
```

Runs on the user's publication schedule. Breaking detection can trigger this DAG immediately.

### Standalone Jobs

| Job | Schedule | Purpose |
|-----|----------|---------|
| `index:breaking-detect` | Every 10 min | Scan priority-1 topics for breaking news |
| `reports:process` | On commission | Run research → write → edit for a commissioned report |
| `cleanup:dedup-sweep` | Daily | Clean up old indexed items and story clusters |

---

## Glove Agent Definitions

### Lead Researcher

- **Type:** System agent (not user-facing).
- **Persona:** No personality — purely functional.
- **Tools:** Web search, web fetch, indexed source database read, story cluster creation.
- **Input:** Unprocessed indexed items grouped by topic.
- **Output:** Research briefs per story cluster.

### Lead Editor

- **Type:** System agent (not user-facing).
- **Persona:** No personality — purely functional, but applies the user's topic priorities as editorial judgment.
- **Tools:** Research brief reader, article draft reader, source verification, edition manifest writer, dedup checker, template assigner.
- **Input:** All research briefs + all writer drafts.
- **Output:** Edition manifest.

### Writers (User-Defined)

- **Type:** User-configured agents.
- **Persona:** Defined by `voice_description` and `style_tags`. Each writer is a distinct Glove agent with a unique system prompt built from these fields.
- **Tools:** Research brief reader, source database read (for additional context).
- **Input:** Research brief + writer assignment from editor.
- **Output:** Article draft.

### Audio Narrator

- **Type:** System agent (not user-facing).
- **Persona:** Neutral, clear narration voice.
- **Tools:** Edition manifest reader, ElevenLabs TTS (via glove-11).
- **Input:** Edition manifest with `audio_script`.
- **Output:** Audio file (MP3/WAV).

### Interactive Assistant

- **Type:** User-facing Glove agent with a screen (slide-out panel in web publication).
- **Persona:** Helpful, concise, source-aware. Speaks in the Signal voice — quiet confidence, utilitarian precision.
- **Tools:** Edition archive search, source lookup, audio playback trigger, navigation commands.
- **Input:** User questions in natural language.
- **Output:** Answers with source links, article references, and navigation actions.

---

## Resolved Decisions

### 1. Storage Layer

**SQLite.** This is a single-user, local-first application. All data — indexed items, editions, articles, user config, engagement tracking — lives in a local SQLite database on the user's machine. No cloud database, no managed infrastructure.

Considerations:

- SQLite handles the read/write patterns well: bulk inserts during indexing, reads during editorial and rendering, append-only for the edition archive.
- Full-text search via SQLite FTS5 for the searchable archive and the interactive assistant's source lookup.
- Semantic similarity for deduplication can use an embedded vector store (sqlite-vss or similar) or a simple cosine similarity computation against stored embeddings.
- The database file lives alongside the Next.js app — portable, backupable, zero-ops.

### 2. Deployment Model

**Single-user, self-hosted.** You clone the repo, configure your API keys, and run it locally. There is no multi-tenant infrastructure, no auth system, no shared hosting. It's your machine, your newsroom.

This means:

- No user authentication (single user implied).
- No rate limiting or usage quotas (the user manages their own API costs).
- Configuration stored locally in SQLite, not a remote service.
- The Station dashboard (station-kit) runs locally and provides observability into all jobs.

### 3. Web Publication Rendering

**Next.js.** The entire application — dashboard, publication rendering, and the interactive assistant — is a single Next.js app running locally.

- Editions are rendered as Next.js pages using the edition manifest + templates (Broadsheet, Dispatch, Editorial, Full-Page Piece).
- No static HTML generation needed — Next.js handles SSR/ISR for each edition.
- The interactive assistant runs as a client-side Glove agent embedded in the publication pages.
- Routes: `/dashboard` for config, `/editions/[id]` for publications, `/editions/[id]/articles/[articleId]` for full-page pieces.

### 4. Twitter/X Access

**Twitterapi.io** as the provider. This is a third-party API service that provides:

- Search for relevant profiles and tweets by keyword.
- Poll recent posts from specific accounts.
- No need for official Twitter API access or OAuth app approval.

The user provides their own Twitterapi.io API key during setup. The `index:twitter` Station job uses this API to fetch posts from followed accounts and search results for topic keywords.

### 5. Audio Voice Selection

**User-configurable.** Since the user brings their own ElevenLabs API key, they choose:

- Which ElevenLabs voice to use for narration (voice ID configured in settings).
- Optionally, different voices per writer agent — so Mara Kessler could have a different narration voice than Jules Nakamura.
- Or a single narrator voice for all audio briefings.

This is configured in the dashboard under Delivery Settings → Audio Briefing → Voice Configuration.

### 6. Cost Model

**Bring your own keys.** The user provides API keys for:

- An LLM provider (for all agent inference — researcher, editor, writers, assistant). Model selection is configurable — the user picks which model to use.
- Twitterapi.io (for Twitter indexing).
- ElevenLabs (for audio briefings, optional).

There is no pricing from dterminal — the user pays their API providers directly. The cost per edition depends on the number of stories, depth of research, and whether audio is enabled. The Station dashboard shows job execution metrics that help the user understand their usage.

### 7. Edition Archive

**Forever, fully searchable.** All editions persist in the local SQLite database indefinitely. The archive is:

- Browsable by date, topic, writer, or edition type.
- Full-text searchable across all article content, headlines, and source citations (via SQLite FTS5).
- Accessible from the dashboard (Archive view) and the interactive assistant ("What have we covered on Solana this month?").

A daily cleanup job (`cleanup:dedup-sweep`) removes stale indexed items that were never used in an edition, but editions and articles are never deleted.

### 8. Collaborative Newsrooms

**Not supported.** Single user only for v1. Future consideration.

### 9. Writer Learning — The Insight Agent

**Yes.** There is a new system agent — the **Insight Agent** — that creates a feedback loop between the user's reading behavior and the newsroom's output.

See the full Insight Agent specification below.

### 10. Source Trustworthiness

**Default list + user additions.** The system ships with a curated default list of reputable sources across common categories (major wire services, established publications, known research institutions). Each source has a default trust score.

Users can:

- Add additional sources (websites, people, X accounts) which start at a neutral trust score.
- Override trust scores for any source (including defaults).
- The Lead Editor factors source trust scores into claim verification — a claim backed only by low-trust sources gets flagged for additional verification or a caveat in the article.

Trust score is a simple 1–5 scale: 1 (unverified/unknown), 2 (user-added, not yet validated), 3 (generally reliable), 4 (established and reputable), 5 (primary source / official record).

---

## 5. Insight Agent

### Purpose

A system agent that observes the user's engagement with their newspaper and produces actionable intelligence for the editorial workflow. This creates a feedback loop — the newsroom gets better at serving the user over time without requiring manual configuration changes.

### Engagement Signals Tracked

The Insight Agent passively collects:

- **Read time per article** — How long the user spends on each piece (measured by scroll depth and time-on-page in the Next.js publication).
- **Article opens** — Which articles the user clicks into from the broadsheet/edition view.
- **Full reads vs. bounces** — Did the user reach the bottom of the article or leave early?
- **Audio listens** — Did the user play the audio briefing? How far did they listen?
- **Assistant interactions** — What questions did the user ask the interactive assistant? Which topics came up?
- **Topic dwell time** — Aggregate time spent on articles within each topic over time.
- **Writer preference** — Which writers' articles get the most engagement?
- **Negative signals** — If the user provides negative feedback on an article (e.g., a comment, a thumbs down, or a dismissal), the agent captures this.
- **Search queries** — What the user searches for in the archive (indicates latent interests not yet captured by topic configuration).
- **Commission patterns** — What topics the user commissions reports on (strong interest signal).

### Insight Generation

The Insight Agent runs as a Station job on a configurable schedule (default: daily, after the last edition of the day). It processes the accumulated engagement data and produces an **Insight Report** — a structured document that the editorial agents consume.

#### Insight Report Structure

- `report_date`
- `engagement_summary` — High-level stats: total read time, articles opened, audio listened.
- `topic_insights` — Per topic:
  - `topic_id`
  - `engagement_score` — Computed from read time, opens, and full reads.
  - `trend` — Rising, stable, or declining interest compared to previous period.
  - `recommendation` — E.g., "User spent 3× more time on AI agent articles than blockchain this week. Consider increasing AI coverage priority."
- `writer_insights` — Per writer:
  - `writer_id`
  - `engagement_score`
  - `style_feedback` — E.g., "User consistently reads Mara's long-form pieces to completion but bounces from Jules's shorter market reports. Jules may benefit from slightly more narrative context."
  - `negative_feedback_summary` — If any negative signals were captured, what specifically was the issue.
- `emerging_interests` — Topics or keywords that appear in assistant queries or archive searches but aren't currently configured as topics. E.g., "User asked the assistant about 'WebAssembly' 3 times this week — not a configured topic."
- `suggested_actions` — Concrete recommendations:
  - "Add 'WebAssembly' as a subtopic under Developer Infrastructure."
  - "Increase priority of AI & Machine Learning from #1 to #1 with expanded keywords."
  - "Adjust Jules Nakamura's voice to include more contextual framing before data points."
  - "User has not engaged with East Africa Tech in 2 weeks — consider reducing frequency or prompting for updated interest."

### How the Insight Report Feeds Back

The Insight Report is consumed by two agents:

1. **Lead Editor** — Uses topic engagement scores to adjust story ranking within editions. A topic with rising engagement gets more prominent placement even if its configured priority hasn't changed. A topic with declining engagement might get fewer stories or shorter treatments.

2. **Writers** — Each writer receives a summary of their engagement feedback as additional context in their system prompt for the next edition cycle. This is subtle — not "the user didn't like your article" but rather "recent engagement data suggests the audience responds well to articles that include more contextual framing before presenting data" (for example).

The user can also review the Insight Report in the dashboard and choose to act on the suggested actions (e.g., add a new topic, adjust a writer's voice).

### Glove Definition

- **Type:** System agent (not user-facing).
- **Persona:** No personality — analytical and observational.
- **Tools:** Engagement database read, topic/writer config read, insight report writer.
- **Input:** Accumulated engagement events since last insight generation.
- **Output:** Insight Report (stored in SQLite, consumed by editorial agents).

### Station Mapping

| Job | Schedule | Notes |
|-----|----------|-------|
| `insight:collect` | Continuous (event-driven) | Writes engagement events to SQLite as they happen |
| `insight:generate` | Daily (after last edition) | Processes events, produces Insight Report |
| `insight:apply` | Before each editorial DAG run | Injects latest insight context into editor and writer prompts |

---

## Updated Data Model

With the resolved decisions, the data model adds:

| Entity | Description |
|--------|-------------|
| `EngagementEvent` | A single engagement signal (article open, read time, audio listen, search query, etc.) |
| `InsightReport` | The Insight Agent's daily output — topic insights, writer insights, emerging interests, suggested actions. |
| `SourceTrustScore` | Per-source trust rating (1–5), with default and user-override values. |
| `UserConfig` | Single-row table storing all user settings: API keys, schedule, notification preferences, etc. |

Updated relationships:

```
User (single, implicit)
├── has many Topics (ordered by priority)
├── has many Sources (each linked to Topics, each with a SourceTrustScore)
├── has many Writers
├── has many CommissionedReports
├── has many EngagementEvents
├── has many InsightReports
├── has one UserConfig
└── has many Editions (persisted forever)
    ├── has many Articles
    │   ├── belongs to Writer
    │   ├── references Sources (verified, with trust scores)
    │   └── belongs to StoryCluster
    ├── has one AudioBriefing (optional)
    └── has many Notifications
```

---

## Updated Glove Agent Definitions

Adding the Insight Agent to the full roster:

| Agent | Type | Persona | Key Tools |
|-------|------|---------|-----------|
| Lead Researcher | System | Functional, no personality | Web search, web fetch, source DB, story clustering |
| Lead Editor | System | Functional, applies user priorities + insight data | Brief reader, draft reader, source verification, manifest writer, dedup, template assignment |
| Writers (N) | User-defined | Unique voice per `voice_description` + `style_tags`, modulated by insight feedback | Research brief reader, source DB |
| Audio Narrator | System | Neutral narration, voice configurable by user | Edition manifest reader, ElevenLabs TTS (glove-11) |
| Interactive Assistant | User-facing (screen) | Helpful, concise, Signal voice | Edition archive search, source lookup, audio playback, navigation |
| Insight Agent | System | Analytical, observational | Engagement DB read, config read, insight report writer |

---

## Updated Station DAG Overview

### DAG 1: Indexing (unchanged)

```
index:web-search ──┐
index:twitter ─────┤──→ index:normalize-store ──→ index:dedup ──→ index:topic-assign
index:rss ─────────┘
```

### DAG 2: Editorial + Delivery (updated with insight injection)

```
insight:apply (inject latest insight context)
       │
       ▼
editorial:gather ──→ editorial:research (fan-out)
                          │
                          ▼
                   editorial:assign (editor, informed by insights)
                          │
                          ▼
                   editorial:write (fan-out, writers receive insight feedback)
                          │
                          ▼
                   editorial:edit (fan-in)
                          │
                          ▼
                   editorial:publish
                     ┌────┼────┐
                     ▼    ▼    ▼
              render  audio  notify
```

### DAG 3: Insight (new)

```
insight:collect (continuous) ──→ insight:generate (daily) ──→ insight:apply (before each editorial run)
```

### Standalone Jobs

| Job | Schedule | Purpose |
|-----|----------|---------|
| `index:breaking-detect` | Every 10 min | Scan priority-1 topics for breaking news |
| `reports:process` | On commission | Research → write → edit for a commissioned report |
| `cleanup:dedup-sweep` | Daily | Remove stale unused indexed items |
| `insight:generate` | Daily (after last edition) | Produce insight report from engagement data |

---

## Tech Stack Summary

| Layer | Technology |
|-------|-----------|
| Runtime | Node.js |
| Framework | Next.js (app router) |
| Database | SQLite (via better-sqlite3 or drizzle-orm + SQLite driver) |
| Full-text search | SQLite FTS5 |
| Vector similarity | sqlite-vss or embedded cosine similarity |
| Agent runtime | Glove |
| Job orchestration | Station (station-signal + station-broadcast + station-kit) |
| LLM inference | User-configured (any provider, user brings API key) |
| Twitter data | Twitterapi.io (user brings API key) |
| Text-to-speech | ElevenLabs via glove-11 (user brings API key, optional) |
| Telegram bot | Telegram Bot API (user creates bot, provides token) |
| Styling | Signal brand system (Instrument Serif, IBM Plex Mono, Space Grotesk, rust/patina/concrete palette) |

---

## Project Structure (Proposed)

```
daily-planet/
├── src/
│   ├── app/                          # Next.js app router
│   │   ├── dashboard/                # Dashboard pages
│   │   │   ├── topics/
│   │   │   ├── writers/
│   │   │   ├── delivery/
│   │   │   ├── reports/
│   │   │   └── insights/
│   │   ├── editions/                 # Publication pages
│   │   │   ├── [editionId]/
│   │   │   │   ├── page.tsx          # Edition view (broadsheet/dispatch/editorial)
│   │   │   │   └── articles/
│   │   │   │       └── [articleId]/
│   │   │   │           └── page.tsx  # Full-page piece view
│   │   │   └── archive/
│   │   └── api/                      # API routes
│   │       ├── indexing/
│   │       ├── editorial/
│   │       ├── delivery/
│   │       └── assistant/
│   ├── agents/                       # Glove agent definitions
│   │   ├── researcher.ts
│   │   ├── editor.ts
│   │   ├── writer.ts                 # Factory that builds writer agents from user config
│   │   ├── narrator.ts
│   │   ├── assistant.ts
│   │   └── insight.ts
│   ├── jobs/                         # Station job definitions (station-signal)
│   │   ├── indexing/
│   │   │   ├── web-search.ts
│   │   │   ├── twitter.ts
│   │   │   ├── rss.ts
│   │   │   ├── normalize.ts
│   │   │   ├── dedup.ts
│   │   │   └── breaking-detect.ts
│   │   ├── editorial/
│   │   │   ├── gather.ts
│   │   │   ├── research.ts
│   │   │   ├── assign.ts
│   │   │   ├── write.ts
│   │   │   ├── edit.ts
│   │   │   └── publish.ts
│   │   ├── delivery/
│   │   │   ├── render.ts
│   │   │   ├── audio.ts
│   │   │   ├── telegram.ts
│   │   │   └── email.ts
│   │   ├── insight/
│   │   │   ├── collect.ts
│   │   │   ├── generate.ts
│   │   │   └── apply.ts
│   │   └── reports/
│   │       └── process.ts
│   ├── workflows/                    # Station broadcast DAGs (station-broadcast)
│   │   ├── indexing-dag.ts
│   │   ├── editorial-dag.ts
│   │   └── insight-dag.ts
│   ├── db/                           # Database layer
│   │   ├── schema.ts                 # Drizzle schema definitions
│   │   ├── migrations/
│   │   └── seed/                     # Default source trust scores, etc.
│   ├── templates/                    # Publication templates (React components)
│   │   ├── broadsheet/
│   │   ├── dispatch/
│   │   ├── editorial/
│   │   └── full-page/
│   ├── components/                   # Shared UI components
│   │   ├── dashboard/
│   │   ├── publication/
│   │   └── assistant/
│   └── lib/                          # Shared utilities
│       ├── config.ts                 # User config / API key management
│       ├── trust-scores.ts           # Default source trust list
│       └── engagement.ts             # Engagement event tracking
├── data/                             # SQLite database file lives here
│   └── planet.db
├── package.json
├── station.config.ts                 # Station configuration
└── README.md
```

---

_The Daily Planet — Built with Glove × Station — dterminal_
