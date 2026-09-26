# Whale — Technical Implementation Specification

**This document is the single source of truth for building Whale.** Where anything here differs from `Whale_V1_Implementation_Plan.md` or `Whale_YC_Demo_Sprint.md`, this document wins — it consolidates every decision made across both, plus the final visual spec in `Whale_V1_Screens2.html`, into one buildable, unambiguous reference. Treat this as a code-generation spec: exact table names, exact API behavior, exact seed content, exact prompts.

**A few assumptions are flagged inline with `ASSUMPTION:`** — proceed on these unless corrected, don't block on them.

---

## 1. Product summary

Whale is a cross-platform (iOS + Android) messaging app built around three mandatory pillars — **personal AI (W)**, **original-quality file sharing**, and **Bands** (structured, categorized group chats) — plus a fourth surface, **Actions**, that aggregates tasks/schedules/mentions/decisions into one feed. Identity is username-based with a required phone number field (collected, not SMS-verified in V1). The system must support at least **1,000 registered users** and reasonable concurrent Realtime load on Supabase's standard tiers — this is not a scaling problem requiring special architecture.

**Non-negotiable constraints:**
- Cross-platform via a single Expo (React Native) codebase — not two native apps.
- No compression, ever, regardless of file size.
- No end-to-end encryption — Cloud Mode only (§9), stated accurately, never marketed as E2E.
- A fully functional **demo account** with pre-seeded content that also supports live use (§13) — required for YC evaluation.

---

## 2. Information architecture

```
Whale App
 ├── Chats     — unified list: 1:1 chats + Bands together, one list, sorted by recency
 ├── W         — personal AI home (also reachable in-context via a W button next to every composer)
 ├── Actions   — aggregated feed: tasks, scheduled messages, mentions, decisions, file-expiry alerts
 └── Profile   — identity, privacy, AI settings, storage, subscription
```

Four bottom tabs, always visible, in this order: Chats, W, Actions, Profile. No fifth tab. No separate tabs for Bands, original-quality sharing, or public Band discovery — all three live inside the Chats surface (discovery via a header icon, §8a, §10.3a).

**In-chat / drill-down screens** (reached from Chats, not from the tab bar): Personal Chat, Band Chat, W:AI Popup, Chat Settings, Attach Sheet, Send Preview, File Message Card, Schedule Flow, Smart File View, Task View, Create Band, Band Overflow Menu (incl. Create Sub-Band), Family's Calendar/Reminders/Emergency (real, §8), and the Coming Soon placeholder every other category's chips route to.

---

## 3. Identity & authentication

- **Signup fields:** username (required, unique, 3–20 chars, `[a-z0-9_]`, case-insensitive uniqueness), password (required, min 8 chars), phone number (required, stored, **not** SMS-verified in V1), email (optional, for recovery).
- **Recovery:** a 12-word recovery phrase is generated and shown once at signup; user must check "I've saved this" before continuing. If email was provided, also support email-based password reset. There is no SMS OTP fallback — do not build one for V1.
- **Auth implementation — use Supabase Auth's email/password flow under the hood, not custom JWT infrastructure:**
  1. At signup, construct a synthetic internal email: `{username}@users.whale.internal` (never sent anywhere, never shown to the user).
  2. Call `supabase.auth.signUp({ email: syntheticEmail, password })`.
  3. On success, insert a row into `profiles` (username, phone_number, email nullable, recovery_phrase_hash, avatar_color) keyed to `auth.uid()`.
  4. At login, the user enters username + password. Client looks up the synthetic email via a `profiles` query (or an RPC that avoids exposing the table to anon lookups — see RLS note in §5), then calls `signInWithPassword({ email: syntheticEmail, password })`.
- Biometric unlock (Face ID / fingerprint) after first login, backed by platform secure storage (Expo `SecureStore`) — stores a session token, not the password.
- **Device attestation:** Apple App Attest (iOS) / Play Integrity API (Android) checked at signup to rate-limit fake account creation. `ASSUMPTION:` implement as a best-effort check that logs/flags rather than hard-blocks in V1, to avoid false-positive lockouts during the demo period.

---

## 4. Data model (Postgres / Supabase)

