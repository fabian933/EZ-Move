# Codex prompts (run in order)

Paste one prompt at a time. After each: run `npm run dev`, click through, then commit and push (Vercel redeploys). Don't start the next prompt until the checkpoint passes.

---

## Prompt 0: Repo setup (do manually, before Codex)
1. Create a GitHub repo `ez-move`.
2. Add `AGENTS.md`, `SPEC.md`, and `/data` (with `steps.json` = verified copy of `steps.seed.json`, `areas.json`, `pains.json`).
3. Commit and push. Connect the repo to Codex.

---

## Prompt 1: Scaffold and deploy
```
Read AGENTS.md and SPEC.md. Scaffold a Next.js App Router project with TypeScript (strict), Tailwind, Vitest and Recharts in this repo, keeping the existing /data folder, AGENTS.md and SPEC.md. Create the folder structure from SPEC.md section 11 with placeholder pages for /, /start, /plan, /areas, /insights, /connect, /embed/hub71. Build the landing page per SPEC section 3.1 (headline "Moving to Abu Dhabi, step by step", value prop, "Build my plan" button, two preset buttons that do nothing yet). Add npm scripts: dev, build, test. Make sure npm run build passes.
```
**Checkpoint:** builds locally; import into Vercel and see the live landing page.

---

## Prompt 2: Types, data, planner engine and tests
```
Implement SPEC.md sections 4 and 5. Create lib/types.ts, lib/data.ts (typed loaders for /data/*.json with runtime validation that throws a clear error on bad data), lib/tags.ts, lib/presets.ts (the two demo personas from README: "The Reeds" UK family of 4, kids 6 and 9, dog, employee, arrival 8 weeks from today; "Priya" Indian solo founder, hub71 true, arrival 4 weeks from today), and lib/planner.ts with deriveTags, selectSteps, prune, topoSort, schedule, summarise, blockedBy. Write lib/planner.test.ts covering every test listed in SPEC section 5. No UI changes. npm test and npm run build must pass.
```
**Checkpoint:** all tests green. If a test fails because of data (e.g. a cycle), fix the data, not the test.

---

## Prompt 3: Intake flow
```
Build /start per SPEC section 3.2: one question per screen, progress bar, back button, validation, mobile-first. Nationality is a searchable select of countries with ISO codes. On finish, save the Profile to localStorage and route to /plan. Wire the landing page preset buttons to save the preset profile and go straight to /plan.
```
**Checkpoint:** both the form and the presets land on /plan with the profile saved.

---

## Prompt 4: Roadmap timeline
```
Build /plan per SPEC section 3.3. Load the profile from localStorage (redirect to /start if missing), run the planner, and render: summary header, stage tabs with counts, vertical timeline of StepCard components (title, due date, duration, cost or "Varies", checkbox, "blocked by" chips, late flag "Start now"). Completed step ids persist in localStorage. Blocked steps are greyed with the blockers named. Leave placeholders for the pain badge, area card and chat button. Must look good at 380px.
```
**Checkpoint:** the Reeds and Priya get visibly different plans with sensible dates.

---

## Prompt 5: Step drawer, pain badges, areas
```
Build StepDrawer per SPEC section 3.4 (all sections except the AI explain button, which should be a disabled placeholder). Add PainBadge on StepCard using data/pains.json (sum counts of clusters whose step_ids include the step). Implement lib/areas.ts per SPEC section 6 with tests, an AreaCard (top 3) on /plan, and the full /areas page. Unknown rent shows "rent data pending".
```
**Checkpoint:** every step opens with an official link and source; the area card shows 3 areas with reasons.

---

## Prompt 6: AI explain and grounded chat
```
Implement SPEC section 7 using the openai npm SDK and the Responses API, model from process.env.OPENAI_MODEL. 7.1: /api/explain plus enable the "Explain for my situation" button in StepDrawer (loading state, error state). 7.2: /api/chat with streaming, a ChatPanel slide-over opened from /plan, suggested question chips, and parsing of [step_id] citations into chips that open StepDrawer. Use exactly the system prompts in the spec. Handle a missing API key with a clear error message. Never call OpenAI from the client.
```
**Checkpoint:** add `OPENAI_API_KEY` and `OPENAI_MODEL` in Vercel env vars, redeploy. Ask "Can I bring my dog?": the answer cites a step. Ask something off-topic: it refuses gracefully.

---

## Prompt 7: Hub71 mode
```
Implement SPEC section 8: hub71 track steps included when the hub71 tag is set, the housing-benefit banner on find_and_sign_lease, a Hub71 badge and resources card on /plan, and /embed/hub71 (compact intake plus roadmap, no site chrome, headers allowing iframe embedding). Add an /embed-demo.html in /public that iframes /embed/hub71 to prove it embeds.
```
**Checkpoint:** the Priya preset shows the Hub71 track; the embed demo page works.

---

## Prompt 8: Insights page
```
Build /insights per SPEC sections 3.6 and 9 using data/pains.json and Recharts: total posts stat, posts by source (bar), top 10 pain clusters (horizontal bar), pains by stage, and 6-8 paraphrased example cards. Show the headline stat on the landing page. Link clusters to their steps.
```
**Checkpoint:** real numbers render; clicking a cluster leads to its steps.

---

## Prompt 9: UAE Pass mock and calendar export
```
Build /connect per SPEC section 3.7 (clearly labelled prototype; mock sign-in, then auto-tick emirates_id_biometrics, residence_visa_issued, health_insurance and tawtheeq_registration in the user's completed list with a short animation). Implement lib/ics.ts per SPEC section 10 with a test, and an "Export to calendar" button on /plan.
```
**Checkpoint:** the .ics opens in Google/Apple Calendar; the mock flow is clearly labelled.

---

## Prompt 10: Polish and final checks
```
Polish pass against SPEC section 13 and 14: consistent spacing and type scale, one accent colour, empty/loading/error states everywhere, keyboard and screen-reader basics, no console errors, 380px layout check on every page, favicon and page titles "EZ Move". Fix anything failing the acceptance criteria. Run npm run build and npm test.
```
**Checkpoint:** both presets demo cleanly on the live URL, on a phone.
