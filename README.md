# Debate Lab

**How do AI models change their minds — and why?**

Debate Lab puts frontier LLMs (Claude, GPT, Gemini and others) into structured, multi-round debates over questions with known answers, logs every turn, and measures what actually moves a model from one answer to another.

Multi-model "council" and debate apps are everywhere. Most of them assume that when models converge on an answer, the answer got better. Research on multi-agent debate says that's often not true: models conform to the majority, defer to confident peers, and abandon correct answers. Debate Lab treats the debate itself as the object of study:

- **Who caves first?** Which models change their answer most often under peer pressure.
- **Does debate fix answers or break them?** Wrong→right flips vs. right→wrong flips.
- **What actually persuades?** New evidence and worked calculations vs. bare assertion and "most of us agree."
- **Is the stated reason the real reason?** Models explain why they changed their mind. Debate Lab tests those explanations by re-running the turn with the cited message removed.

The deliverables are an open-source experiment harness, a public dataset of logged debates, a web viewer for replaying them, and a series of write-ups on the findings.

## How it works

1. **Pre-screen questions.** Take benchmark questions with known answers and keep only those where models disagree on their first, independent answer.
2. **Round 0.** Each model answers alone: argument, answer, confidence.
3. **Debate rounds.** Each model sees its peers' previous arguments and answers, responds to them, and gives an updated answer. It also reports privately whether it changed its position and which message influenced it. Peers never see that self-report.
4. **Controlled conditions.** Anonymous vs. brand-labeled participants, shuffled speaking order, confidence visible or hidden, a planted "saboteur" arguing for a wrong answer, and a no-peers "reconsider" baseline.
5. **Causal checks.** For each answer change, re-run the turn with the credited message removed, with only that message, and with no peers at all. This separates real persuasion from noise and self-doubt.
6. **Label arguments.** A separate labeler model classifies each argument type (evidence, calculation, rebuttal, assertion, consensus appeal…).
7. **Analyze and publish.** Charts, replays, raw data.

## Docs

- [`DATA_SCHEMA.md`](DATA_SCHEMA.md) — SQLite schema, design principles, example analysis queries.
- [`STRUCTURED_OUTPUT.md`](STRUCTURED_OUTPUT.md) — JSON schemas for every model call, prompt assembly, provider handling, validation.

## Planned stack

- **Harness:** Python, OpenRouter (one API for all providers), Pydantic for validation, SQLite for storage.
- **Analysis:** pandas + notebooks; charts exported for posts.
- **Viewer:** a lightweight web app to browse experiments and replay debates turn by turn, color-coded by stance.

## Milestones

### M0 — Skeleton (week 1)
- [ ] Repo setup, config format, SQLite init script from `DATA_SCHEMA.md`
- [ ] OpenRouter client with raw call logging (`llm_calls`), cost and token tracking
- [ ] Pydantic models and per-question answer enums from `STRUCTURED_OUTPUT.md`
- [ ] **Done when:** one 3-model, 2-round debate runs end to end and every turn is in the database

### M1 — Question set and pilot (week 2)
- [ ] Import 1–2 benchmark datasets; prescreen script that keeps initial-disagreement questions
- [ ] Pilot ~20 debates with cheaper models; check parse-failure rates per model and strict-schema support
- [ ] Freeze prompts; record prompt hashes
- [ ] **Done when:** a fixed question set of a few hundred disagreement items and a stable prompt

### M2 — Core experiments (weeks 3–4)
- [ ] Exp 1: **Persuadability**: flip rate per model
- [ ] Exp 2: **Fix or break**: W→R vs. R→W, compared with the single-model "reconsider" baseline
- [ ] Exp 3: **Saboteur**: how many honest models a confidently wrong agent drags along
- [ ] `analyze.py` builds the `flips` table and the core charts
- [ ] **Done when:** each experiment has a written hypothesis, results with sample sizes, and one headline chart

### M3 — Causal attribution (week 5)
- [ ] Counterfactual reruns (`remove_turn`, `only_turn`, `no_peers_reflect`) for every flip in M2
- [ ] Self-report honesty metrics: claimed change vs. actual change, credited cause vs. verified cause
- [ ] Argument labeler with two labeler models plus ~50 hand-labeled turns for agreement
- [ ] **Done when:** "who persuades whom" heatmap built from verified causes only

### M4 — Viewer (week 6)
- [ ] Browse experiments, conditions and debates
- [ ] Turn-by-turn replay with stance timeline, confidence and flip markers
- [ ] Persuasion heatmap and flip-flow charts
- [ ] Public read-only deployment with the published dataset
- [ ] **Done when:** any chart in a post links to the replay of a real debate behind it

### M5 — More experiments (weeks 7–8)
- [ ] Exp 4: **Brand deference**: anonymous vs. labeled vs. fake-labeled
- [ ] Exp 5: **Order effects**: does the first speaker anchor the result
- [ ] Exp 6: **Confidence vs. correctness**: does a confident wrong model beat a hesitant right one
- [ ] Exp 7: **Mixed panel vs. same-model panel**

### M6 — Publish (week 9+)
- [ ] Long-form write-up with methods, results, caveats and limitations
- [ ] Release the code, dataset (SQLite + JSONL export) and reproduction instructions
- [ ] Post series: one finding + one chart + one replay link per post
- [ ] Re-run the suite when major new models ship ("did the new model get more stubborn?")

## Reporting standards

- Every result states model versions (as returned by the API), sample size, condition and date.
- Report parse-failure rates. Failed calls are never dropped or hand-repaired.
- Hypotheses are written before experiments run.
- Claims are scoped to the setup tested: these models, these questions, this prompt.
- All raw calls are published so anyone can audit or reproduce a result.

## Status

Design phase. Schema and output formats are drafted; harness not yet started.

## License

TBD (MIT likely).