```sql
profiles (
  id uuid primary key references auth.users(id),
  username text unique not null,
  phone_number text not null,
  email text,
  recovery_phrase_hash text not null,
  avatar_color text not null,        -- hex, assigned at signup from a fixed palette rotation
  created_at timestamptz default now()
)

dms (
  id uuid primary key default gen_random_uuid(),
  user_a_id uuid references profiles(id),
  user_b_id uuid references profiles(id),
  created_at timestamptz default now(),
  unique (user_a_id, user_b_id)
)

bands (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  type text not null check (type in ('general','family','student','creator','work','community')),
  owner_id uuid references profiles(id),
  parent_band_id uuid references bands(id),   -- null unless this is a sub-Band
  avatar_color text not null,
  member_count int default 1,
  requires_approval boolean default false,   -- gates joins through band_join_requests; general-purpose, not Community-only (see §8, §8a)
  is_public boolean default false,            -- discoverable in the Discover screen, see §8a
  discovery_category text check (discovery_category in ('university','local','professional','startup','travel','event')),  -- required if is_public = true
  location_label text,                        -- free-text, self-declared (e.g. "Atlanta, GA") — NOT live/GPS location
  tagline text,                                -- shown in discovery list, required if is_public = true
  created_at timestamptz default now()
)

band_reports (
  id uuid primary key default gen_random_uuid(),
  band_id uuid references bands(id) not null,
  reported_by uuid references profiles(id) not null,
  reason text not null,
  created_at timestamptz default now()
)

band_members (
  band_id uuid references bands(id),
  user_id uuid references profiles(id),
  role text not null check (role in ('owner','admin','member','guest')),
  joined_at timestamptz default now(),
  primary key (band_id, user_id)
)

rooms (
  id uuid primary key default gen_random_uuid(),
  band_id uuid references bands(id) not null,
  name text not null,
  type text not null check (type in ('chat','files','tasks','scheduled','calendar','reminders','emergency','announcements')),
  created_at timestamptz default now()
)
-- every band gets a 'chat' room on creation.
-- type='family' bands additionally get 'calendar', 'reminders', 'emergency' rooms on creation (§10).

messages (
  id uuid primary key default gen_random_uuid(),
  room_id uuid references rooms(id),
  dm_id uuid references dms(id),
  sender_id uuid references profiles(id) not null,
  content text not null,
  created_at timestamptz default now(),
  check (num_nonnulls(room_id, dm_id) = 1)   -- exactly one target
)

files (
  id uuid primary key default gen_random_uuid(),
  message_id uuid references messages(id),
  uploader_id uuid references profiles(id),
  filename text not null,
  size_bytes bigint not null,
  storage_path text not null,
  tier text not null check (tier in ('small','large')),   -- small = under 1GB, large = at/over 1GB
  encryption_key text,          -- only set for tier='large'; AES-256-GCM per-file key
  expires_at timestamptz,        -- null = never expires (default for both tiers unless sender sets one)
  one_time_download boolean default false,
  password_hash text,
  downloaded_at timestamptz,
  deleted_at timestamptz,
  created_at timestamptz default now()
)

tasks (
  id uuid primary key default gen_random_uuid(),
  band_id uuid references bands(id) not null,
  title text not null,
  assignee_id uuid references profiles(id),
  status text not null default 'todo' check (status in ('todo','in_progress','done')),
  due_at timestamptz,
  source_message_id uuid references messages(id),   -- set when AI or a user converts a message into a task
  created_at timestamptz default now()
)

pinned_decisions (
  id uuid primary key default gen_random_uuid(),
  band_id uuid references bands(id) not null,
  message_id uuid references messages(id) not null,
  pinned_by uuid references profiles(id),
  pinned_at timestamptz default now()
)

scheduled_messages (
  id uuid primary key default gen_random_uuid(),
  sender_id uuid references profiles(id) not null,
  target_type text not null check (target_type in ('dm','room')),
  target_id uuid not null,
  body text not null,
  send_at timestamptz not null,
  recurrence text,               -- null = one-off; else 'daily'|'weekly'|'monthly'
  status text not null default 'pending' check (status in ('pending','sent','cancelled')),
  created_at timestamptz default now()
)

ai_settings (
  id uuid primary key default gen_random_uuid(),
  scope text not null check (scope in ('dm','room')),
  scope_id uuid not null,
  ai_enabled boolean default true,
  allow_summary boolean default true,
  allow_search boolean default true,
  allow_memory boolean default true,
  do_not_remember boolean default false,
  unique (scope, scope_id)
)

mentions (
  id uuid primary key default gen_random_uuid(),
  message_id uuid references messages(id) not null,
  mentioned_user_id uuid references profiles(id) not null,
  created_at timestamptz default now()
)

-- Family Band extension tables
calendar_events (id uuid pk, band_id references bands, title text, starts_at timestamptz, ends_at timestamptz, created_by uuid references profiles, created_at timestamptz default now())
reminders (id uuid pk, band_id references bands, kind text check (kind in ('medication','birthday','general')), title text, remind_at timestamptz, recurrence text, created_at timestamptz default now())
emergency_contacts (id uuid pk, band_id references bands, user_id uuid references profiles, name text, phone text, relationship text, created_at timestamptz default now())
expenses (id uuid pk, band_id references bands, description text, amount numeric, paid_by uuid references profiles, split jsonb, created_at timestamptz default now())
```

