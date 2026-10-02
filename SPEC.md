# EZ Move: Product and Technical Spec

> Source of truth for the build. Codex: read this file and `AGENTS.md` before every task.

## 1. Product

**One line:** EZ Move turns "I'm moving to Abu Dhabi" into a personalised, dated, dependency-ordered plan, from first considering the move to fully settled.

**Users:** individuals and families relocating to Abu Dhabi. A **Hub71 mode** serves founders joining Hub71.

**Core principle:** the plan comes from a **curated process graph** (`data/steps.json`). The LLM never invents steps, fees, or requirements. It only explains, personalises wording, and answers questions *from the data*.

### In scope (MVP)
1. Intake (6 questions) plus 2 one-click demo presets
2. Planner engine: filter, order, and schedule steps by date
3. Roadmap timeline UI, grouped by journey stage
4. Step detail drawer: documents, cost, duration, official link, Abu Dhabi-vs-Dubai note, common mistakes, pain badge, source, last verified
5. Area matcher: ranks neighbourhoods from budget, household, and commute
6. AI: per-step "explain for me" and a grounded chat assistant
7. Hub71 mode: founder track
8. Pain insights page: proof of real problem
9. "Connect UAE Pass" screen: **mock, labelled "Coming soon"**
10. Export plan to calendar (.ics)

### Out of scope
- HR/company dashboards
- Real UAE Pass/TAMM integration (no open API; mock only)
- User accounts, database, payments

## 2. Journey stages (enum `Stage`)

| id | Label | Meaning |
|---|---|---|
| `considering` | Considering | Is it worth it? Costs, areas |
| `committed` | Committed | Visa route, documents, attestation |
| `pre_arrival` | Before you land | Housing search, schools, pets, shipping |
| `first_30_days` | First 30 days | Medical test, Emirates ID, insurance, bank, lease, Tawtheeq, utilities |
| `settled` | Settled | UAE Pass, TAMM, licence, car, family sponsorship done |
| `embedded` | At home | Renewals, community |

## 3. User flow and screens

1. **Landing (`/`)**: headline, one-line value prop, "Build my plan" button, two preset buttons ("UK family of 4", "Hub71 founder"), and a headline pain stat from `pains.json`.
2. **Intake (`/start`)**: one question per screen, progress bar, back button. Questions:
   1. Nationality (searchable select; store ISO code)
   2. Why are you moving? `employee | founder | investor | joining_family | retiree`
   3. Joining Hub71? `yes | no` (shown only if founder)
   4. Who's coming? partner (y/n), kids (number and ages), pets (y/n)
   5. Will you drive? (y/n)
   6. Target arrival date (date picker) and monthly housing budget (AED band) and main commute destination (select: Al Maryah Island/ADGM, Downtown/Corniche, Yas/airport, Masdar/Khalifa City, flexible/remote)
3. **Roadmap (`/plan`)**:
   - Header: arrival date, "X steps · Y AED estimated known costs · Z days of lead time".
   - Left/top: stage tabs with completion counts.
   - Main: vertical timeline of step cards. Card fields: title, due date, duration, cost, status checkbox, "blocked by" chips, and a pain badge ("🔥 seen in 38 posts").
   - Steps that are blocked show greyed out with the blocking steps named.
   - Right/bottom: **Area matcher** card (top 3 areas) and **Ask EZ Move** chat button.
   - Buttons: "Export to calendar", "Connect UAE Pass (soon)".
   - Progress checkboxes are saved in localStorage.
4. **Step drawer**: opens on card click. Sections: What & why · Documents · Cost · Where (official link) · ⚠️ Abu Dhabi ≠ Dubai · Common mistakes · What movers say (2-3 paraphrased pain snippets with source type) · "Explain for my situation" (AI) · Source and last verified.
5. **Areas (`/areas`)**: all areas ranked with score breakdown.
6. **Insights (`/insights`)**: the proof-of-pain page. Total posts, sources breakdown (bar chart), top 10 pains (ranked bar chart), pains by journey stage, 6-8 paraphrased example posts.
7. **UAE Pass (`/connect`)**: mock sign-in, then a mock "synced" view where Emirates ID, visa, and Tawtheeq steps auto-tick. A banner says: "Prototype: shows the planned UAE Pass integration. No real data is accessed."

## 4. Data model (TypeScript, `lib/types.ts`)

