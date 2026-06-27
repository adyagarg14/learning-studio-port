# Porting `learning-studio-app` → `learning-studio-port`

The deployed bundle is single-file: `reference/deployed-index.html` (734 lines, ~38 KB).
The bundled app is **vanilla DOM** — no React in production. The new project is
**React 19 + TanStack Query + `lemma-sdk/react`** (per the bundled starter's AGENTS.md),
so every imperative DOM section maps to a component or hook.

> The original source archive isn't retained on the platform. Treat the
> deployed bundle as the only authoritative reference. Section line numbers
> below are against `reference/deployed-index.html`.

## Top-level scaffolding map

| Deployed JS concept | Lines | React / SDK replacement |
|---|---|---|
| `boot()` + `<script src=…/sdk/lemma-client.js>` | 169–184 | `AuthGuard` + `QueryClientProvider` in `src/main.tsx` |
| `el(html)` template helper | 187 | React JSX |
| `fmt(iso)`, `todayISO()` | 188–189 | `src/lib/format.ts` |
| `weaknessPill(w)` | 190 | `src/components/Pill.tsx` |
| `initNav()` + `switchTab()` + `#nav button[data-tab]` | 196–204, 196–213 | React Router `NavLink` (or single `useState<tab>`) |
| Tab registry `renderers = { dashboard, decks, srs, quiz, plan, coach }` | 205–212 | `<Route>` per view |
| `liveWires()` / `client.datastore.watchChanges(...)` | 214–224 | `useLiveRecords` per active view (auto-merges deltas) |
| `showModal(content)` + `close()` | 226–237 | `<Modal open=… onClose=…>` |
| `escapeHtml(s)` | 731 | React's default escaping (delete) |

The Tabs enumerated in the deployed bundle:

1. **dashboard** → `views/Dashboard.tsx` (line 240)
2. **decks** → `views/Decks.tsx` (322)
3. **srs** → `views/SRS.tsx` (445)
4. **quiz** → `views/Quiz.tsx` (508)
5. **plan** → `views/Plan.tsx` (597)
6. **coach** → `views/Coach.tsx` (693)

## Per-view porting

### `views/Dashboard.tsx` (deployed: `loadDashboard`, 240–315)
**Reads**
- `client.records.list("subjects", { limit: 200 })`
- `client.records.list("topics", { limit: 500 })`
- `client.records.list("plan_items", { limit: 200, filter: [{field:"date", op:"eq", value: today}] })`
- `client.functions.run("compute_weakness", { top_n: 8 })`

**Mutations**
- `client.records.update("plan_items", it.id, { status: v, completed_at: v === "done" ? new Date()… })`

**SDK equivalent**
- 3× `useRecordList(lemmaClient, "subjects"|"topics"|"plan_items", { limit, filter, sort })`
- `useFunctionRun(lemmaClient, "compute_weakness", { top_n: 8 })` (or a dedicated `functionHooks.computeWeakness`)
- `useRecordUpdate("plan_items", { onSuccess: invalidates ["plan_items"] })`

**Fragment-by-fragment**
| Deployed section | Component |
|---|---|
| 256–257: `queueItems` / pendingCount | `<TodayQueue items={pendingToday} />` |
| 268–283: `drivers` (motivation line, date) | `<DashboardHeader today={todayISO()} />` |
| 309–315: row with `<select data-status=…>` + `update` | `<PlanRow item topic={…} onChange={…} />` |

### `views/Decks.tsx` (deployed: `loadDecks`, 322–443)
**Reads**: subjects, topics, materials.

**Mutations** (modals):
- L369: create subject
- L389: create topic under selected subject
- L431: upload file → `client.files.upload(f, path)` then create material

**SDK equivalent**: `useRecordList` ×3, `useRecordCreate` per table, and the file-upload uses `lemmaClient.files.upload` (raw call). Make `<NewMaterialModal>` use a `<form>` + `useRecordCreate`.

### `views/SRS.tsx` (deployed: `loadSRS`, 445–506)
**Reads**: `srs_cards` (sorted by `due_date asc`), `topics`.
**Mutations**:
- L476: `client.functions.run("sm2_update", { card_id: c.id, quality: g })` (writes back to `srs_cards`).
- L498: `client.records.create("srs_cards", …)` for new cards.

**SDK equivalent**: dedicated hook `useSm2Update()` with `useFunctionRun`; surface the grade as 0–5 buttons.

### `views/Quiz.tsx` (deployed: `loadQuiz`, 508–595)
**Reads**: `quizzes` (sorted by taken_at desc), `subjects`, `topics`, `quiz_responses`.
**Mutations** (L561, L578, L591):
- Create quiz
- Auto-create topic if missing (`bulkCreate` flow): line 578 creates a `topics` row first, then proceeds.
- `bulkCreate("quiz_responses", creates)`

**SDK equivalent**: `<QuizCaptureForm onSubmit={…}>` → `useRecordCreate` on `quizzes` and `bulkCreate` on `quiz_responses`. The "auto-create topic" path becomes a conditional `useRecordCreate` on `topics` first.

### `views/Plan.tsx` (deployed: `loadPlan`, 597–649)
**Reads**: `revision_plans`, `plan_items`, `topics`.
**Mutations**:
- L637: `client.records.update("revision_plans", p.id, { status: "archived" })`
- L642–644: `bulkDelete("plan_items", its.map(x => x.id))` then `delete("revision_plans", p.id)`

**The big modal**: lines 651–691 (`openNewPlan` / `_newPlanModal`)
- L678: `client.workflows.run("generate-revision-plan", { … })` — first creates an empty plan via `create_revision_plan` workflow, then triggers `generate-revision-plan` workflow.

**SDK equivalent**: `<NewPlanModal>` posts the form to `useWorkflowRun("generate-revision-plan", { input })` after `useWorkflowRun("create_revision_plan", …)` (or vice versa — check the deployment order in step 5 below).

### `views/Coach.tsx` (deployed: `loadCoach`, 693–722)
**Reads**: none directly.
**Mutations**: none.
**One call**: L717
```js
const task = await client.agents.run(
  "study-coach-agent",
  { input: text },
  { wait: true, timeout_seconds: 180 }
);
```

**SDK equivalent**: `useAgentRun("study-coach-agent", { input })`. The deployed bundle surfaces raw `finalOutputText || output_text || output` chain (L719) — replicate via the SDK's `.finalOutputText` (or whichever the SDK exposes — see `lemma-sdk/react`'s `useAgentRun`).