**RLS (Row-Level Security):** enable on every table above. Core policy shape: a user can `select` a `messages`/`tasks`/`files`/etc. row only if they're a member of the `band_id` (via `band_members`) or a participant in the `dm_id` (via `dms.user_a_id`/`user_b_id`). `ai_settings` additionally gates whether AI Edge Functions are allowed to read a scope's messages (checked in the Edge Function itself, not just RLS, since the AI calls run with elevated service-role access to assemble context — the Edge Function must explicitly check `ai_settings.ai_enabled`/`allow_summary`/`allow_search`/`allow_memory` before including a scope's messages in any prompt). Never ship the Supabase service-role key to the client.

**RLS carve-out for public discovery (§8a):** the `bands` table specifically needs a second `select` policy: `is_public = true` rows are readable by *any* authenticated user, regardless of `band_members` — that's what makes the Discover screen possible. This carve-out applies **only** to the `bands` table row itself (name, tagline, category, member_count) — it does not extend to that band's `messages`, `tasks`, `files`, or any other table, which stay strictly membership-gated as normal. Get this distinction right: a non-member can see that a public Band exists and what it's about, never its actual conversation content.

---

## 5. Backend architecture

- **Platform:** Supabase (Postgres + Auth + Realtime + Storage + Edge Functions). No separate backend service.
- **Realtime:** Supabase Realtime subscriptions on `messages` (filtered by `room_id`/`dm_id`) power live message delivery, typing indicators (ephemeral, via Realtime Presence, not persisted to a table), and read state.
- **Storage buckets:**
  - `files` — all file uploads (both tiers). Path convention: `{band_id_or_dm_id}/{file_id}-{filename}`.
  - `avatars` — not required for V1 (avatars are colored initials, no image upload needed).
- **Edge Functions** (Deno, deployed via Supabase CLI):
  | Function | Trigger | Purpose |
  |---|---|---|
  | `ai-rewrite` | HTTP, on-demand | §7.1 |
  | `ai-summarize` | HTTP, on-demand | §7.2 |
  | `ai-find` | HTTP, on-demand | §7.3 |
  | `ai-w-query` | HTTP, on-demand | §7.4 (powers the W tab and the W query-result screen) |
  | `file-sweep` | Scheduled (pg_cron, every 15 min) | Deletes storage objects for `files` where `expires_at < now()` or (`one_time_download = true` and `downloaded_at is not null`); sets `deleted_at` |
  | `scheduled-dispatch` | Scheduled (pg_cron, every 1 min) | Sends any `scheduled_messages` where `status='pending'` and `send_at <= now()`; inserts the resulting `messages` row; updates status to `'sent'`; re-enqueues next occurrence if `recurrence` is set |
  | `demo-reset` | Scheduled (pg_cron, every 6 hours) | §13.4 |

---

## 6. Original-quality file sharing — exact logic

**Threshold:** 1,073,741,824 bytes (1 GiB). `size_bytes < threshold` → small tier. `size_bytes >= threshold` → large tier.

**Attach sheet:** exactly three options — Photo, Camera, File. Powered by Whale Tunnel in UI
**Small tier flow:**
1. User picks a file. Client uploads directly via a single `supabase.storage.from('files').upload(...)` call.
2. `files` row inserted with `tier='small'`, no `encryption_key`, `expires_at` null unless the user explicitly sets one.
3. Message sent immediately with the file attached — no progress UI, no options screen. The file card in-thread shows: filename, size, "Original Quality · Compression: None." Nothing else.

**Large tier flow:**
1. Client splits the file into 8 MB chunks.
2. Each chunk is encrypted client-side with AES-256-GCM using a per-file symmetric key generated on-device.
3. Chunks upload via Supabase Storage's resumable (TUS) upload API; client persists a manifest of completed chunk indices to local storage so an interrupted upload resumes rather than restarts.
4. `files` row inserted with `tier='large'`, `encryption_key` (protected by RLS — never exposed to any user except sender/recipients of that room/dm).
5. Send preview screen shows: file card, a **"Send Original"** toggle (on by default, disabling it is not a supported V1 action — leave the toggle visible for transparency but functionally it's always on), and a **"More options"** row (`Expiry, one-time, password ›`) that expands to: expiry picker (24h / 48h / 7 days / never), one-time-download toggle, password field.
6. In-thread file card for large-tier files shows a progress bar while uploading/downloading, and an expiry countdown only if the sender set one.
7. `file-sweep` (§5) handles cleanup.

**Non-user recipients:** **confirmed out of scope — not building this.** No public web download page. A shared file is only reachable by someone with a Whale account who's a member of the room/dm it was shared in. Do not build a separate web app or Vercel-hosted download page for this.

---

## 7. AI features — exact behavior and prompts

Every AI call must first check `ai_settings` for the relevant scope(s) and silently exclude any scope where `ai_enabled=false` or the specific permission (`allow_summary`/`allow_search`/`allow_memory`) is false. If the *only* relevant scope for a user's request is disabled, return a plain message: "AI is turned off for this chat" — never fail silently or hallucinate.

**Model:** Anthropic Claude API (a current Claude model — `ASSUMPTION:` use whichever Claude model is designated for general-purpose chat/reasoning at build time; do not use an open-source/self-hosted model per the earlier explicit decision).

### 7.1 Rewrite (`ai-rewrite`)
- **Trigger:** W:AI popup → "Rewrite my message," with the current composer draft as input.
- **Input:** draft text, target tone (`professional`|`friendlier`|`shorter`|`translate:{lang}`).
- **Prompt:** `"Rewrite the following message to be {tone}. Return ONLY the rewritten text, no preamble.\n\nMessage: {draft}"`
- **Output:** shown in a preview card with "Keep original" / "Use this" — never auto-replaces the composer text without user confirmation.

### 7.2 Summarize (`ai-summarize`)
- **Trigger:** the dismissible AI summary card at the top of a Band, or W:AI popup → "Summarize this chat."
- **Input:** `room_id` or `dm_id`, plus messages since the user's `last_read_at` (or last 50 messages if unread count is large).
- **Prompt:** `"Summarize the key updates from this conversation in 3-5 short bullet points. Conversation:\n{messages, formatted as 'sender: text' per line}"`
- **Category-aware framing** (Band chats only, based on `bands.type`) — prepend to the prompt above:
  | Band type | Framing prefix |
  |---|---|
  | family | "This is a family group chat. Emphasize reminders, events, and anything needing a response from a family member." |
  | student | "This is a student project group. Emphasize deadlines, task ownership, and decisions made." |
  | creator | "This is a creative production team. Emphasize file/version activity, feedback, and approvals." |
  | work | "This is a work team chat. Emphasize decisions, owners, and blockers." |
  | community | "This is a community discussion. Emphasize the most-discussed topics and any moderation-relevant activity." |
  | general | (no prefix) |

### 7.3 Find a message (`ai-find`)
- **Trigger:** W:AI popup → "Find a message," or the Search bar on the Chats screen.
- **V1 implementation — no embeddings/vector DB.** Based on query if single chat Fetch the last 50 messages, across the chats 300 messages (or all messages from the last 30 days, whichever is smaller) across the scope(s) the query applies to (a single room/dm if invoked in-context; all AI-allowed rooms/dms if invoked from the global Search bar). Send them directly to Claude with the query.
- **Prompt:** `"Find messages relevant to this query: \"{query}\"\n\nReturn up to 5 matches as a JSON array of {room_or_dm_label, snippet, approximate_time}. Only include genuine matches — return an empty array if nothing is relevant.\n\nMessages:\n{messages}"`
- Parse the JSON response and render as result cards (matches the Search results screen in the mockup: grouped by Messages / Bands / Files sections — `ASSUMPTION:` Files results come from a simple `ilike` filename match against the `files` table, not from the AI call).

### 7.4 W — standalone tab + query result (`ai-w-query`)
- **Trigger:** typing into the "Ask W anything" hero input on the W tab and submitting.
- **No scheduled distillation job in V1** — compute on demand, per query. Fetch the last 50 messages (or last 3 days, whichever is smaller) from every room/dm where `ai_settings.allow_memory=true` for the requesting user.
- **Prompt:** `"You are W, the user's personal AI assistant inside Whale. Answer their question using ONLY the conversation context provided below. After your answer, list which Band/chat and rough time each fact came from. If the context doesn't contain an answer, say so plainly — never guess.\n\nQuestion: {query}\n\nContext:\n{messages, grouped by band/dm with labels}"`
- **Response rendering:** the answer text goes in `.w-answer .wa-text`; citations (parsed from the structured part of the response — instruct Claude to end its response with a line `CITATIONS: Band Name · timeframe | Band Name · timeframe`) go in `.wa-cites` as chips. Below the answer, always show 2 follow-up suggestion chips (`ASSUMPTION:` generate these as a second, cheap Claude call or a simple static set like "What tasks are still open?" — for V1, static/heuristic suggestions are acceptable, not a hard requirement to generate dynamically).
- **W tab default state (no query yet):** hero input with placeholder "What did I miss today?", two suggested prompt chips, then two sections — "Unread summaries" (bands with unread messages, one `ai-summarize` call per band shown, or a lightweight unread count if compute budget is a concern) and "Pending tasks" (query `tasks` where `assignee_id = current_user` and `status != 'done'`, sorted by `due_at`).

### 7.5 Task detection
- **Trigger:** automatic, async, non-blocking — after a message is inserted in a Band room (not DMs), a lightweight check (`ASSUMPTION:` a cheap Claude call with a strict "does this message contain an actionable commitment? reply ONLY 'yes' or 'no'" prompt, run async so it never blocks message send) flags candidate messages. On a "yes," surface a subtle in-thread prompt: "Turn this into a task?" — never auto-create a task without confirmation.

---

## 8. Band categories — exact feature-to-implementation mapping

**Confirmed scope decision: Family is the only category with working category-specific features.** Every other category (Student, Creator, Work, Community) gets full category *selection* — its own icon, color, and AI framing (§7.2) apply correctly — but tapping any of its category-specific shortcut chips opens the **Coming Soon** screen (§10, new) instead of a real view. Do not build Deadline Board, Notes Hub, Media Vault, Review & Approval, Edit Space, Decision Log, Action Tracker, or any other named category feature outside Family. This replaces the earlier "these are just views with different labels" approach — it's simpler still: don't build the view at all yet, show Coming Soon.

The one exception is Community's **Join Approval**, which is being built for real (see the new `band_join_requests` table below) — everything else in Community remains Coming Soon.

| Category | Shortcut chips shown | What happens when tapped |
|---|---|---|
| Family | Calendar, Reminders, Emergency | **Real, working** — `calendar_events`, `reminders`, `emergency_contacts` (§4). `bands.type='family'` auto-provisions `calendar`, `reminders`, `emergency` rooms at creation. |
| Student | Deadlines, Notes Hub, Submission | Coming Soon screen |
| Creator | Media Vault, Review, Edit Space | Coming Soon screen |
| Work | Decisions, Actions, Files | Coming Soon screen |
| Community | Announcements, Topic Rooms, Join Approval | Join Approval is real (below); the rest open Coming Soon |
| General | None shown | N/A — General never had category-specific chips |

**Community — Join Approval (the one built exception):**
```sql
band_join_requests (
  id uuid primary key default gen_random_uuid(),
  band_id uuid references bands(id) not null,
  user_id uuid references profiles(id) not null,
  status text not null default 'pending' check (status in ('pending','approved','denied')),
  requested_at timestamptz default now(),
  decided_by uuid references profiles(id),
  decided_at timestamptz,
  unique (band_id, user_id)
)
```
Flow: any Band can be toggled (at creation or in Band settings) to `requires_approval = true` — this is a general-purpose mechanism (§4), not Community-specific; it's the same switch used by public discovery (§8a). When set, joining inserts a `band_join_requests` row with `status='pending'` instead of inserting directly into `band_members`. Admins/owners see a pending-requests list (a simple screen: requester name + Approve/Deny buttons); approving inserts into `band_members` and sets `status='approved'`; denying just sets `status='denied'`. This is a small, self-contained feature.

**Sub-Bands (universal, every category):** `bands.parent_band_id` — creating a sub-Band is creating a normal `bands` row with `parent_band_id` set. The Band Overflow Menu → "Create Sub-Band" opens the same Create Band flow (§ screens), pre-associating the new band's `parent_band_id`.

**Category selection UI:** the Create Band screen shows all six types (General, Family, Student, Creator, Work, Community) as selectable rows with icon + one-line description (exact copy in §11 screen spec). Selecting a type sets `bands.type` and, for `family` only, triggers auto-provisioning of the extra rooms.

---

## 8a. Public Band discovery — **in scope for this build**

**Positioning:** *"Discover people through conversations, not followers."* This is the mechanism that gives someone a reason to open Whale before any of their friends have joined — every other feature in this spec assumes some existing contact is already reachable; this one doesn't. Keep it explicitly distinct from an algorithmic feed: discovery is search/browse over public Bands, ranked by nothing more clever than recency/category match — no engagement-optimized ranking, no infinite scroll of content, no passive consumption. You find a Band, you join it, you talk. That's the whole interaction.

**Discovery categories (`bands.discovery_category`):** `university`, `local`, `professional`, `startup`, `travel`, `event`. Mapped from the original examples: "Georgia State Fall 2026 Students" → university; "Atlanta Weekend Soccer" / apartment-community Bands → local; "AI Engineers in Atlanta" → professional; "YC Founders Building in Healthcare" → startup; travel Bands → travel; "People Attending This Conference" → event.

Location is handled via `location_label`, a free-text, self-declared field ("Atlanta, GA") — zero device permissions required, no GPS involved anywhere in this feature.

**Discover screen (new, §10.3a):** reached via a compass icon in the Chats header, next to the compose icon. Search field + category filter chips (All / University / Local / Professional / Startup / Travel / Event). Results are Band cards: name, tagline, `location_label` if set, member count, category tag. Tapping a card opens a lightweight preview (tagline, member count, category — **not** the Band's actual messages, per the RLS carve-out above) with a **Join** button (if `requires_approval=false`) or **Request to Join** button (if `true`).

**Join mechanics:** reuses `band_join_requests` and the admin approval screen already specified above — no new join infrastructure, just a new entry point (search/browse instead of an invite link) into the same mechanism.

**Minimum safety net for V1 — deliberately small, not a full moderation system:** a **Report** action on any public Band (in its preview and in its overflow menu once joined) inserts a row into `band_reports`. V1 does not need automated moderation or an admin review dashboard — reports are logged for manual founder review. Do not build AI-based content moderation for this feature; that was already flagged elsewhere in this spec as needing its own dedicated design pass, and this is exactly that situation — a minimal, honest safety net now, a real moderation system later.

**The bootstrap problem — this is a content/seeding requirement, not just an engineering one:** an empty Discover screen is worse than no feature at all. §13's demo seed data includes four real, populated public Bands specifically so this never renders empty in the demo. Post-demo, the same logic applies to real launch — public Bands need to be seeded with genuine activity before real strangers will find value in browsing them; this is not solved by the schema alone.

---

## 9. Encryption & privacy model

One mode — **Cloud Mode** — stated accurately everywhere in the product and in any marketing copy: messages are encrypted in transit (TLS) and at rest (Supabase-managed encryption), processed server-side to power AI features. This is explicitly **not** end-to-end encryption, and the product must never claim it is.

`ai_settings` (§4) is a separate control from encryption: a user can leave a chat's encryption as-is while turning AI off for that specific chat via Chat Settings → AI Mode (On / Allow summaries / Allow semantic search / Allow memory / Do not remember this chat). The UI must never conflate these two controls into one toggle.

Live location sharing is explicitly out of scope for V1 (and Family Band specifically) due to background-location permission review overhead on both app stores — do not build it.

---

## 10. Screen-by-screen specification

Visual reference: `Whale_V1_Screens2.html` is the authoritative visual spec — match spacing, color, and type exactly as implemented there. Palette and type tokens below are extracted directly from that file.

### 10.1 Design tokens
| Token | Value |
|---|---|
| Background | `#FAFAF8` |
| Ink (primary text) | `#171B1A` |
| Ink soft | `#4B5652` / `#6B7570` |
| Ink faint | `#9AA39F` / `#8A9490` |
| Line/border | `#ECEDE9` / `#E7E8E4` |
| Accent (primary indigo) | `#4F46E5` |
| Accent strong (dark indigo) | `#3730A3` |
| Accent soft (light tint bg) | `#E8E7FC` |
| Accent soft border | `#C7C4F5` |
| Accent very-light tint (cards) | `#F5F4FE` |
| Display/heading font | `-apple-system, "SF Pro Display", "Segoe UI", Arial, sans-serif`, weight 800, letter-spacing -0.03em |
| Body font | `-apple-system, BlinkMacSystemFont, "Segoe UI", "Helvetica Neue", Arial, sans-serif` |
| Timestamp/metadata font | `ui-monospace, "SF Mono", monospace` |
| Message bubbles | received `#F0F1EE` bg / sent `#4F46E5` bg, white text, 16px radius, 6px on the "tail" corner |
| Card radius (list rows, task cards, file rows) | 16px |
| Avatars | circular, 44px, colored fill + initials (2 letters) |

### 10.2 Onboarding
Single-decision screens: wordmark "Whale" → username field (`@handle`, helper text "This is how people find you on Whale — no phone number required" — `ASSUMPTION:` update this helper copy since phone number is now collected; correct copy: "No one sees your phone number — it's just for account recovery") → phone number field → password field → recovery phrase display + confirmation checkbox → done. One field group per screen, "Continue" CTA (`#4F46E5` fill).

### 10.3 Chats (home tab)
- Header: "Chats" (display font) + two icons top right — **compass (Discover, new)** and compose (pencil).
- Body: single list, Bands and 1:1 chats mixed by recency — **do not split into separate sections**. Each row is a card (`#fff` bg, 1px `#ECEDE9` border, 16px radius, subtle shadow): avatar, name (bold), timestamp (mono, top-right), preview text below, unread dot (`#4F46E5`) if unread. Band rows additionally show a small "kind" tag (group icon) distinguishing them from 1:1 rows — this is the only visual distinction; no physical separation.
- Search: a light card (`#F1F2EE` bg, bordered) positioned **above the tab bar, below the chat list** — not at the top of the screen. Fixed/sticky in that position while the list above scrolls. This searches the user's own chats/files/tasks (§10.4) — it is separate from Discover, which searches public Bands the user hasn't joined (§10.3a).
- Tab bar: Chats (active) / W / Actions / Profile.

### 10.3a Discover (new screen, §8a)
Reached by tapping the compass icon. Header "Discover." A search field, then category filter chips (All / University / Local / Professional / Startup / Travel / Event). Below: public Band result cards — name, tagline, `location_label` (if set), member count, category tag. Tapping a card opens a preview (same info, slightly expanded, no message content) with a **Join** or **Request to Join** button depending on `bands.requires_approval`. A small "Report" affordance is available from the preview's overflow menu.

### 10.4 Search results
Reached by tapping the search field on Chats. Full-screen: typed query shown at top, results grouped under section labels (Messages / Bands / Files), each result showing source + snippet with the matched term highlighted. This is scoped to the user's own joined chats — it does not surface public Bands the user hasn't joined (that's Discover, §10.3a).

### 10.5 Personal chat (1:1)
Header: back chevron, small avatar, name, "Active now" status. Thread: bubbles (received left/grey, sent right/indigo), grouped by sender with minimal gaps between consecutive messages from the same sender. Composer: rounded pill (`#F1F2EE`), `+` attach icon, text field, `W` button (opens W:AI popup), send button (`#4F46E5` filled circle, arrow icon).

### 10.6 W:AI popup
Triggered by the `W` button in any composer. A bottom sheet over a dimmed/blurred thread: a prompt field ("Ask W or choose an action") followed by action rows — Rewrite my message, Summarize this chat, Find a message, Create action items, Schedule this message. Each row: icon chip + label, tap target the full row width.

### 10.7 Band chat
Same shell as personal chat, plus: a dismissible AI summary card above the thread ("Since you left · N updates," bulleted, close `×`), and a shortcut-chip row above the composer showing that Band's category-specific chips (per §8). For Family, these chips are real and open working screens. For every other category, tapping a chip opens the Coming Soon screen (§10.7a) — the chip itself still renders with the correct icon/label so the category feels represented, it just doesn't have a working view behind it yet.

### 10.7a Coming Soon (new screen)
A minimal placeholder, not a dead end: icon (reuse the tapped feature's icon), the feature name as the title, one line of copy — `"{Feature name} is coming soon for {Category} Bands."` — and a back affordance. `ASSUMPTION:` no further interactivity needed (no waitlist signup, no notify-me toggle) for V1; this is purely to avoid a broken/missing screen when a non-Family category chip is tapped, not a growth mechanism.

### 10.8 Create Band
Header "Create Band." Band name field. Category list — six rows (General/Family/Student/Creator/Work/Community), each: icon chip, name, one-line description (copy below), selected state = light indigo tint background + checkmark.

Exact category descriptions to display:
- General — "Just chat and the essentials"
- Family — "Calendar, reminders, emergency"
- Student — "Deadlines, notes, study planning"
- Creator — "Media vault, review, edit space"
- Work — "Decisions, tasks, approvals"
- Community — "Announcements, topic rooms"

CTA: "Create Band" (`#4F46E5` filled).

### 10.9 Band overflow menu
Triggered by the `⋯` icon in a Band header. Bottom sheet: Band Info, Members, **Create Sub-Band** (visually highlighted — light indigo tint row), Leave Band (destructive, red text). Available identically on every category.

### 10.10 Attach sheet → Send preview → sent state
Per §6. Attach sheet: Photo / Camera / File, nothing else. Send preview (large tier only — small tier skips straight to sending): file card, "Send Original" toggle, collapsed "More options" row. In-thread: file card with progress (large tier, while uploading) or static info (small tier, no progress needed).

### 10.11 Schedule flow
Triggered by long-press on send, or via W:AI popup "Schedule this message." Bottom sheet: quick options (Today 6pm, Tomorrow 9am — one visually highlighted as suggested default), "Custom date & time," Confirm CTA.

### 10.12 Smart File View / Task View / Family Calendar
The shared-core Smart File View and Task View (available in every Band, any category) and Family's real Calendar/Reminders/Emergency screens all use the same underlying list-card pattern (icon + title + metadata + chevron or status pill), differentiated by header title and filter chips where relevant (All/Photos/Videos/Expiring soon for files). Notes Hub, Deadline Board, Media Vault, and every other non-Family category label route to Coming Soon (§10.7a) instead — they are not real screens in V1.

### 10.13 W (standalone tab)
Default state: dark hero card (`#171B1A` bg with a subtle indigo radial glow) containing "ASK W ANYTHING" label + input ("What did I miss today?" placeholder), two suggested-prompt chips below, then "Unread summaries" and "Pending tasks" sections as white card rows. **Query-result state** (after typing + submitting): hero now shows the typed query in place of the placeholder; below it, a white answer card (`.w-answer`) with the synthesized response and citation chips; below that, two follow-up suggestion chips replacing the original defaults. Tab bar present, W active.

### 10.14 Actions
Header "Actions." A feed of cards, each: colored icon chip (semantic color per type — task=teal, scheduled=indigo, alert=warm red, mention=ink, decision=slate), title, subtitle. Aggregation logic per the "Actions feed" spec (§7 area — see below).

**Actions feed query (exact):**
```
tasks: assignee_id = current_user AND status != 'done' AND due_at < now() + interval '7 days', across all member bands
scheduled_messages: sender_id = current_user AND status = 'pending'
files: tier = 'large' AND expires_at < now() + interval '24 hours' AND (uploader_id = current_user OR current_user is a member of the file's band/dm)
mentions: mentioned_user_id = current_user AND created_at > (last time user opened Actions — track via a profiles.last_actions_view_at column, ASSUMPTION: add this column)
pinned_decisions: pinned_at > now() - interval '48 hours', across all member bands
```
Combine, sort by recency, cap at 20 items. Tapping an item navigates to its source (the Band/chat/file/task).

### 10.15 Chat Settings
Header "Chat Settings." Rows: Members, Media and Files, Starred Messages, then an "AI Mode" section with the exact controls from `ai_settings` (§4): On for this chat (radio), Allow summaries (toggle), Allow AI search (toggle), Allow memory (toggle), Do not remember this chat (toggle). Clear Chat (destructive) at the bottom.

### 10.16 Profile
Header "Profile." Avatar + name + username + "Show QR code" button. Settings rows: Privacy Settings, W:AI Settings, Storage and Data (shows usage), Notifications, Subscription Plan. **No phone number field shown** — it's collected at signup for recovery purposes only, never displayed in the UI.

---

## 11. Business model & resources

Restated from the main plan, unchanged:
- **Free tier:** 5 GB total original-quality storage, rate-limited AI calls per day.
- **Premium:** 15 GB, higher/no AI rate limits. Price point unvalidated — treat $4.99–$9.99/month as a starting hypothesis, not a commitment.
- **Required accounts/services:** Supabase project (free tier to start), Anthropic API key + billing (usage-based, budget ~$20–50 for initial build/test), Expo/EAS account (free tier), Apple Developer Program ($99/year, optional/parallel — not required for Expo Go demo), Google Play Console ($25 one-time, optional). No Voyage AI / vector DB account needed (§7.3 explicitly avoids embeddings for V1).

---

## 12. Local feature review — verify before mobile testing

**Purpose: catch backend/logic bugs by calling Edge Functions and querying the database directly, before burning time on the mobile UI loop.** Use `curl`, Postman, or a simple script against the Supabase project (local dev instance or a staging project) for every row below before installing on a device.

| Feature | How to verify locally | Concrete use cases to check |
|---|---|---|
| Auth | `curl` the Supabase Auth REST endpoints directly | Sign up with a taken username fails cleanly; login with wrong password fails cleanly; recovery phrase login path works |
| Messaging (DM) | Insert rows into `messages` via the Supabase client library in a script, subscribe via Realtime in a second script | Message inserted by user A appears in user B's Realtime subscription within ~1s; RLS blocks a user not in the `dm` from reading it |
| Messaging (Band) | Same, scoped to a `room_id` | A user NOT in `band_members` cannot select messages for that room (RLS test — attempt with their own JWT, expect empty/denied) |
| Small file upload | Call `storage.upload` directly with a test file under 1GB | File appears in the `files` table with `tier='small'`; no `encryption_key` set; downloadable via a signed URL |
| Large file upload | Script a chunked upload against a >1GB test file (can be a dummy file of that size, doesn't need real content) | Upload resumes correctly if interrupted mid-chunk (kill the script, rerun, confirm it continues from the last completed chunk, not from zero); `file-sweep` deletes it after manually setting `expires_at` to the past and manually invoking the function |
| Rewrite | `curl` the `ai-rewrite` Edge Function directly with a sample draft + each tone option | All four tones (`professional`/`friendlier`/`shorter`/`translate`) return sensible output; empty draft is rejected with a clear error, not a crash |
| Summarize | `curl` `ai-summarize` against a room with seeded messages (§13) | Output is 3-5 bullets, not a wall of text; category framing changes the emphasis (test the same message set against a `family`-type band vs. a `work`-type band and confirm the outputs read differently) |
| Find a message | `curl` `ai-find` with a query that has a real match and one with no match | Real-match query returns the correct message; no-match query returns an empty array, not a hallucinated result |
| W query | `curl` `ai-w-query` with "what did I miss today?" against the seeded demo account (§13) | Answer correctly references the seeded Bands/messages; citations point to real band names, not fabricated ones; a chat with `ai_settings.allow_memory=false` is correctly excluded (seed one such chat specifically to test this) |
| Task auto-detection | `curl` the task-detection check with an obviously actionable message ("I'll send the deck by Friday") and an obviously non-actionable one ("lol nice") | Actionable message flags "yes"; casual message flags "no" — check both directions, false positives are as bad as false negatives here |
| Scheduled messages | Insert a `scheduled_messages` row with `send_at` in the past, manually invoke `scheduled-dispatch` | Message is sent (appears in `messages`), status flips to `'sent'`; a `recurrence='weekly'` row re-enqueues a new pending row roughly 7 days out |
| Actions feed | Seed one of each type (overdue task, pending scheduled message, expiring-soon file, a mention, a recent pinned decision) for the demo account, query the aggregation directly | All five item types appear; sorting is by recency; a `done` task does NOT appear |
| Family category views | Query `calendar_events`/`reminders`/`emergency_contacts` for the seeded Family Band | Each returns only that band's rows; a Student/Creator/Work/Community band tapping its own category chips correctly hits no real query at all — confirm the client routes those taps to Coming Soon rather than attempting a query |
| Community Join Approval | Set `bands.requires_approval=true` on a test community band, attempt to join as a second user | A `band_join_requests` row is created with `status='pending'`, NOT a `band_members` row; after an admin approves it (update `status='approved'` + insert `band_members`), the user can now read that band's messages (RLS check) |
| Public Band discovery (§8a) | Query `bands where is_public = true` as a user who is NOT a member of any of them | Row is returned (confirms the RLS carve-out works) but a parallel query for that band's `messages` returns empty/denied (confirms the carve-out is scoped correctly — this is the single easiest thing to get wrong here, verify it explicitly); joining an instant-join band inserts into `band_members` directly; joining a `requires_approval=true` band inserts into `band_join_requests` instead |
| Sub-Bands | Create a band with `parent_band_id` set to another band's id | Query confirms the relationship; a user who's a member of the parent but not explicitly added to the sub-Band cannot read the sub-Band's messages (sub-Bands have independent membership — verify this isn't accidentally inherited) |
| Per-chat AI permission | Set `ai_settings.ai_enabled=false` for a specific room, then call `ai-summarize` against it | Function returns the "AI is turned off for this chat" message, does not silently proceed |
| 1,000-user headroom | Seed 1,000 dummy `profiles` rows + a realistic spread of bands/messages via a script | No query in the app (Chats list, Actions feed, W query) takes more than ~1-2s against this volume; if anything is slow, check for a missing index before it ever reaches mobile testing |

Only after every row above passes should mobile-device testing begin — at that point you're testing UI/UX and platform behavior, not chasing backend logic bugs through the much slower phone-rebuild loop.

---

## 13. Demo account — exact seed specification

**Requirement, stated plainly: the demo account is a fully real, fully functional account.** Seeding pre-populates its initial state; it does not mock, fake, or special-case any behavior. A judge must be able to send a new message, use every W:AI action, upload a real large file, create a task, and see it all work against the live backend, indistinguishable from any other account.

### 13.1 Credentials
- Username: `demo`
- Password: `WhaleDemo2026!`
- Phone number: a placeholder value (`+10000000000`)
- Displayed in the YC submission text and video description exactly as above — no separate signup needed to try it.

### 13.2 Seed content (matches the mockups in `Whale_V1_Screens2.html` exactly, for visual consistency between what's demoed and what's built)

**Bands:**
| Name | Type | Members | Seed content |
|---|---|---|---|
| AI Team Band | work | 128 (seed 4-5 real dummy profiles, display count as 128) | Messages: "alright, search is looking solid in staging" / "nice — memory next then." A pinned decision. 2-3 tasks matching the Task View mockup (Ship search to staging — in progress; Review privacy labels — to do; Finalize band_memory job — done). |
| Family Band | family | 6 | Messages: "Mom: don't forget dinner Sunday 6pm" / "on it, I'll be there by 5:30" / "Dad: I'll bring the car in for its checkup Monday." Calendar events: "Dinner at Mom's" (Sun 6pm), "Dad's checkup" (Mon 10am). One reminder: "Grandma's medication — 8:00 AM daily." One emergency contact. |
| Wedding Film Team | creator | 12 | Messages: "Priya: uploaded the ceremony raw clips" / "on it, starting the edit now." Tapping this Band's shortcut chips (Media Vault, Review, Edit Space) shows Coming Soon — the messages are real, the category chips are not. |
| GSU AI Project Band | student | 5 | Messages: "Charan: pushed the dataset link, check it out" / "nice, I'll start on the demo script tonight." Tapping this Band's shortcut chips (Deadlines, Notes Hub, Submission) shows Coming Soon — same reasoning as Creator above. |

**1:1 chats (DMs):**
| Contact | Seed messages |
|---|---|
| Shareef | "Hey are we still on for the biryani place at 8?" / "yeah works for me, see you there" / "Perfect 🙌" |
| Charan | "weekend trip pics are on the way" |
| Priya | "pushed the fix, can you review?" |

**Files:** seed one genuinely large file (~1.1–1.5 GB, real content — e.g., a sample video, not a zero-byte placeholder) in AI Team Band, named `CheXpert_Dataset.zip`, so the resumable-upload/progress/expiry mechanics are authentically demonstrable, not faked. `ASSUMPTION:` a synthetic/dummy binary of that size is acceptable content — it doesn't need to be a real medical dataset, just a real file of that size sitting in Storage.

**Tasks, decisions, mentions:** seed at least one of each so the Actions feed is non-empty on first view (§12's Actions feed test row).

**Public Bands (§8a) — seed four, so Discover never renders empty:**
| Name | `discovery_category` | `location_label` | `requires_approval` | Seed messages |
|---|---|---|---|---|
| AI Engineers in Atlanta | professional | Atlanta, GA | false (instant join, demo the direct path) | "Anyone going to the ML meetup Thursday?" / "yeah I'll be there, bringing a friend too" / "nice, see you both there" |
| Georgia State Fall 2026 Students | university | Atlanta, GA | false | "anyone else in CS 4650 this semester?" / "yep! the workload looks rough ngl" / "we should start a study group early" |
| Atlanta Weekend Soccer | local | Atlanta, GA | true (demo the request-to-join + admin-approval path; demo account is admin) | "game's on for Saturday 10am at Piedmont Park" / "in, bringing 2 more people" |
| YC Founders Building in Healthcare | startup | *(unset — not every public Band needs a location_label)* | true | "anyone dealt with HIPAA compliance for a chat feature?" / "yeah happy to jump on a call, DM me" |

The demo account should not be pre-joined to all four — leave at least one (e.g., YC Founders) unjoined so the live demo can show the actual Discover → preview → Join/Request flow end-to-end, not just a pre-populated list.

### 13.3 What must work live on this account
- Sending a new message in any seeded Band or DM (Realtime delivery).
- W:AI popup — rewrite, summarize, find a message — all functioning against real seeded content.
- W tab — typing "what did I miss today?" must return a real answer grounded in the seeded messages, with accurate citations.
- Uploading a new file, both tiers.
- Creating a task, scheduling a message, creating a new Band (any category) including a sub-Band.
- Discover (§8a): searching/filtering public Bands, joining one directly (instant-join Bands), sending a request to join one that requires approval, and — as the admin of the Atlanta Weekend Soccer band — approving that request from the other side.

### 13.4 Reset mechanism
`demo-reset` Edge Function (§5), scheduled every 6 hours: truncates the demo account's bands/messages/tasks/files/etc. back to the exact seed state in §13.2, so repeated or messy judge sessions don't degrade the demo over the review window — this includes re-establishing the four public Bands in §13.2 and un-joining the demo account from any it joined during a prior session, so the Discover flow is repeatable for the next person too. Confirmed at 6 hours (§15).

---

## 14. Build phases

1. **Foundations** — Supabase project, schema (§4) + RLS incl. the public-Bands carve-out, Expo app skeleton, auth flow (§3).
2. **Core messaging** — DMs, Bands (all six types selectable; Family's category views fully real; Student/Creator/Work/Community route to Coming Soon except Join Approval, §8), Realtime, unified Chats list (§10.3).
3. **Original-quality sharing** — both tiers (§6), attach/send/receive flow.
4. **Public Band discovery** — Discover screen, join/request-to-join flow, Report action (§8a, §10.3a).
5. **AI layer** — rewrite, summarize, find, W tab + query-result, task auto-detection (§7).
6. **Actions + Scheduling** — aggregation feed, scheduled-message dispatch (§10.14, §5).
7. **Demo account** — seed script including the four public Bands (§13), reset function.
8. **Local review pass** — every row in §12, before touching a physical device.
9. **Mobile testing & polish** — real iPhone + Android via Expo Go, per the platform decision in the sprint doc (iPhone as the primary/hero device for the final demo video, per the most recent platform discussion).
10. **Record demo, submit.**

---

## 15. Resolved decisions log

Everything previously flagged as an open question has been confirmed, except one:

| # | Decision | Resolution |
|---|---|---|
| 1 | Web download page for non-app recipients | **Cut entirely** — not building it (§6) |
| 2 | Category-specific features outside Family | **Coming Soon placeholder** for Student/Creator/Work/Community's category chips — do not build the underlying views (§8, §10.7a) |
| 3 | Community's Join Approval | **Building it for real** — new `band_join_requests` table + `bands.requires_approval` (§4, §8) |
| 4 | Demo-reset cadence | **Confirmed at 6 hours** (§13.4), no change |
| 5 | Device attestation strictness | **Still open** — best-effort logging (not hard-blocking) is the current spec (§3); confirm or override before the auth phase of the build |

Remaining inline `ASSUMPTION:` markers elsewhere in this document (Claude model selection, `ai-find`'s file-match method, follow-up suggestion generation, task-detection prompt, `tasks.category`/`is_blocker` columns, seed-file content, `profiles.last_actions_view_at`) are lower-stakes implementation details, not scope decisions — proceed on them as written.
