# Whale — YC Demo Sprint (7 Days, Solo + Claude Code)

> Companion to `Whale_V1_Implementation_Plan.md` — that doc describes the fuller V1 (Actions, Family Band, per-chat AI permissions, agent memory). This doc is a deliberately smaller cut of it for one goal only: a working app on real phones, filmable, in 7 days, by one person. Backend stack, encryption approach, and Tunnel technical design from the main plan still apply — only the *feature scope and timeline* are compressed here.

---

## Reality check, stated plainly

Real-time chat + AI + resumable large-file transfer + two mobile platforms, solo, in 7 days, is genuinely aggressive — even with Claude Code accelerating the coding itself. It's achievable **only** if you cut everything not on the direct path to "AI + Uncompressed + Bands working live on a phone," and lean entirely on managed services instead of building any backend infrastructure by hand. Every cut below exists to protect that one week — none of it means the fuller plan is wrong, it means this week has one job.

---

## LLM provider: Claude, not a self-hosted open-source model — for this week specifically

You asked directly, so the direct answer: **keep Claude for the demo build.** Not for brand loyalty — because this is the one week where the cost of being wrong about it is highest.

- Self-hosting an open-source model (Llama 3.1, Mixtral, etc.) well enough to look polished on camera is its own infrastructure project — GPU provisioning, an inference server, latency tuning — real engineering time spent on something that isn't one of your three mandatory pillars, in the week with zero slack.
- It's not obviously cheaper this week either. A rented GPU costs money whether it's idle or not; a week of Claude API usage at demo scale is realistically $20-50.
- The core risk is quality: "AI helps you chat smarter" has to look obviously true in a 60-second clip. This isn't the week to gamble your best demo moment on a model you haven't had time to properly evaluate against Claude for rewrite/summarize/search quality.
- It doesn't buy the differentiation it sounds like it would — whichever model you use, you'd almost certainly call a hosted inference provider (Together AI, Fireworks, Groq) rather than truly self-host in a week, so the trust/privacy story doesn't actually change.

**Nothing about using Claude locks you in.** The integration is a plain text-in/text-out API call — swapping providers later, once there's time to properly evaluate quality and cost at real scale, is cheap. If the open-source direction matters for your long-term story, that's a strong post-YC roadmap item, not a Day 1 decision this week.

---

## Platform decision: build once, demo on both — skip the app stores entirely

**Use Expo (React Native).** One codebase targets both iOS and Android simultaneously — this isn't really an "either/or" feasibility question once you're on Expo, it's the same code either way.

**For the demo, don't submit to the App Store or Play Store at all.** That review process (Apple especially) can take days you don't have, and YC doesn't need a store listing — they need to see it work.

- **Fastest path:** run the app live via **Expo Go** on a real iPhone and a real Android phone. Expo Go loads your app instantly from a dev/preview build with no store review, no waiting. This is what you film.
- **If you want YC to install it themselves** (stronger than a video alone): use **EAS Update** to publish a shareable preview link that opens directly in Expo Go on their own phones — still no store review, available same-day.
- **Optional parallel track:** enroll in the Apple Developer Program ($99/year) *today, regardless* — individual account approval sometimes takes 24-48 hours, so starting it now costs nothing and keeps a TestFlight option open later in the week if you finish early. Don't plan around it being ready in time; treat it as a bonus if it lands.
- **Android APK:** if you want a direct-install fallback, `eas build --platform android` produces an `.apk` you can just send as a file — no Play Console account needed for that.

**If forced to harden one platform more than the other for the video:** Android. No App Store-style account approval blocking you, and sideloading an APK is instant, so it's the lower-friction platform to have as your guaranteed fallback if iOS setup hits a snag mid-week.

---

## How YC actually experiences the link — this is the real problem you're flagging

You're right, and it's worth designing for explicitly: a reviewer who signs up fresh lands in an empty inbox with nobody to message and nothing to click. A chat app with zero content looks broken or pointless even if every feature works perfectly. Two things fix this, and the video matters more than either.

