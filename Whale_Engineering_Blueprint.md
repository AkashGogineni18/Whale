# Whale — Engineering Blueprint

**This is the low-level companion to `Whale_Technical_Specification.md`.** That document defines *what* to build and *why* (features, scope decisions, schema shape, screen behavior). This document defines *how*: exact folder structure, exact runnable SQL (including full RLS policies, not prose descriptions of them), exact Edge Function request/response contracts, exact package choices, exact upload/encryption implementation. Where the two overlap, this document is more specific and wins on implementation detail; the Technical Specification still wins on scope decisions (what's built vs. Coming Soon, etc).

Read this top to bottom once, then use it as a reference while building each phase from the Technical Specification's §14 build order.

---

## 1. External accounts & subscriptions — set these up before writing any code

| # | Service | URL | What you need from it | Cost | Priority |
|---|---|---|---|---|---|
| 1 | Supabase | supabase.com | New project → copy **Project URL**, **anon/public key**, **service_role key** (Settings → API), and the **database password** (set at project creation) | Free tier to start; Pro ($25/mo) once you need it | Day 1, blocking |
| 2 | Anthropic | console.anthropic.com | Create an API key (Settings → API Keys), attach a billing method (pay-as-you-go) | Usage-based, budget $20–50 for the build/test period | Day 1, blocking |
| 3 | Expo (EAS) | expo.dev | Create an account, run `eas init` in the project to link it | Free tier for development; ~$29+/mo for priority build queues closer to launch | Day 1, blocking |
| 4 | GitHub (or equivalent) | github.com | A private repo for the app code and Supabase migrations | Free | Day 1, blocking |
| 5 | Apple Developer Program | developer.apple.com/programs | Enroll as an individual — approval can take 24–48h, **start this on day 1 even though it's not needed until later** | $99/year | Day 1 (start), not blocking until TestFlight/App Store |
| 6 | Google Play Console | play.google.com/console | Register a developer account | $25 one-time | Only needed for Play Store distribution — not required for Expo Go or a sideloaded APK |
| 7 | Firebase | firebase.google.com | A project with Cloud Messaging enabled → download `google-services.json` for Android push credentials | Free (Cloud Messaging has no charge) | Needed before building Android push notifications specifically |
| 8 | Sentry | sentry.io | A project, copy the DSN | Free tier | Optional for the demo build; recommended before any real users |

**Local tooling to install** (not accounts): Node.js LTS, the Supabase CLI (`npm install -g supabase`), the EAS CLI (`npm install -g eas-cli`), and Deno (for running/testing Edge Functions locally — the Supabase CLI bundles this, a separate install isn't required).

---

## 2. Project structure

Use **Expo Router** (file-based routing) — the route tree *is* the navigation config, which removes an entire category of "navigation set up wrong" mistakes. TypeScript throughout.

```
whale/
├── app.config.ts
├── package.json
├── tsconfig.json
├── eas.json
├── .env.local                          # gitignored — EXPO_PUBLIC_* vars only
├── app/                                # Expo Router — route = file path
│   ├── _layout.tsx                     # root layout: loads session, redirects to (auth) or (tabs)
│   ├── (auth)/
│   │   ├── _layout.tsx
│   │   ├── username.tsx
│   │   ├── phone.tsx
│   │   ├── password.tsx
│   │   ├── recovery-phrase.tsx
│   │   └── login.tsx
│   ├── (tabs)/
│   │   ├── _layout.tsx                 # bottom tab bar: Chats / W / Actions / Profile
│   │   ├── chats/
│   │   │   ├── index.tsx               # Chats home (§10.3)
│   │   │   ├── search.tsx              # §10.4
│   │   │   ├── discover.tsx            # §10.3a
│   │   │   └── discover/[bandId].tsx   # public band preview
│   │   ├── w/
│   │   │   └── index.tsx               # §10.13
│   │   ├── actions/
│   │   │   └── index.tsx               # §10.14
│   │   └── profile/
│   │       ├── index.tsx
│   │       ├── privacy.tsx
│   │       └── ai-settings.tsx
│   ├── chat/
│   │   ├── dm/[dmId].tsx               # §10.5
│   │   └── band/[bandId]/
│   │       ├── _layout.tsx             # shared header for all band sub-routes
│   │       ├── index.tsx               # Band chat (§10.7)
│   │       ├── settings.tsx            # §10.15
│   │       ├── files.tsx               # Smart File View (§10.12)
│   │       ├── tasks.tsx               # Task View (§10.12)
│   │       ├── calendar.tsx            # Family only (§10.12)
│   │       ├── reminders.tsx           # Family only
│   │       ├── emergency.tsx           # Family only
│   │       ├── coming-soon.tsx         # §10.7a — accepts ?feature= param for the label
│   │       └── join-requests.tsx       # admin approval queue (§8)
│   ├── create-band.tsx                 # §10.8
│   └── +not-found.tsx
├── src/
│   ├── components/
│   │   ├── ChatRow.tsx
│   │   ├── MessageBubble.tsx
│   │   ├── Composer.tsx
│   │   ├── FileCard.tsx
│   │   ├── TaskCard.tsx
│   │   ├── BandDiscoveryCard.tsx
│   │   ├── AvatarCircle.tsx
│   │   ├── AISummaryCard.tsx
│   │   ├── WAIPopupSheet.tsx           # modal, triggered from Composer
│   │   ├── AttachSheet.tsx             # modal
│   │   ├── ScheduleSheet.tsx           # modal
│   │   └── BandOverflowSheet.tsx       # modal
│   ├── lib/
│   │   ├── supabase.ts                 # client init, singleton
│   │   ├── api/
│   │   │   ├── auth.ts
│   │   │   ├── messages.ts
│   │   │   ├── bands.ts
│   │   │   ├── discovery.ts
│   │   │   ├── files.ts
│   │   │   ├── tasks.ts
│   │   │   ├── scheduling.ts
│   │   │   ├── actions.ts
│   │   │   └── ai.ts                   # thin wrappers calling Edge Functions
│   │   ├── hooks/
│   │   │   ├── useSession.ts
│   │   │   ├── useChatsList.ts
│   │   │   ├── useMessages.ts          # paginated + realtime-subscribed
│   │   │   ├── useRealtimeChannel.ts
│   │   │   ├── useBand.ts
│   │   │   └── useAISettings.ts
│   │   └── upload/
│   │       ├── smallFileUpload.ts
│   │       └── largeFileUpload.ts      # chunking, encryption, resumable state
│   ├── store/
│   │   └── sessionStore.ts             # Zustand — current user, active theme
│   └── theme/
│       ├── tokens.ts                   # §10.1 design tokens as a typed object
│       └── icons.tsx                   # SVG icon components, ported from the sprite in Whale_V1_Screens2.html
├── supabase/
│   ├── config.toml
│   ├── migrations/
│   │   ├── 0001_core_schema.sql
│   │   ├── 0002_rls_policies.sql
│   │   ├── 0003_discovery.sql
│   │   ├── 0004_join_approval.sql
│   │   └── 0005_indexes.sql
│   ├── functions/
│   │   ├── ai-rewrite/index.ts
│   │   ├── ai-summarize/index.ts
│   │   ├── ai-find/index.ts
│   │   ├── ai-w-query/index.ts
│   │   ├── task-detect/index.ts
│   │   ├── file-sweep/index.ts
│   │   ├── scheduled-dispatch/index.ts
│   │   └── demo-reset/index.ts
│   └── seed.sql                        # §13 demo data, idempotent (safe to re-run)
└── scripts/
    └── local-review/                   # §12 test scripts, one per row in that table
```

---

## 3. Dependencies — exact package list

```json
{
  "dependencies": {
    "expo": "~52.0.0",
    "expo-router": "~4.0.0",
    "@supabase/supabase-js": "^2.45.0",
    "@tanstack/react-query": "^5.59.0",
    "zustand": "^5.0.0",
    "expo-secure-store": "~14.0.0",
    "expo-local-authentication": "~15.0.0",
    "expo-image-picker": "~16.0.0",
    "expo-document-picker": "~13.0.0",
    "expo-file-system": "~18.0.0",
    "expo-av": "~15.0.0",
    "expo-notifications": "~0.29.0",
    "expo-device": "~7.0.0",
    "react-native-quick-crypto": "^0.7.0",
    "react-native-svg": "15.8.0",
    "date-fns": "^4.1.0"
  }
}
```

`react-native-quick-crypto` is the AES-256-GCM implementation for large-tier file chunk encryption (§6) — it's WebCrypto-API-compatible and has native performance, unlike pure-JS crypto libraries which will visibly stall the UI thread on multi-hundred-MB files. Do not substitute a pure-JS AES library for this reason.

---

## 4. Environment variables

**Client-side** (`.env.local`, prefixed `EXPO_PUBLIC_` so Expo bundles them into the app — safe to expose, protected by RLS):
```
EXPO_PUBLIC_SUPABASE_URL=https://<project-ref>.supabase.co
EXPO_PUBLIC_SUPABASE_ANON_KEY=<anon-key>
```

**Server-side only** (set via `supabase secrets set KEY=value`, never in client code, never in `.env.local`):
```
ANTHROPIC_API_KEY=
SUPABASE_SERVICE_ROLE_KEY=
SUPABASE_URL=
```

---

## 5. Database — exact migrations

### `0001_core_schema.sql`
Run the full `CREATE TABLE` statements from Technical Specification §4 verbatim, in this dependency order (foreign keys require it): `profiles` → `dms` → `bands` → `band_members` → `rooms` → `messages` → `files` → `tasks` → `pinned_decisions` → `scheduled_messages` → `ai_settings` → `mentions` → `calendar_events` → `reminders` → `emergency_contacts` → `expenses` → `band_reports` → `band_join_requests`.

### `0002_rls_policies.sql`
```sql
-- profiles
alter table profiles enable row level security;
create policy "profiles readable by any authenticated user" on profiles
  for select using (auth.role() = 'authenticated');
create policy "users update own profile" on profiles
  for update using (auth.uid() = id);
create policy "users insert own profile on signup" on profiles
  for insert with check (auth.uid() = id);

-- dms
alter table dms enable row level security;
create policy "dms readable by participants" on dms
  for select using (auth.uid() = user_a_id or auth.uid() = user_b_id);
create policy "users create dms they participate in" on dms
  for insert with check (auth.uid() = user_a_id or auth.uid() = user_b_id);

-- bands (includes the discovery carve-out, §8a)
alter table bands enable row level security;
create policy "bands readable by members or if public" on bands
  for select using (
    exists (select 1 from band_members bm where bm.band_id = bands.id and bm.user_id = auth.uid())
    or is_public = true
  );
create policy "authenticated users create bands" on bands
  for insert with check (auth.uid() = owner_id);
create policy "owner or admin updates band" on bands
  for update using (
    auth.uid() = owner_id
    or exists (select 1 from band_members bm where bm.band_id = bands.id and bm.user_id = auth.uid() and bm.role in ('owner','admin'))
  );

-- band_members
alter table band_members enable row level security;
create policy "members readable by co-members" on band_members
  for select using (
    exists (select 1 from band_members bm2 where bm2.band_id = band_members.band_id and bm2.user_id = auth.uid())
  );
create policy "users insert own membership (direct join)" on band_members
  for insert with check (auth.uid() = user_id);
create policy "admins remove members" on band_members
  for delete using (
    exists (select 1 from band_members bm2 where bm2.band_id = band_members.band_id and bm2.user_id = auth.uid() and bm2.role in ('owner','admin'))
    or auth.uid() = user_id  -- users can remove themselves (leave band)
  );

-- rooms
alter table rooms enable row level security;
create policy "rooms readable by band members" on rooms
  for select using (
    exists (select 1 from band_members bm where bm.band_id = rooms.band_id and bm.user_id = auth.uid())
  );
create policy "admins create rooms" on rooms
  for insert with check (
    exists (select 1 from band_members bm where bm.band_id = rooms.band_id and bm.user_id = auth.uid() and bm.role in ('owner','admin'))
  );

-- messages
alter table messages enable row level security;
create policy "messages readable by room/dm participants" on messages
  for select using (
    (room_id is not null and exists (
      select 1 from rooms r join band_members bm on bm.band_id = r.band_id
      where r.id = messages.room_id and bm.user_id = auth.uid()
    ))
    or
    (dm_id is not null and exists (
      select 1 from dms d where d.id = messages.dm_id and (d.user_a_id = auth.uid() or d.user_b_id = auth.uid())
    ))
  );
create policy "participants send messages" on messages
  for insert with check (
    auth.uid() = sender_id and (
      (room_id is not null and exists (
        select 1 from rooms r join band_members bm on bm.band_id = r.band_id
        where r.id = messages.room_id and bm.user_id = auth.uid()
      ))
      or
      (dm_id is not null and exists (
        select 1 from dms d where d.id = messages.dm_id and (d.user_a_id = auth.uid() or d.user_b_id = auth.uid())
      ))
    )
  );
-- announcements room type: only owner/admin may insert (Technical Spec §8a's Community handling)
create policy "only admins post in announcement rooms" on messages
  for insert with check (
    not exists (select 1 from rooms r where r.id = messages.room_id and r.type = 'announcements')
    or exists (
      select 1 from rooms r join band_members bm on bm.band_id = r.band_id
      where r.id = messages.room_id and bm.user_id = auth.uid() and bm.role in ('owner','admin')
    )
  );

-- files, tasks, pinned_decisions, scheduled_messages, mentions, calendar_events, reminders, emergency_contacts, expenses:
-- same shape — readable/writable only by band_members of the owning band_id. Apply this identical pattern to each:
create policy "band-scoped select" on files for select using (
  exists (select 1 from band_members bm join messages m on m.id = files.message_id
          where bm.band_id = (select band_id from rooms where id = m.room_id) and bm.user_id = auth.uid())
  or exists (select 1 from dms d join messages m on m.id = files.message_id
             where m.dm_id = d.id and (d.user_a_id = auth.uid() or d.user_b_id = auth.uid()))
);
-- (repeat the equivalent band_id-scoped select/insert policy for tasks, pinned_decisions, calendar_events, reminders, emergency_contacts, expenses — each keyed on band_id directly, simpler than files' join-through-messages)
create policy "band-scoped select" on tasks for select using (
  exists (select 1 from band_members bm where bm.band_id = tasks.band_id and bm.user_id = auth.uid())
);
create policy "band-scoped insert" on tasks for insert with check (
  exists (select 1 from band_members bm where bm.band_id = tasks.band_id and bm.user_id = auth.uid())
);
-- repeat this exact two-policy pattern (band-scoped select + band-scoped insert, keyed on band_id) for:
-- pinned_decisions, calendar_events, reminders, emergency_contacts, expenses

-- ai_settings
alter table ai_settings enable row level security;
create policy "ai_settings readable/writable by scope participants" on ai_settings
  for all using (
    (scope = 'room' and exists (
      select 1 from rooms r join band_members bm on bm.band_id = r.band_id
      where r.id = ai_settings.scope_id and bm.user_id = auth.uid()
    ))
    or
    (scope = 'dm' and exists (
      select 1 from dms d where d.id = ai_settings.scope_id and (d.user_a_id = auth.uid() or d.user_b_id = auth.uid())
    ))
  );

-- band_reports
alter table band_reports enable row level security;
create policy "any authenticated user can file a report" on band_reports
  for insert with check (auth.uid() = reported_by);
-- no select policy for regular users — reports are for founder/service-role review only

-- band_join_requests
alter table band_join_requests enable row level security;
create policy "users see own requests, admins see all for their band" on band_join_requests
  for select using (
    auth.uid() = user_id
    or exists (select 1 from band_members bm where bm.band_id = band_join_requests.band_id and bm.user_id = auth.uid() and bm.role in ('owner','admin'))
  );
create policy "users create own join request" on band_join_requests
  for insert with check (auth.uid() = user_id);
create policy "admins update request status" on band_join_requests
  for update using (
    exists (select 1 from band_members bm where bm.band_id = band_join_requests.band_id and bm.user_id = auth.uid() and bm.role in ('owner','admin'))
  );
```

### `0003_discovery.sql`
The `bands.is_public`/`discovery_category`/`location_label`/`tagline` columns and the public-read carve-out are already included in `0001`/`0002` above if you're building fresh — this migration file exists as a marker for teams adding discovery to an *already-deployed* schema (`ALTER TABLE bands ADD COLUMN ...` for each).

### `0005_indexes.sql`
```sql
create index idx_messages_room_created on messages (room_id, created_at desc);
create index idx_messages_dm_created on messages (dm_id, created_at desc);
create index idx_tasks_band_status on tasks (band_id, status);
create index idx_tasks_assignee_status on tasks (assignee_id, status) where status != 'done';
create index idx_bands_public_category on bands (discovery_category) where is_public = true;
create index idx_scheduled_pending on scheduled_messages (send_at) where status = 'pending';
create index idx_files_expiring on files (expires_at) where deleted_at is null;
```
These four are load-bearing for the 1,000-user non-functional requirement (Technical Spec §1) — without `idx_messages_room_created` specifically, the Chats list and any message-history fetch degrades badly well before 1,000 users. Do not skip this file.

---

## 6. Edge Function contracts — exact request/response shape

All Edge Functions: `POST` only, `Authorization: Bearer <user JWT>` header required (Supabase validates this automatically and exposes `auth.uid()` inside the function via the Supabase client initialized with that header — do not manually parse the JWT). Every function checks `ai_settings` before touching message content, per Technical Spec §7's opening paragraph — implement this check as a shared helper (`src/lib/checkAiAllowed.ts` equivalent inside `supabase/functions/_shared/`), not copy-pasted per function.

### `ai-rewrite`
```
Request:  { "draftText": string, "tone": "professional" | "friendlier" | "shorter" | "translate", "targetLanguage"?: string }
Response: { "rewrittenText": string }
Errors:   400 if draftText is empty; 200 with a passthrough of draftText if the underlying Claude call fails (never block sending a message because rewrite failed)
```

### `ai-summarize`
```
Request:  { "scope": "room" | "dm", "scopeId": string, "windowHours"?: number }  // default windowHours = since last read, capped at 168 (7 days)
Response: { "bullets": string[], "aiDisabled": boolean }  // aiDisabled=true + bullets=[] if ai_settings blocks this scope
```

### `ai-find`
```
Request:  { "query": string, "scope": "room" | "dm" | "global", "scopeId"?: string }  // scopeId required unless scope="global"
Response: { "matches": [{ "sourceLabel": string, "snippet": string, "approxTime": string }] }
```

### `ai-w-query`
```
Request:  { "query": string }
Response: { "answer": string, "citations": [{ "label": string, "timeframe": string }], "followUps": string[] }
```

### `task-detect`
```
Request:  { "messageId": string }
Response: { "isActionable": boolean }
```
Called async, fire-and-forget, immediately after a message insert in a Band room (not DMs) — the client should not await this before showing the message as sent.

### `file-sweep`, `scheduled-dispatch`, `demo-reset`
No client-facing contract — these are `pg_cron`-invoked, not called from the app. Configure via `supabase/config.toml`:
```toml
[functions.file-sweep]
schedule = "*/15 * * * *"

[functions.scheduled-dispatch]
schedule = "* * * * *"

[functions.demo-reset]
schedule = "0 */6 * * *"
```

---

## 7. Original-quality file sharing — exact implementation

**Small tier** (`src/lib/upload/smallFileUpload.ts`):
```
1. supabase.storage.from('files').upload(`${bandOrDmId}/${fileId}-${filename}`, fileBlob)
2. insert into `files` (tier='small', no encryption_key)
3. insert a `messages` row referencing it
```
One function, no chunking, no progress UI — matches Technical Spec §6's "no options screen" requirement exactly.

**Large tier** (`src/lib/upload/largeFileUpload.ts`):
```
1. Generate a per-file AES-256-GCM key client-side (react-native-quick-crypto).
2. Split file into 8MB chunks via expo-file-system's readAsStringAsync with position/length, or better, expo-file-system's newer chunked-read API if available in the SDK version pinned above.
3. Encrypt each chunk with the per-file key before upload.
4. Upload via Supabase Storage's TUS-resumable endpoint (@supabase/storage-js exposes this — use `createSignedUploadUrl` + the TUS client, not the plain `.upload()` call, which is not resumable).
5. Persist the chunk manifest (array of completed chunk indices) to AsyncStorage/SecureStore keyed by fileId, so a killed app resumes correctly on relaunch by checking which chunks are already marked complete before re-uploading anything.
6. On completion: insert `files` row (tier='large', encryption_key stored — protected by RLS per §5's file-scoped policy), insert `messages` row.
7. Show upload progress by tracking (completed chunks / total chunks) — update UI on every chunk completion, not just at start/end.
```

---

## 8. Screen implementation pattern

Rather than hand-specifying JSX for all ~25 screens, every screen follows one of four patterns below. Match visual details exactly against `Whale_V1_Screens2.html` (colors, spacing, type from Technical Spec §10.1); match data/interaction behavior to the pattern for its type.

### Pattern A — List screen (Chats home, Discover, Smart File View, Task View, Actions, Search results)
```tsx
export default function ScreenName() {
  const { data, isLoading } = useQuery({ queryKey: [...], queryFn: () => api.fetchX(...) });
  useRealtimeChannel(channelName, () => queryClient.invalidateQueries([...]));  // where realtime applies (Chats, Band chat)
  return (
    <Screen>
      <Header title="..." rightIcons={[...]} />
      {isLoading ? <ListSkeleton /> : <FlashList data={data} renderItem={...} />}
    </Screen>
  );
}
```

### Pattern B — Thread screen (Personal chat, Band chat)
```tsx
export default function ThreadScreen() {
  const { messages, sendMessage } = useMessages(scopeType, scopeId);  // realtime-subscribed inside the hook
  const [draft, setDraft] = useState('');
  return (
    <Screen>
      <ThreadHeader ... />
      {isBand && <AISummaryCard bandId={bandId} />}
      <MessageList messages={messages} />
      {isBand && <ShortcutChipRow bandType={band.type} bandId={bandId} />}
      <Composer value={draft} onChangeText={setDraft} onSend={() => sendMessage(draft)} onOpenWPopup={...} onOpenAttach={...} />
    </Screen>
  );
}
```

### Pattern C — Modal/sheet (W:AI Popup, Attach Sheet, Schedule Sheet, Band Overflow)
Implemented as an Expo Router modal group (`presentation: 'transparentModal'` in the relevant `_layout.tsx`) rendering over a dimmed/blurred snapshot of the screen behind it, matching the `.scrim` + `.sheet` pattern in the mockup CSS exactly. Each action row is a simple `Pressable` calling its corresponding `api/ai.ts` function and closing the sheet on completion.

### Pattern D — Form screen (Onboarding steps, Create Band)
Controlled inputs, local component state, a single submit handler calling the relevant `api/` function, `router.push()` to the next step on success. One field group visible per screen (Technical Spec §10.2) — do not combine steps.

---

## 9. Screen manifest — route → file → data → key interactions

| Screen | Route | Data hooks | Key interactions |
|---|---|---|---|
| Chats home | `(tabs)/chats/index` | `useChatsList()` | tap row → `chat/dm/[id]` or `chat/band/[id]`; tap compass → `chats/discover`; tap pencil → new chat flow; tap search card → `chats/search` |
| Discover | `(tabs)/chats/discover` | `useDiscoverBands(category, query)` | tap card → `chats/discover/[bandId]` |
| Public band preview | `chats/discover/[bandId]` | `useBandPreview(bandId)` | Join → `api.bands.join()`; Request to Join → `api.bands.requestJoin()`; Report → `api.bands.report()` |
| Personal chat | `chat/dm/[dmId]` | `useMessages('dm', dmId)` | Pattern B |
| Band chat | `chat/band/[bandId]/index` | `useMessages('room', chatRoomId)`, `useBand(bandId)` | Pattern B; shortcut chips route to `files`/`tasks`/`calendar`/`coming-soon` per §8's mapping |
| W tab | `(tabs)/w/index` | `useWQuery()` (fires only on submit, not on mount) | submit query → renders `.w-answer` state in place |
| Actions | `(tabs)/actions/index` | `useActionsFeed()` | tap item → navigates to source (task's band, file's band, etc.) |
| Create Band | `create-band` | none (local form state) | submit → `api.bands.create()` → `router.replace('chat/band/[newId]')` |
| Coming Soon | `chat/band/[bandId]/coming-soon?feature=X` | none | back only |
| Join requests (admin) | `chat/band/[bandId]/join-requests` | `useJoinRequests(bandId)` | Approve/Deny → `api.bands.decideJoinRequest()` |

---

## 10. Testing strategy

- `scripts/local-review/` — one script per row in Technical Spec §12, written as plain Node/TS scripts using `@supabase/supabase-js` against either the local Supabase instance (`supabase start`) or a staging project. Run all of them (`npm run review:local`) before any Expo Go device testing begins, per Technical Spec §12's explicit ordering requirement.
- No React Native component test suite is required for the demo build — prioritize the local review pass and manual device testing over unit-testing UI components this week; revisit post-demo.

---

## 11. Build & deployment

`eas.json`:
```json
{
  "build": {
    "development": { "developmentClient": true, "distribution": "internal" },
    "preview": { "distribution": "internal" },
    "production": {}
  }
}
```
For the demo: `eas update --branch preview` publishes an over-the-air update reachable via Expo Go on any device with the project's QR/link — this is the "share the app with YC" mechanism from earlier in this process, not a store submission.