## Live updates → `useLiveRecords`

L214–224 sets one `watchChanges` watcher for all 10 tables. Convert to per-view:
each view already calls `useRecordList` for its tables, and replacing those with
`useLiveRecords` yields the same auto-re-render-on-change behavior without
manually invalidating queries.

Tables to enable `useLiveRecords` on (from L215):
`subjects`, `topics`, `materials`, `quizzes`, `quiz_responses`,
`confidence_ratings`, `srs_cards`, `revision_plans`, `plan_items`, `study_sessions`.

## Step-by-step checklist

1. **Read** `reference/deployed-index.html` end-to-end. Note the boot sequence
   at L178–184 (`client.initialize()` → auth redirect) — this is replaced by
   `<AuthGuard client={lemmaClient}>` (already wired in `src/main.tsx`).
2. **Replace** `src/main.tsx`'s sample `<App />` with a shell that has the
   sidebar (`<aside className="sidebar">`) and `<main>`. Reproduce the dark
   palette tokens from `:root` in `src/styles.css` (lines 7–16 of the bundle).
3. **Create** `src/lib/format.ts` for `fmt(iso)`, `todayISO()`, and the
   `weaknessPill(w)` helper at L190 (returns class names — see L36–44 of CSS).
4. **Create components** in `src/components/`:
   - `Card`, `Pill`, `Bar`, `PlanRow`, `TopicRow`, `Modal`, `Stat`.
   - Map directly to CSS classes from the bundle's `<style>` (`.card`,
     `.pill.pending|done|skipped|weak|moderate|strong`, `.bar`, `.plan-row`,
     `.topic`, `.modal-bg`, `.stat-num`).
5. **Create views** in `src/views/`, one per tab. Use `useRecordList`,
   `useRecordCreate`, `useRecordUpdate`, `useLiveRecords`, `useFunctionRun`,
   `useWorkflowRun`, `useAgentRun` from `lemma-sdk/react`.
6. **Wire** routing (React Router or a single `useState<"dashboard"|"decks"|…>`).
7. **Test** with `npm run dev`, then `npm run build`, then
   `lemma apps deploy learning-studio-port --source-dir . --yes`.

## Untouched but useful

- `lemma.app.json` — `nav: sidebar`, `stylePreset: soft` (matches the bundled
  dark-on-panels look loosely; consider the `terminal` preset if you want the
  closer-to-original aesthetic).
- `AGENTS.md` (project root) — the AI-editing contract for this app.
- `.env.local` — already points at `pod 019efe5e-b261-76be-8124-a686c1d07311`.