**First, recalibrate what the link is for.** YC partners review at high volume with a few minutes per application. Most will watch your video as the primary evidence — the live link is there to prove the video isn't vaporware, not to be feature-tested end to end by a stranger. Don't over-invest in making the live link a bulletproof multi-user experience; do make sure it never looks empty if someone opens it.

**Second, ship one seeded demo account — never a blank signup.** Create a single login (e.g. `demo` / a simple password) pre-populated before you record anything:
- 2-3 realistic Bands across categories — the Family Band with real calendar events/reminders/emergency contacts already in it, a Work Band with decisions and tasks already logged
- Message history with varied, realistic timestamps, not everything "just now"
- At least one file already sitting in a Band via original-quality sharing (genuinely large, a few hundred MB or more) so a reviewer can tap Download and watch a real resumable, uncompressed transfer happen without you doing anything
- A W:AI summary already generated and visible, so the payoff is instant instead of an empty "ask me anything" box waiting for a first question

Put this login directly in your YC submission text and in the video description — zero friction, no signup flow required to explore it.

**Optional, stronger: a second test login for real interaction.** If you want a partner to verify real-time messaging or the W:AI popup themselves rather than just browse seeded data, provide two logins (`test1` / `test2`) with one line of instruction: log into one on their phone, the other on a second device or browser tab, and message between them. This lets them confirm it's not staged without needing you online to respond live — don't design around "message me and I'll reply," since you can't guarantee you're awake when someone tries it at 2am your time.

**One real friction point: Expo Go itself.** Since you're skipping the App Stores this week, opening the link requires installing the free Expo Go app first — an extra step a busy reviewer might not bother with. State this explicitly rather than assuming it's obvious: *"1) Install Expo Go (free, App Store/Play Store) 2) Open this link on your phone 3) Log in with `demo` / [password]."* This friction is exactly why the video stays your primary artifact — it's guaranteed to work precisely as intended; the live link is upside for whoever chooses to try it.

---

## What's IN this build — the only things that ship

| Feature | Why it's in |
|---|---|
| Username + password + **phone number** signup | Your explicit requirement this round — see note below on scope |
| Unified Chats list: 1:1 chats + Bands together | Minimum viable version of the core IA |
| Personal chat + Band chat (real-time, via Supabase Realtime) | Has to actually work live for the video |
| **Bands, all types** — category selector (General, Family, Student, Creator, Work, Community); **Family Band ships with real working modules** (Calendar, Reminders, Emergency Contacts), other categories get icon/framing only | Now in scope — see §"Band categories" below for the two-tier approach that keeps this feasible in a week |
| **W:AI in-chat popup** — rewrite-before-send, summarize this chat, find a message | The mandatory AI pillar, the 3 highest-impact-on-camera actions |
| **Standalone W tab + memory** — "ask W anything" across your Bands, computed on demand (not a cron job — see below) | Added back per your request, simplified for time |
| **Actions tab** — aggregated feed of tasks, scheduled messages, mentions, decisions | Added back per your request |
| **Scheduled messages** — schedule from the composer, dispatched by a scheduled Edge Function | Added back per your request |
| **Original-quality file sharing** — chunked upload with visible resume/progress, expiry + one-time-download toggles | The mandatory uncompressed pillar; the resumable progress bar is your best "wow" shot |
| Minimal Profile (avatar, username, logout) | Just enough to look like a real account system |

## What's still OUT — cut for time, not because they're wrong ideas

| Cut | Why | Where it lives in the fuller plan |
|---|---|---|
| Full category-specific module sets (Raw Footage Vault, Assignment Tracker, Decision Log, etc. — the 7-9 unique modules per category from the Band Categories vision) | Category *selection* is in this week; the deep unique tooling per category is 40+ features, genuinely a post-YC build | Main plan §21 |
| Per-chat granular AI permissions | One global AI on/off is enough for a demo; the nuanced version is a trust feature for real users, not judges | Main plan §12 |
| Push notifications | Not visible in a recorded demo | — |
| Public web download page for non-app recipients | A growth feature, not a demo feature | Main plan §8 |
| Real SMS OTP verification on signup | See note below | — |
| Sentry, custom domain | Operational polish, not demo-critical | — |

