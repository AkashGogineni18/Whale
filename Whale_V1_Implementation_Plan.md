# Whale V1 — Implementation Plan
iOS + Android · Username-based identity · Chats, W (AI), Actions, Profile

> **How to use this document:** this is written as a build spec for an AI coding agent (e.g. Claude Code) to implement directly — there is no hired team assumed anywhere in this plan. Feed it one phase at a time (§15), in order; each phase is scoped to be a coherent, testable unit of work rather than a role assignment.

---

## 1. Executive summary

Whale V1 is a familiar messaging app with three things built into the normal chat experience rather than bolted on as separate destinations: **personal AI (W)**, **original-quality file sharing**, and **Bands** (structured group chats) living in the same inbox as 1:1 chats. The product rule that shapes every screen in this document: *Whale should not force users to learn a new app structure — it should feel like normal messaging, with AI, organization, and original-quality sharing built naturally into the chat experience.*

Four main destinations, not more: **Chats, W, Actions, Profile.** Everything else — a specific Band, a file, a scheduled message, AI settings — is reached by drilling into a chat or a settings screen, never by adding a fifth tab.

---

## 2. Information architecture

```
Whale App
 ├── Top Search (global, on the Chats screen)
 ├── Chats        — personal chats + Bands, one list
 ├── W            — personal AI assistant home
 ├── Actions      — tasks, scheduled messages, mentions, alerts
 └── Profile      — identity, privacy, AI settings, storage, subscription
```

**In-chat screens/overlays** (reached from within Chats, not from the tab bar): Personal Chat, Band Chat, W:AI Popup, Chat Settings, File Picker, File Message Card, Schedule Message Flow, Smart File View, Task View, AI Summary View.

**Deliberately not in V1:** a separate Whale Tunnel screen, a separate AI Memory main screen, a separate Bands tab, a separate Scheduled Messages main screen, voice/video calling, voice-note transcription. Each of these is either folded into an existing screen (Tunnel → file picker, Memory → Profile/W settings, Bands → Chats list, Scheduled → inside each chat) or pushed to V2 (calling, voice transcription).

---

## 3. What's in V1

| Included in V1 | Deferred to V1.1 | Deferred to V2+ |
|---|---|---|
| Username-based signup/login, no phone number | Smart notification triage (folded partly into Actions ranking) | Bands at 500K-member scale (infra scaling, not a feature gap) |
| Unified Chats list: personal chats + Bands, pinned row, global search | — | Multi-Band scheduled broadcast |
| Band Chat: normal group chat + pinned summary card + lightweight Files/Tasks/Scheduled shortcuts | — | Live travel location sharing (§10) |
| **Family Band**: calendar, medication/birthday reminders, documents vault, emergency contacts, expense split | — | Voice-note transcription + search |
| Original-quality file sharing (up to 5 GB / 15 GB premium) as a normal attachment — no separate Tunnel UI | — | Voice/video calling |
| — | — | Band categories beyond Family — Student, Creator, Work, Community, plus the General category's sub-Band capability, each with its own specialized module set (§21) |
| W: personal AI home, W:AI popup inside every chat, agent memory, semantic search, rewrite-before-send | — | — |
| Actions: tasks, scheduled messages, mentions, file-expiry alerts, decisions — one aggregated feed | — | — |
| Per-chat AI permission (AI off / summaries / search / memory / do-not-remember) | — | — |
| Cloud Mode encryption (transit + at rest) | — | — |

No true end-to-end encryption / Privacy Mode — out of scope entirely, not deferred (§9). Per-chat AI permission is a separate control from encryption: even inside Cloud Mode, a user can deny AI access to a specific chat (§11).

---

## 4. Identity: username-based, no phone number

- **Signup:** username + password, optional email (recommended) for account recovery. Optional biometric unlock (Face ID / fingerprint) via platform keychain after first login.
- **Recovery without a phone number:** email-based reset if provided, plus a 12-word recovery phrase / backup-code set shown once at signup. Build this in the same phase as signup, not after.
- **Abuse/spam:** device attestation (Apple App Attest / Google Play Integrity API) at signup, rate-limited account creation per device/IP.
- **Discovery:** username search, QR code, or shareable profile/Band invite link — no phone-contact-hash matching.
- **Profile screen** (§4 of the tab bar, detailed in §12) shows username, QR code, and account controls. No phone number field — it would contradict the no-phone-number identity model that's a deliberate product decision, not an oversight.

