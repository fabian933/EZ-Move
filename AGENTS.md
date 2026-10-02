# AGENTS.md: rules for Codex in this repo

Project: **EZ Move**, an Abu Dhabi relocation roadmap. Full spec: `SPEC.md` (read it before every task).

## Non-negotiables
- **Never invent relocation facts.** All steps, fees, documents, links, and requirements come from `/data/*.json`. If data is missing, render "Varies" / "Check official source". Do not hardcode facts in components.
- The LLM (OpenAI API) is only used in `/app/api/*` routes, server-side. Never expose `OPENAI_API_KEY` to the client. Model comes from `process.env.OPENAI_MODEL`.
- AI prompts must instruct the model to use only provided data and to cite step ids.
- No real UAE Pass/TAMM integration. `/connect` is a clearly labelled mock.
- No user accounts or databases. Persist with localStorage only.
- Pain examples are paraphrased. No usernames, no verbatim posts.

## Stack
Next.js App Router, TypeScript (strict), Tailwind, Recharts, Vitest, `openai` SDK. Deploy target: Vercel.

## Conventions
- Business logic lives in `/lib` as pure functions with tests. Components stay thin.
- Keep components small; one component per file in `/components`.
- Mobile-first layouts; test at 380px width.
- UI copy: short, plain, no hype.
- After each task: `npm run build` and `npm test` must pass. Fix errors before finishing.
- Don't add dependencies beyond the stack without saying why.

## Done means
Builds cleanly, tests pass, the feature works in `npm run dev`, and nothing in the spec's acceptance criteria regressed.