**On the phone number requirement:** noted and included as a required signup field, stored on the account. What I'd cut for time specifically is *verifying* it via a real SMS OTP (that means integrating Twilio Verify, handling delivery failures, rate limits — real engineering time for something a judge never sees). Collect it, store it, don't verify it this week. If you want real verification, it's a genuinely small add (~2-3 hours) — see budget note below.

---

## Band categories: framing for most, real working features for Family

Building the full unique module set for every category (§21 of the main plan — Raw Footage Vault, Assignment Tracker, Decision Log, and so on, 7-9 modules × 7 categories) is a 40+ feature build on its own. That's not a 1-week cut. But you're right that a category picker where every category just relabels the same generic screens isn't a real demo of the idea either — tapping into a Band needs to actually show that category's tools, not just a different color.

**Two tiers this week, not one:**

**Tier 1 — General, Student, Creator, Work, Community: framing only.** At Band creation the user picks a category; it gets its own icon/accent tint, and the AI summary + W:AI popup are prompted differently per category (a Work Band's summary emphasizes decisions, a Creator Band's emphasizes file/version activity) — a config change, not new screens. This still gets you a Band creation flow that visibly represents the full category vision, honestly labeled as "framing" rather than pretending it's more than it is.

**Tier 2 — Family: real, working, category-specific features.** Opening a Family Band shows genuinely different tools, not relabeled generic ones:

| Family module | What it actually does | Backing table |
|---|---|---|
| **Calendar** | List of upcoming family events, add one from the Band | `calendar_events` |
| **Reminders** | Medication/birthday/general reminders, add and mark done | `reminders` |
| **Emergency Contacts** | Name, phone, relationship — a real list, not a mock | `emergency_contacts` |
| Expense Split *(stretch)* | Log a shared cost, split across members | `expenses` — cut first if time runs short |

This matches the Family Band design already worked out in the main plan (§11) — the tables and screen shapes aren't new invention, just built for real this time instead of only described. Opening a Family Band's shortcut chips (Calendar · Reminders · Emergency) takes you to these actual working screens; opening any other category's chips takes you to the shared generic components. That asymmetry is fine to show as-is — "here's our first fully-built category, and here's the framework the rest plug into" is a coherent, honest story for a judge, arguably a *better* one than pretending everything is equally finished.

**General** stays the default/no-special-framing option for Tier 1 categories, and is also your fallback if a category-specific prompt ever produces something odd on camera.

---

## Standalone W tab: memory without a cron job

Same simplification logic as the search cut below, extended to cover the full W tab: instead of a scheduled Edge Function that periodically distills every Band into a `band_memory` table (real background-job infrastructure — cron scheduling, failure handling, staleness tracking), compute the answer **on demand** when the user opens W or asks a question. Pull recent messages across the user's Bands (scoped to what AI is allowed to see) as context, send straight to Claude with the question, return a grounded answer with source references.

This is architecturally simpler (no scheduled job, no separate memory table to keep fresh, no staleness bugs to debug under deadline) and, for a demo, indistinguishable from the "real" version on camera. The tradeoff — it re-reads message history on every query rather than working off a pre-computed summary — doesn't matter at 1000-user/demo message volumes. Build the real distillation pipeline post-YC once there's time to get the scheduling and staleness handling right.

---

## Simplification that buys you the most time: search without a vector database

The fuller plan's semantic search uses async embeddings (Voyage AI) written to `pgvector` and queried at search time — the right architecture for real scale, but it's a whole extra pipeline (embedding job, vector queries, ranking) to build and debug in a week you don't have to spare.