---

## 5. UI direction: familiar bones, elevated execution

The explicit product rule: **Whale should look like a smarter app, not an old-school SMS/WhatsApp clone.** A flat list of grey rows with a plain header reads as "traditional messaging app" even if the underlying feature set is different — the visual language has to signal "modern and professional" on first glance, while the structure stays instantly familiar (nobody should have to learn where anything is).

Concretely, what elevates the Chats screen specifically:

- **Pinned row at the top** — pinned chats/Bands as a horizontal row of avatar chips (with a thin accent ring), not buried in the list. This is the single biggest visual differentiator from a plain SMS list and is functional, not decorative.
- **Global search as a real search bar**, not a small icon — a pill-shaped, always-visible search field at the top of Chats ("Search across chats, Bands, files, tasks, decisions"), giving the home screen an anchor the way a search-first product looks considered rather than a bare list.
- **Visual distinction between a person and a Band** without extra chrome — Band avatars carry a small structural badge (e.g., a subtle grouped-circle mark), personal chat avatars don't. The eye should tell the two apart at a glance in a mixed list.
- **Semantic accenting, used sparingly** — the brass accent marks the things that matter (unread dot, active tab, a pinned ring, a highlighted AI element), not decoration. A screen where the accent is used everywhere reads as unconsidered; used only at 2–3 points per screen, it reads as deliberate.
- **W:AI is a visible button, not a hidden gesture** — inside any chat, a persistent small W icon sits next to the composer. Tapping it opens the W:AI popup (§11). This is the "AI built naturally into the chat experience" principle made concrete: it's one tap away, never a separate app-within-the-app.
- **Bands feel like group chats first** — inside a Band, the default view is just messages, exactly like a personal chat. A pinned summary card and a row of lightweight shortcut chips (Files · Tasks · Scheduled) sit above the composer, not a persistent tab strip fighting for attention. The organization is available, not imposed.
- **Theme:** light mode is the primary, first-class theme — bright neutral backgrounds, soft grey-scale surfaces, a single restrained accent, generous whitespace. Modern and airy, not the near-black chat-app default.
- **Typography:** a confident display face for the wordmark and section headers (something with real character, not the same system sans as the body text), clean system font for chat body text. The display face is part of what signals "designed product," not just "functional app."
- **Motion:** short spring-based transitions (send, popup open/close, tab switch); respect OS-level reduced-motion settings.
- **Onboarding:** one decision per screen — username → optional email → notification permission → first contact/Band. (This screen already tested well — keep its restraint as the design-quality bar for the rest of the app.)

---

## 6. Client platform: Expo (React Native) + EAS

**Recommendation: Expo-managed React Native, built and shipped via EAS**, not bare React Native or fully native Swift/Kotlin — unchanged from the earlier decision. With no dedicated mobile team, Expo abstracts native project configuration, code signing, and provisioning profiles that are fragile to hand-maintain, and EAS Build/Submit handles cloud builds and store submission without a locally configured native toolchain.

Where to still reach for a native module regardless: file chunking/encryption for original-quality sends is CPU-bound work that belongs in a native module for performance and reliable background upload (iOS Background Transfer / Android WorkManager + Foreground Service).

---

## 7. Backend architecture: Supabase-centric

**Recommendation: Supabase** — Postgres (with `pgvector`), Auth, Realtime (WebSocket subscriptions + presence), Storage (resumable/chunked uploads), and Edge Functions, as one managed project, so there's no separate backend team standing up and operating five services.

```
Client (Expo app)
   │  TLS
   ▼
Supabase Auth ── username/password sessions, device-attestation check on signup
   │
   ├── Supabase Realtime ── message delivery, presence, typing, Band events
   │        │
   │        ▼
   │   Postgres ── users, chats, bands, messages, tasks, scheduled_messages, ai_settings
   │
   ├── Supabase Storage ── original-quality file chunks, lifecycle rule = delete on expiry/one-time-download
   │
   ├── Edge Functions ── on-demand summarize/rewrite/W:AI popup actions via Claude API;
   │        │             async embedding + band_memory distillation; scheduled_messages dispatcher
   │        ▼
   │   pgvector (same Postgres) ── message embeddings for semantic search
   │
   └── Postgres Row-Level Security (RLS) ── per-user/per-chat access control, gated further by ai_settings
```