```ts
type Stage = 'considering'|'committed'|'pre_arrival'|'first_30_days'|'settled'|'embedded';
type Track = 'core'|'hub71';
type CommuteDest = 'al_maryah'|'corniche'|'yas_airport'|'masdar_khalifa'|'flexible';

interface Step {
  id: string;                    // snake_case, unique
  title: string;
  stage: Stage;
  track: Track;
  requires_tags: string[];       // ALL must be in profile tags
  excludes_tags: string[];       // NONE may be in profile tags
  depends_on: string[];          // step ids
  summary: string;               // 1-2 plain sentences
  documents: string[];
  cost_aed: string | null;       // display string e.g. "AED 50 / contract year"; null = "Varies"
  cost_min_aed: number | null;   // for totals
  duration_days: number;         // typical elapsed time
  lead_days_before_arrival: number | null; // for pre-arrival steps: start this many days before arrival
  offset_days_after_arrival: number | null;// for post-arrival steps: earliest day after arrival
  where: { name: string; url: string | null };
  ad_vs_dubai: string | null;
  mistakes: string[];
  sources: string[];             // URLs
  verified: boolean;
  last_verified: string | null;  // ISO date
}

interface Area {
  id: string; name: string; vibe: string; tags: string[];
  rent_1bed_aed_year: [number, number] | null;
  rent_3bed_aed_year: [number, number] | null;
  rent_villa_aed_year: [number, number] | null;
  commute_min: Record<CommuteDest, number | null>; // 'flexible' is null
  commute_note: string;
  family_score: number; // 1-5
  rent_source: string | null; rent_as_of: string | null;
}

interface PainCluster {
  id: string; label: string; stage: Stage; step_ids: string[];
  count: number; sources: Record<string, number>;
  examples: { paraphrase: string; source_type: string }[]; // paraphrased, no usernames
}

interface Profile {
  nationality: string; reason: 'employee'|'founder'|'investor'|'joining_family'|'retiree';
  hub71: boolean; partner: boolean; kids: number[]; pets: boolean; drives: boolean;
  arrival_date: string; budget_band: 'lt_60k'|'60_100k'|'100_160k'|'160_250k'|'gt_250k'; // AED/year
  commute: CommuteDest;
}
```

**Profile → tags** (`lib/tags.ts`): `employee`, `founder`, `investor`, `joining_family`, `retiree`, `hub71`, `has_partner`, `has_kids`, `has_school_age_kids` (any kid aged 3-17), `has_pets`, `drives`, `has_dependents` (partner or kids), plus `nat_<iso>` (e.g. `nat_gb`).

## 5. Planner engine (`lib/planner.ts`)

Pure functions, unit-tested, no AI.

1. `deriveTags(profile) → Set<string>`
2. `selectSteps(steps, tags)`: keep the step if every `requires_tags` is in tags and no `excludes_tags` is in tags. When `hub71` is set, include `track: 'hub71'` steps.
3. `prune(selected)`: drop `depends_on` references that point to unselected steps (e.g. `entry_permit` depends on `hub71_establishment_card` only for founders).
4. `topoSort(selected)`: Kahn's algorithm with stage order as the tiebreaker. Throw on a cycle (and a test checks the seed data has none).
5. `schedule(sorted, arrivalDate)`:
   - Pre-arrival steps: `start = arrival - lead_days_before_arrival`; `due = start + duration_days`.
   - Post-arrival steps: `start = max(arrival + offset_days_after_arrival, max(due of dependencies))`; `due = start + duration_days`.
   - If a pre-arrival start is in the past → flag `late: true` (the UI shows "Start now").
6. `summarise(plan)`: step count, sum of `cost_min_aed`, earliest start, count per stage.
7. `blockedBy(stepId, completedIds)`: list of incomplete dependencies.

**Tests (`lib/planner.test.ts`, Vitest):** both presets produce plans; there are no cycles; the founder preset includes the hub71 steps and the family preset doesn't; the family preset includes `school_applications` and `pet_import_permit`; the Tawtheeq step comes before `utilities_setup` and `sponsor_family`.

## 6. Area matcher (`lib/areas.ts`)

Score = 0.4 × budget fit + 0.35 × commute + 0.25 × household fit.
- Budget fit: overlap of the user's band with the relevant rent range (villa or 3-bed for families, 1-bed for solo). If rent is null → neutral 0.5 and the UI shows "rent data pending".
- Commute: if 'flexible' → 1.0. Otherwise linear from 1.0 (≤10 min) to 0 (≥45 min).
- Household: family_score/5 if there are kids, otherwise 1 - (family_score - 3)/5 clamped. Keep it simple.

Return the top 3 with a one-line "why" per area, generated from the score components (no AI needed).

