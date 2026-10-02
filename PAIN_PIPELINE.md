# Proof-of-pain pipeline

**Goal:** show judges this is a real, recurring problem: "We collected N posts from M sources; these are the top pains, and each maps to a step in EZ Move."

## Be honest about sources (judges may ask)

| Source | Method | Notes |
|---|---|---|
| Reddit (r/abudhabi, r/UAE, r/dubai, r/expats) | Reddit API (PRAW) or JSON endpoints | Easiest and largest. Filter for Abu Dhabi and relocation keywords |
| Public forums (Expat Woman, Expatriates.com, Mumsnet/expat threads, TripAdvisor UAE forum) | Scrape public pages, respecting robots.txt | Good for family, school, and pet questions |
| X/Twitter | Search API if you have access, otherwise manual export | Small sample is fine |
| Facebook groups (Abu Dhabi expat/mums groups) | **Manual sampling only**: copy 20-40 posts by hand, paraphrase | Don't scrape; it breaks their terms and is blocked anyway |
| Instagram, LinkedIn | **Manual sampling only** | Same |
| Your team's own experience | Write-ups | Counts as lived evidence |

In the pitch say: "automated collection from Reddit and public forums, plus manual sampling from Facebook groups, LinkedIn, and Instagram." That is accurate and defensible.

**Search keywords:** moving to Abu Dhabi, relocating Abu Dhabi, Tawtheeq, Emirates ID delay, medical test Abu Dhabi, family visa Abu Dhabi, school places Abu Dhabi, ADEK, bring dog UAE, cheques rent, bank account without Emirates ID, driving licence conversion Abu Dhabi, Saadiyat vs Reem, cost of living Abu Dhabi, Hub71 relocation.

## Steps

1. **Collect** into `pains_raw.csv` using `data/pains.template.csv` columns. Keep the URL for public posts; for manual samples record only the source type and date.
2. **Tag with an LLM** (batch script, OpenAI API, structured output). For each post return:
   - `is_relocation_pain` (bool), `stage` (Stage enum), `topic` (short), `step_ids` (from `steps.json` ids, max 3), `severity` (1-3), `paraphrase` (≤25 words, no names).
3. **Cluster**: group by topic and step ids into 10-20 clusters (let the LLM propose labels, then merge by hand).
4. **Export** `data/pains.json` matching the `PainCluster` type in SPEC.md.
5. **Headline numbers:** total relevant posts, number of sources, top 3 clusters, and the share that is about "order/dependency" problems (e.g. needing an Emirates ID before a bank account). That last stat is the core thesis of EZ Move.

## Privacy rules
- No usernames, profile links, or verbatim quotes from private/closed groups.
- Paraphrase everything shown in the product.
- Raw data stays off the public repo (add `pains_raw.csv` to `.gitignore`).

## Tagging prompt (copy into the script)
```
You classify posts about relocating to Abu Dhabi. Given a post and the list of step ids with titles, return JSON:
{"is_relocation_pain": bool, "stage": one of [considering, committed, pre_arrival, first_30_days, settled, embedded],
 "topic": string (≤5 words), "step_ids": [up to 3 ids from the list, or []], "severity": 1|2|3,
 "paraphrase": string (≤25 words, no names, no quotes)}
Only use step ids from the list. If the post isn't about a relocation difficulty, set is_relocation_pain=false.
```