| Layer | Choice | Why |
|---|---|---|
| Auth, DB, Realtime, Storage | Supabase | One managed platform instead of five self-operated services |
| Vector search | `pgvector` (same Postgres) | No separate vector DB to operate at V1 |
| AI inference | Anthropic Claude API | Hosted, powers the W:AI popup's actions and W's assistant answers |
| Embeddings | Voyage AI | Purpose-built embeddings API, pairs cleanly with Claude |
| Access control | Postgres RLS + `ai_settings` table | Encodes both "who can read/write" and "can AI touch this chat" at the data layer |

---

## 8. Original-quality file sharing (no separate Tunnel screen)

**The internal name "Whale Tunnel" stays a backend/technical concept only — it is not a user-facing screen, icon, or branded flow.** The attach sheet has exactly the options a user expects from any messaging app: Photo, Camera, File. Nothing fourth, nothing separately branded.

**Never compressed, any size — that promise doesn't change.** What *does* change by size is the delivery mechanism, and that split is deliberate, not an inconsistency:

- **Under ~1 GB — the common case, no Tunnel involved.** A single-shot upload to Supabase Storage, stored the same way any chat attachment is, appears in the thread instantly, persists in chat history normally. No chunking, no progress bar, no expiry, no resume logic — building that machinery for a 2 MB photo is wasted complexity, and nobody expects an everyday photo to auto-delete after one download.
- **Over ~1 GB — a temporary tunnel.** This is where the full mechanism earns its cost: the client splits the file into chunks, encrypts each chunk (AES-256-GCM) with a per-file key, and uploads via Supabase Storage's resumable (TUS) multipart upload, tracking a manifest so a dropped connection resumes instead of restarting. Storage is temporary — a scheduled Edge Function deletes the chunks after download or expiry. A multi-GB transfer can genuinely fail partway through on a real connection, and a raw-footage dump isn't something people expect to live in chat forever the way a photo does — expiry, one-time-download, and resumable progress only make sense, and only appear in the UI, at this tier.

Concretely: the send preview for a small file just shows the file and a Send button — no options row at all. The send preview for a file over the threshold shows Send Original (already the default), plus optional expiry/one-time-download/password, because at that size those controls are relevant rather than clutter.

1. User taps the attachment button → picks Photo/Video/File from a normal picker.
2. Client checks file size against the ~1 GB threshold and takes the corresponding path above.
3. The recipient sees a **file message card** inline in the chat (§9), not a separate screen — file name, size, "Original Quality · Compression: None," and for large files, progress/resume state and an expiry countdown if the sender set one.
4. **Non-user recipients:** a shared file link resolves to a plain web download page even without the app installed — this remains the primary growth loop for a username-based app with no phone-contact graph (§13).

**Cost flag, unchanged:** storage + egress for large files is the largest ongoing infra line; auto-delete-on-expiry above the threshold is a cost control as much as a privacy feature — it also means the high-volume common case (small files, persisted normally) doesn't carry that same cost pattern. Model $/GB storage and egress against expected file sizes before finalizing free-tier limits, and revisit whether 1 GB is the right cut line once real usage data exists.

---

## 9. File message card — elements

Appears as a normal chat bubble, not a distinct UI system — but the elements shown depend on which tier the file fell into (§8):

**Under ~1 GB:**
```
Site_Photos.zip
340 MB
Original Quality · Compression: None
```
That's it — no progress state, no expiry, because none applies.

**Over ~1 GB:**
```
Wedding_4K.mov
4.8 GB
Original Quality · Compression: None
[progress bar or Download]
Expires in 48h
```
Download resumes if interrupted, using the same chunk-manifest approach as upload.

---

## 10. Encryption model

One mode, stated accurately: messages are **encrypted in transit (TLS) and at rest** (Supabase/cloud-provider-managed encryption), and processed server-side to power AI features. This is not end-to-end encryption, and Whale does not claim it is anywhere in-product or in marketing.