**For this week:** implement "find a message" and "summarize this chat" as direct Claude API calls over a Band's recent message history at query time — no embeddings, no vector DB, no background job. Send the last N messages (or all of them, at demo scale) as context with the user's question, let Claude answer directly. This is *less* scalable (fine — you're demoing at Band-sized conversations, not millions of messages) but it's dramatically faster to build correctly, and it looks identical on camera. Revisit real embeddings once you're past this week.

This also means **you don't need a Voyage AI account at all this week** — one less service, one less cost line.

---

## 1,000-user capacity: not actually a concern this week

Supabase's free tier comfortably handles low thousands of users for Auth, Postgres, and Realtime connections at the traffic level a pre-launch demo app will see. This isn't something you need to engineer for specifically — just don't write obviously unbounded queries (e.g., fetching every message in a Band with no limit), which Claude Code should default to avoiding anyway. Revisit capacity planning after YC, when real usage patterns exist.

---

## Day-by-day

This is now a full week with no buffer built in. If you fall behind, cut in this order: Expense Split first (of the Family modules), then scheduled messages, then Actions tab, then category-specific AI framing for the Tier 1 categories (fall back to "General" framing for all of them). The three mandatory pillars, Family Band's Calendar/Reminders/Emergency, and the standalone W tab are the non-negotiable floor — that combination is your actual demo.

| Day | Focus |
|---|---|
| 1 | Supabase project, Expo app skeleton, auth (username/password + phone field, no OTP), core data model (users, bands incl. `type` column, band_members, rooms, messages, tasks, scheduled_messages, calendar_events, reminders, emergency_contacts), Realtime wired end-to-end |
| 2 | Chats list UI + Personal chat + Band chat UI + Band creation flow with category selector (icon/tint/framing per category); seed a couple of demo Bands across different categories with realistic content |
| 3 | Family Band's real modules — Calendar (list + add event), Reminders (list + add + mark done), Emergency Contacts (list + add); wire the Family shortcut chips to these instead of the shared generic components |
| 4 | W:AI popup — rewrite-before-send, summarize this chat, find a message (direct Claude calls, category-aware prompts) |
| 5 | Standalone W tab (on-demand cross-Band query) + Actions tab (aggregation over tasks/scheduled/mentions/decisions) |
| 6 | Original-quality file sharing — chunked encrypted upload with a real progress bar, resume, download, expiry/one-time toggles, file message card in the thread |
| 7 | Scheduled messages if time allows, visual polish pass, **seed the `demo` account with realistic Bands/history/files** (per the YC-link section above), test on a real iPhone and Android phone via Expo Go, rehearse, record the video |

Day 7 is doing the most duty here — polish, device testing, and recording all land on the last day. If Days 1-6 slip at all, the fallback cut list above is what protects the video getting made rather than protecting feature completeness. Original-quality file sharing (Day 6) is deliberately late in the sequence *only* because it's the most self-contained piece to build and test in isolation right before recording — not because it matters less; it's still one of your three mandatory pillars and should be rock solid before Day 7.

---

## Budget: what to actually buy this week

| Item | Cost | Buy now? |
|---|---|---|
| Anthropic API key + billing (powers W:AI in the app — **separate from your Claude Code subscription**, which is your dev tool, not the app's runtime AI) | Usage-based, budget ~$20–50 for a week of building + testing | **Yes, today** — nothing works without it |
| Supabase project | Free tier | **Yes, today** — free, just create it |
| Expo/EAS account | Free tier | **Yes, today** — free |
| Apple Developer Program | $99/year | Optional, start enrollment today if you want a TestFlight fallback — don't depend on it |
| Google Play Console | $25 one-time | Skip — APK sharing works without it |
| Voyage AI (embeddings) | — | **Skip entirely** — cut per the search simplification above |
| Domain / Vercel | — | Skip — the web download page is cut this week |
| Twilio (real phone OTP) | ~$0.05–0.10/verification + small monthly | Optional, only if you decide real verification is worth ~2-3 hours this week |
| Sentry | — | Skip |

**Realistic total to spend before you start building: under $150**, and most of that is the optional Apple Developer Program — the actual required spend is just Claude API usage, which is pay-as-you-go and likely $20-50 for the week.

Claude Code itself (your dev tool) is a separate cost you already have — don't confuse it with the Anthropic API key the *app* needs at runtime; they're billed separately even though both go through Anthropic.