## 7. AI features (`app/api/*`)

Use the **OpenAI API** (Responses API). Model comes from env `OPENAI_MODEL`. Server-side only; never expose the key.

### 7.1 `POST /api/explain`
Input: profile and stepId. The server loads the step and its pain cluster. Instructions:
> You are EZ Move, a relocation assistant for Abu Dhabi. Using ONLY the step data provided, explain in ≤90 words what this person must do and what to watch out for, tailored to their profile. Do not add requirements, fees, or links that aren't in the data. If something depends on their situation and the data doesn't say, tell them to confirm with the official source named in `where`.

Output: plain text. Cache per (stepId, profile hash) in memory.

### 7.2 `POST /api/chat`
Grounding: the dataset is small, so **no vector DB**. Each request sends the system prompt, the user profile, the user's selected plan steps (compact JSON), all area summaries, and the top pain clusters.
System prompt:
> You are EZ Move. Answer questions about relocating to Abu Dhabi using ONLY the provided data. Cite step ids in square brackets, e.g. [tawtheeq_registration]. If the answer isn't in the data, say: "That's not in my verified data yet. Check the official source." and name the most relevant `where`. Be short and plain. Never give legal, immigration, or financial advice beyond the data.

The UI turns `[step_id]` into clickable chips that open the step drawer. Stream the response.

Suggested question chips: "What do I need before I can sign a lease?", "Can I bring my dog?", "What's different from Dubai?", "What will this cost upfront?"

## 8. Hub71 mode

Triggered by `hub71` tag. Effects:
- Adds `track: 'hub71'` steps (see seed data: application/acceptance, incentive onboarding, licence, establishment card, founder visa, corporate bank account, Hub71 housing benefit).
- `find_and_sign_lease` shows a banner: "Check your Hub71 housing benefit first" linking to `hub71_housing_benefit`.
- Roadmap header gets a Hub71 badge and a "Hub71 resources" card (links from data).
- **Embeddable:** `/embed/hub71` renders the intake plus roadmap in compact layout with no site header, suitable for an iframe on hub71.com. Add `frame-ancestors` headers that allow embedding.

## 9. Pain insights

Data comes from `data/pains.json` (produced offline, see `PAIN_PIPELINE.md`). Charts use Recharts. Example posts are **paraphrased** with no usernames or verbatim text from private groups.

## 10. Calendar export

`lib/ics.ts` generates a VCALENDAR with one all-day VEVENT per step (start/due dates, title, summary, official link in the description). Download as `ez-move-plan.ics`.

## 11. Tech stack and repo structure

- Next.js (App Router) + TypeScript + Tailwind CSS
- Recharts for charts, Vitest for tests
- `openai` npm SDK
- Data: static JSON in `/data`, imported at build time
- State: React state + localStorage (profile, completed step ids)
- Hosting: Vercel

```
/app
  page.tsx            landing
  start/page.tsx      intake
  plan/page.tsx       roadmap
  areas/page.tsx
  insights/page.tsx
  connect/page.tsx    UAE Pass mock
  embed/hub71/page.tsx
  api/explain/route.ts
  api/chat/route.ts
/components           StepCard, StepDrawer, Timeline, StageTabs, AreaCard, ChatPanel, PainBadge, PresetButtons
/lib                 types.ts tags.ts planner.ts areas.ts ics.ts data.ts presets.ts
/data                steps.json areas.json pains.json
```

## 12. Environment

```
OPENAI_API_KEY=...
OPENAI_MODEL=...      # current GPT model available on the key
```

## 13. Design

- Clean, calm, trustworthy, government-service clarity rather than a startup gloss. White background, one accent colour, generous spacing, large tap targets.
- Mobile-first: the timeline must work at 380px.
- Copy is short and plain. No hype words.
- Every factual element shows its source. Unverified steps show a small "unverified" tag (all steps should be verified before demo).

## 14. Acceptance criteria (demo-ready)

- Both presets go from landing to full plan in ≤2 clicks.
- Plans differ visibly (family: schools, pets, sponsorship; founder: Hub71 track).
- Every step drawer has an official link and a source.
- Chat cites step ids, and refuses gracefully outside the data.
- Insights page renders real collected numbers.
- Lighthouse mobile performance is acceptable; no console errors.
- Works on the live Vercel URL.

## 15. Stretch (only if everything above is done)

- ChatGPT app version via the OpenAI Apps SDK (an MCP server exposing `build_plan` and `get_step`), to hit the "ChatGPT site" deploy option.
- Arabic language toggle for UI chrome.