This is a separate control from **per-chat AI permission** (§11): encryption determines whether the server can technically decrypt a message to deliver/sync it; the `ai_settings` permission determines whether Whale's AI pipeline is *additionally allowed* to read that content for summarize/search/memory/rewrite. A user can leave Cloud Mode encryption as-is (required for basic delivery) while still turning AI off for a specific sensitive chat — those are two different switches, and the UI should never conflate them.

Practical safeguards, appropriately scoped for a solo/AI-built V1: Postgres RLS enforced on every table, no raw message content in AI observability/debug logs, service-role keys never shipped to the client.

Live travel location sharing (from the original Family Band vision) is deliberately pushed to V2 — continuous background location needs "Always Allow" permission, which both Apple and Google review far more strictly than any other permission, and deserves its own focused privacy review rather than riding along with the rest of Family Band.

---

## 11. Bands — feels like a group chat first

A Band opens exactly like a personal chat: messages, composer, W:AI button. What's different is available, not imposed:

- A **pinned summary card** can sit above the thread (AI-generated recap of what happened since the user last opened it) — dismissible, not permanent chrome.
- A **lightweight shortcut row** (Files · Tasks · Scheduled) sits near the composer — small pill buttons, not a persistent tab strip competing with the conversation for visual weight.
- Tapping a shortcut opens a focused overlay (Smart File View, Task View) over the current chat, not a navigation to a different part of the app.

**Data model** (unchanged from the earlier plan, still the right shape):
- `bands` — id, name, owner, member_count, type (`standard` / `family`)
- `band_members` — band_id, user_id, role
- `rooms` — band_id, name, type (chat / files / tasks / scheduled) — a Band's "main chat" is its default room
- `messages` — room_id, sender_id, content, created_at, embedding (pgvector)
- `tasks` — band_id, title, assignee, status, due_at, source_message_id (nullable — set when AI or a user converts a message into a task)
- `pinned_decisions` — band_id, message_id, pinned_by, pinned_at
- `ai_settings` — scope (`room` or `dm`), scope_id, ai_enabled, allow_summary, allow_search, allow_memory, do_not_remember

**Family Band extension**, unchanged: `bands.type = 'family'` auto-provisions Calendar/Documents/Reminders/Emergency rooms, backed by `calendar_events`, `reminders`, `emergency_contacts`, `expenses` tables.

---

## 12. W — personal AI, in two forms

**W:AI popup (inside any chat or Band)** — tapping the W icon next to the composer opens an action sheet, not a full-screen takeover:

```
Rewrite my message
Summarize this chat
Find a message
Create action items
Detect decisions
Translate
Schedule this message
Find shared files
Explain this conversation
```

Plus a free-text prompt field for anything not in the list ("find what Shareef said about biryani"). Every action:
1. Checks `ai_settings` for this chat — if AI is off or memory is disallowed for this specific action, the popup says so plainly rather than silently failing or silently ignoring the setting.
2. Runs against Claude (for generation: rewrite, summarize, explain) or the search/embedding pipeline (for retrieval: find message, find files).
3. Returns an answer with source-message references the user can tap into — never a silent action. Rewrite suggestions specifically are always shown for approval before sending, never auto-applied.

**W main screen (bottom tab)** — the assistant's home, for questions that span more than one chat:

```
Ask W anything
Suggested prompts
Recent AI searches
Unread summaries
Pending tasks
Important decisions
Memory & privacy shortcut
```

Architecture: a scheduled Edge Function periodically distills each room's recent activity into a `band_memory` table (band_id/room_id, summary_text, key_facts jsonb, updated_at) via a Claude summarization pass. A W query first retrieves relevant `band_memory` rows plus top semantic-search hits — **scoped only to rooms where `ai_settings.allow_memory` and `allow_search` are true** — then answers via Claude grounded in both. **Global search** (the search bar on the Chats screen) uses the same permission scoping: results only ever come from chats/Bands the user has allowed AI to access.

**Memory & privacy controls live inside Profile → W:AI Settings, not as a separate main screen:**

```
AI on/off (global)
Memory on/off (global)
View saved memories
Delete saved memories
Chats/Bands W:AI can access
Temporary AI mode (nothing from this session is remembered)
Do not remember this [chat] — same control, reachable from Chat Settings too
Data usage explanation
```

**Chat Settings** (opened from any chat/Band header) carries the per-chat version of the same switch, so a user doesn't have to leave the conversation to turn AI off for it specifically:

```
Chat Info · Members · Media and Files · Starred Messages
AI Mode: Off / On for this chat / Allow summaries / Allow semantic search / Allow memory / Do not remember this chat
Disappearing Messages · Privacy Settings · Clear Chat
```

---

## 13. Actions — the aggregated feed

Actions replaces a plain notifications screen with one feed of everything that needs a decision or a follow-up, ranked by recency/urgency rather than split across separate systems:

```
Actions
 ├── 2 tasks due today
 ├── 1 scheduled message pending
 ├── Family Band file expires in 4 hours
 ├── Shareef mentioned you
 └── AI Team Band has 3 unread decisions
```

This is an aggregation, not a new source of truth — it queries across existing tables (`tasks` due soon, `scheduled_messages` pending, file records nearing `expires_at`, `@mentions` parsed from recent messages, newly `pinned_decisions`) rather than duplicating that state into a separate table. Tapping an item opens the source chat/Band/file/task directly.

**Scheduled messages** live inside each chat, not as a separate main screen — entry points are a long-press on Send, a schedule icon near the composer, or asking W:AI directly ("send this tomorrow at 9am," which creates a draft the user still confirms). Data model: `scheduled_messages` — id, sender_id, target (room_id or recipient_id), body, send_at, recurrence, status. A scheduled Edge Function dispatches due messages and updates status; recurring messages re-enqueue their next occurrence on send.

---

## 14. Profile

```
Profile Info (avatar, username, QR code — no phone number)
Privacy Settings
W:AI Settings (§12)
Storage and Data
Linked Devices
Notifications
Subscription Plan
Help and Support
```

---

## 15. Go-to-market: solving cold start

No feature list changes the fact that WhatsApp/Telegram own the contact graph. Concrete mitigations for V1:

1. **Original-quality sharing as the wedge:** the recipient of a shared file feels the product's core value (a link that works even without the app, full quality, no compression) before installing anything.
2. **Niche-first, not broad consumer:** target creators trading raw video, student project teams, small studios — where the file-size pain is acute and daily.
3. **Invite-link-first onboarding:** no phone-contact-hash matching, so onboarding is built around shareable links (profile links, Band invite links, file links), not a "find people you know" step that comes up empty for a new user base.

---

## 16. Additional resources needed

### Cloud / backend
- **Supabase project** — Postgres + Auth + Realtime + Storage + Edge Functions + pgvector. Free tier for development; Pro tier (~$25+/month) once past free-tier limits.
- **Vercel** (or similar) — hosts the web download page for non-app recipients and any marketing site.
- **Domain name.**

### AI / LLM
- **Anthropic API key + billing** — powers the W:AI popup actions, W's assistant answers, agent memory distillation. Usage-based.
- **Voyage AI API key + billing** — embeddings for semantic search. Usage-based, smaller line item.

### Mobile platform
- **Apple Developer Program** — $99/year.
- **Google Play Console** — $25 one-time.
- **Expo/EAS account** — free tier for development; paid plan (~$29+/month) for production build concurrency near launch.
- **Firebase project** (Cloud Messaging only) — free, Android push delivery.

### Observability
- **Sentry** (or similar) — free tier sufficient at V1 scale; matters more here given no dedicated QA function.

### Rough monthly cost floor at V1 (order of magnitude, not a quote)
| Item | Approx. cost |
|---|---|
| Supabase Pro | $25+/month |
| Vercel | $0 (free tier) |
| Claude API usage | $100–500/month, scales with active users |
| Voyage embeddings | $10–50/month |
| Apple Developer Program | ~$8/month (billed yearly) |
| Google Play Console | $25 one-time |
| EAS | $0–29+/month |
| Sentry | $0 (free tier) |
| Domain | ~$12–20/year |
| **Approximate floor** | **~$150–300/month** before meaningful usage-driven AI/storage scaling |

---

## 17. Build phases

Ordered so each phase is a coherent, independently testable unit — feed these to the coding agent one at a time, in order.

1. **Foundations** — Supabase project setup, design-token system (§5), Expo app skeleton, username-based auth + recovery flow, CI via EAS.
2. **Core chat + Bands** — unified Chats list (pinned row, global search), 1:1 chat, Band Chat (pinned summary card + shortcut row), Family Band template, Realtime delivery, push notifications, `ai_settings` table + Chat Settings AI Mode controls.
3. **Original-quality sharing** — attach sheet (Photo/Camera/File only), Send Original toggle in send preview, chunked encrypted upload/download, resume, expiry/one-time controls, file message card, public web download page for non-app recipients.
4. **W** — W:AI popup (rewrite, summarize, find message, create task, detect decision, translate, schedule, find files, explain), semantic search + global search scoping, agent memory (`band_memory` distillation + grounded W main-screen answers), W main screen, Profile → W:AI Settings.
5. **Actions & scheduling** — `scheduled_messages` table + dispatcher, task creation (manual + AI-detected), Actions feed aggregation query, mention parsing.
6. **Hardening & store submission** — RLS policy audit (including `ai_settings` enforcement), load testing, App Store/Play Store review prep (privacy nutrition labels accurately describing Cloud Mode + AI processing), submission.
7. **V1 launch**
8. **V1.1** — Smart notification triage refinements on top of the Actions feed.

---

## 18. Business model

- **Free tier:** 5 GB original-quality limit, rate-limited AI usage (capped W:AI popup/search/rewrite calls per day) — the cap bounds Claude/Voyage inference cost, not an arbitrary restriction.
- **Premium subscription:** 15 GB limit, higher/no AI rate limits, priority support. Price point needs validation; comparable consumer subscriptions sit in the $4.99–$9.99/month range as a starting hypothesis.
- **Unit economics to model before scaling the free tier:** storage + egress per GB, Claude/Voyage inference cost per active user/month, Postgres/pgvector storage cost as message volume grows.

---

## 19. Risks & mitigations

| Risk | Mitigation |
|---|---|
| Scope creep beyond the V1 cut in §3 | Hard cut is explicit; anything not in "Included in V1" needs a deliberate roadmap revision, not an ad-hoc addition mid-build |
| Cold start / no contact graph | Original-quality-sharing-as-wedge + niche-first GTM (§15) |
| Overclaiming encryption | Cloud-only model stated accurately in-product (§10); no E2E claim anywhere |
| Conflating encryption with AI permission | Two explicit, separately-labeled switches — never one toggle implying both (§10, §12) |
| AI inference cost scaling faster than revenue | Rate-limited free tier from launch, not retrofitted after a cost spike |
| Account recovery lockout (no phone fallback) | Recovery phrase/backup codes mandatory at signup (§4) |
| App Store/Play Store review risk around AI reading messages | Accurate privacy nutrition label describing Cloud Mode + per-chat AI permission; legal review before submission |
| `ai_settings` not actually enforced in a query path (silent over-sharing) | RLS-level enforcement, not just application-layer checks; audited explicitly in phase 6 (§17) |
| Background location permission triggering heavy store scrutiny | Live location sharing excluded from V1 Family Band (§10); revisit as its own reviewed V2 feature |

---

## 20. V1 success metrics

- **Activation:** % of signups completing onboarding + sending a first message within 24h
- **Original-quality sharing adoption:** files sent/week, average file size, % of transfers using resume, % of recipients who install after receiving a web-link file (validates the growth loop in §15)
- **Bands & Family Band engagement:** % of active users in at least one Band with recent activity in a non-chat shortcut (Files/Tasks/Scheduled); % of Bands created as `type = 'family'` with a calendar event/reminder/expense logged in week one
- **W engagement:** % of weekly active users opening the W:AI popup at least once; % using the W main screen directly (not just in-chat) at least once/week
- **Agent memory quality:** % of W answers that cite more than one source message (validates synthesis, not just echoing search)
- **Actions engagement:** % of Actions items acted on (tapped through) vs. dismissed/ignored — validates the feed is surfacing the right things
- **Retention:** D1 / D7 / D30
- **Cost sanity check:** infra + AI cost per active user/month vs. the floor modeled in §16, checked monthly from launch

---

## 21. Band Categories — specialized operating layers (post-V1 vision)

**Not V1 scope.** This section documents where Bands go after V1, deliberately kept out of the initial build for the same reason §3 exists at all: five category-specific tool sets is easily 40+ new features, and building them alongside V1's core three pillars would repeat the exact "six products at once" mistake this plan has avoided from §1 onward. V1 ships one generic Band shape plus the Family Band template (§11); everything below is the roadmap once that foundation is proven.

**The idea, stated precisely:** every Band shares one core (chat, W:AI, search, original-quality sharing, Smart File View, scheduled messages, tasks, pinned decisions, AI summary — all already in V1, §11). What changes per category is not the name, it's a *specialized operating layer* — a distinct set of modules solving that category's specific pain, unlocked by the Band's type.

| Band category | Operating layer | Unique pain it solves | Distinctive modules |
|---|---|---|---|
| Family *(V1)* | Household coordination | Reminders, documents, and medical/emergency info get buried in chat | Family Reminders, Documents Vault, Health & Medical Updates, Emergency Info, Expense Split, Family Calendar, Travel Updates |
| Student/Project | Academic layer | Deadlines, notes, and task ownership scatter across a group project | Deadline Board (assignments and exams merged into one view — tracking them separately was redundant), Notes Hub, Study Session Planner, Submission Checklist |
| Creator/Media | Media production layer | Compression, messy feedback, version confusion, approval chaos | Media Vault (raw → editing → final, one versioned file record instead of three separate folders), Review & Approval (feedback and approval status merged into one), **Edit Space** (trim/caption/enhance media in-Band before finalizing), Shot List, Caption Generator, Deliverables Checklist |
| Work/Project Team | Execution/accountability layer | Decisions and owners get lost in chat scrollback | Decision Log, Action Tracker, Meeting Notes, Client Updates, Risk/Blocker Tracker, Weekly Status Summary, Approval Workflow, Project File Hub |
| Community (up to 100K) | Moderation/engagement layer | Large groups get noisy, spammy, and impossible to summarize | Announcement Channel, Topic Rooms, Q&A Board, AI Moderation, Member Roles, Polls, Resource Library, Weekly Digest, Join Approval |
| **General** | No specialized layer — the open container | A group doesn't fit, or doesn't yet need, a specialized category | Only the shared core (§11) — nothing category-specific |

Event is deliberately not a category — it didn't hold up as a distinct operating layer once RSVP/Schedule (genuinely new) is separated from vendor coordination and travel info (both just the guest-access and travel/logistics primitives Family and Work already need). If event planning becomes a priority later, it's an extension of those shared primitives, not a seventh category.

**Sub-Bands — universal, not just General.** Every category can create sub-Bands nested inside it, not only General: a Creator Band nests a sub-Band per client project, a Work Band nests a sub-Band per pod/initiative, a Student Band nests a sub-Band per subteam, a Family Band nests a sub-Band for "kids only" or "extended family." Architecturally this is a `parent_band_id` (nullable, self-referencing) on the `bands` table (§11) — one column, applies the same way regardless of category, not a per-category subsystem.

**Sub-Bands vs. Community's Topic Rooms — worth keeping distinct.** These solve similar-sounding problems but are different mechanisms. Topic Rooms (§11's `rooms` table) are lightweight channels *within* one Band — same membership, same AI settings, just organized by subject. A sub-Band is a fully separate Band entity — its own membership (a subset of the parent's, or different people entirely), its own AI settings, potentially its own category. Use rooms when the group is the same people talking about different things; use a sub-Band when it's genuinely a different, smaller group operating somewhat independently under a shared umbrella.

General's distinguishing trait is now just what it doesn't have — no category framing, no specialized modules — not the sub-Band capability, since that's available everywhere.

**Sequencing logic, not just a wishlist:** Creator/Media is the strongest next candidate after V1 — it's the category most directly aligned with the Whale Tunnel wedge already driving V1's go-to-market (§15), so a media-production layer is a natural extension of a growth motion that's already working, rather than a new one to bootstrap. Work is a close second, cheaper than it looks — most of its module list turns out to be the existing core primitives (Tasks, Pinned Decisions, AI Summary, Smart File View) with different framing, not new engineering. Community is the most structurally different (moderation, roles, scale to 100K) and should come last, once Bands' core infrastructure has been proven at normal group size — moderation-at-scale is its own hard problem and deserves a dedicated pass, not a bolt-on.

**Positioning line worth keeping verbatim:** *"Whale Bands are not groups with names — they're purpose-built group spaces where the tools change based on the reason the group exists."* This is the actual differentiator versus a WhatsApp group or Telegram supergroup, and belongs in marketing copy once these ship, not just this plan.
