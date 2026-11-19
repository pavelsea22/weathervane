# Debate Lab — Data Schema

Storage: **SQLite** (one file per study, e.g. `data/study.db`). Easy to query with pandas and to publish alongside the write-up. Every LLM call is also logged raw, so any result can be re-derived or audited.

## Design principles

1. **Raw first, derived second.** `llm_calls` stores exact requests/responses. Everything else (`turns`, `flips`, labels) is parsed or computed from it and can be rebuilt.
2. **Seats, not model names.** Inside a debate, participants are `Agent A/B/C`. The mapping seat → model lives in `debate_seats`. This is what makes anonymized vs. labeled conditions possible.
3. **Round 0 = independent answer.** Every debater answers alone first. Flips are measured relative to that.
4. **Self-report vs. ground truth are stored side by side.** A model *says* it changed its mind and *why*; the harness *computes* whether it did, and counterfactual reruns test the *why*. Mismatches are findings, not bugs.
5. **Everything reproducible:** config snapshot, git commit, seed, exact model version string, prompt hashes.

## Entity overview

```
experiments ─┬─ conditions ─┐
             │              ├─ debates ─┬─ debate_seats ── agents
questions ───┴──────────────┘           ├─ turns ─── argument_labels
                                        │     └──── llm_calls
                                        └─ flips ─── counterfactual_runs
```

## Tables

### `experiments`
One research question (e.g. "Who caves first?").

| column | type | notes |
|---|---|---|
| id | TEXT PK | e.g. `exp_persuadability_v1` |
| name | TEXT | |
| hypothesis | TEXT | written *before* running |
| config_json | TEXT | full config snapshot |
| git_commit | TEXT | harness commit at run time |
| created_at | TEXT | ISO-8601 UTC |

### `conditions`
A variant within an experiment (anonymized vs. labeled, saboteur on/off, etc.).

| column | type | notes |
|---|---|---|
| id | TEXT PK | e.g. `anon_3r` |
| experiment_id | TEXT FK | |
| name | TEXT | |
| n_rounds | INTEGER | debate rounds after round 0 |
| identity_mode | TEXT | `anonymous` \| `labeled` \| `fake_labeled` |
| show_confidence | INTEGER | 0/1 — do peers see each other's confidence |
| speaking_mode | TEXT | `simultaneous` \| `sequential` |
| order_policy | TEXT | `fixed` \| `shuffled` \| `rotate` |
| saboteur | INTEGER | 0/1 |
| stop_rule | TEXT | `fixed_rounds` \| `unanimous` \| `stable_2_rounds` |
| params_json | TEXT | anything else |

### `questions`
Pre-screened items with known answers.

| column | type | notes |
|---|---|---|
| id | TEXT PK | |
| source | TEXT | dataset name |
| source_id | TEXT | id in that dataset |
| text | TEXT | |
| choices_json | TEXT | `{"A": "...", "B": "..."}` or NULL for free-form |
| correct_answer | TEXT | `"C"` or a normalized value |
| answer_type | TEXT | `multiple_choice` \| `numeric` \| `short_text` |
| domain | TEXT | math, logic, science, trivia… |
| prescreen_json | TEXT | round-0 answers from screening run; why it was kept |
| initial_disagreement | INTEGER | 0/1 — kept only if models disagreed |

### `agents`
A model + settings + role. Same model with a different role or prompt = different agent.

| column | type | notes |
|---|---|---|
| id | TEXT PK | e.g. `claude_debater_t07` |
| provider | TEXT | via OpenRouter or direct |
| model_id | TEXT | requested model string |
| role | TEXT | `debater` \| `saboteur` \| `labeler` \| `reflector` \| `judge` |
| temperature | REAL | |
| max_tokens | INTEGER | |
| system_prompt | TEXT | full text |
| system_prompt_hash | TEXT | sha256 |

### `debates`
One question run under one condition.

| column | type | notes |
|---|---|---|
| id | TEXT PK | ULID |
| experiment_id | TEXT FK | |
| condition_id | TEXT FK | |
| question_id | TEXT FK | |
| seed | INTEGER | drives seat assignment + order shuffles |
| status | TEXT | `running` \| `done` \| `failed` |
| rounds_completed | INTEGER | |
| consensus_reached | INTEGER | 0/1 (all seats same answer at end) |
| final_answer | TEXT | consensus answer, or majority, or NULL |
| final_correct | INTEGER | 0/1/NULL |
| total_cost_usd | REAL | |
| started_at / finished_at | TEXT | |

### `debate_seats`
Seat ↔ agent mapping. The only place model identity is attached to a seat.

| column | type | notes |
|---|---|---|
| debate_id | TEXT FK | PK part |
| seat | TEXT | `A`, `B`, `C`… PK part |
| agent_id | TEXT FK | |
| display_name | TEXT | what peers see: `Agent A`, or `GPT` in labeled mode, or a *wrong* brand in `fake_labeled` |
| speak_position | INTEGER | order within a round (sequential mode) |

### `turns`
One seat's output in one round. The core analysis table.

| column | type | notes |
|---|---|---|
| id | TEXT PK | |
| debate_id | TEXT FK | |
| round | INTEGER | 0 = independent |
| seat | TEXT | |
| llm_call_id | TEXT FK | raw call |
| parse_ok | INTEGER | 0/1 |
| answer | TEXT | normalized |
| is_correct | INTEGER | computed vs. `questions.correct_answer` |
| confidence | INTEGER | 0–100 |
| argument | TEXT | the public text peers see |
| self_reported_changed | INTEGER | what the model claims |
| computed_changed | INTEGER | answer ≠ this seat's previous round answer |
| change_reason_type | TEXT | enum, see STRUCTURED_OUTPUT.md |
| influences_json | TEXT | `[{"seat":"B","round":1,"weight":"primary"}]` |
| key_point | TEXT | one-line summary of what moved it |
| visible_turn_ids_json | TEXT | exactly which peer turns were in its context |

`visible_turn_ids_json` matters: in sequential mode, seat C sees round-N turns from A and B, but A doesn't. Without this you can't do causal analysis.

### `llm_calls`
Raw log of every API call, for any purpose.

| column | type | notes |
|---|---|---|
| id | TEXT PK | |
| purpose | TEXT | `turn` \| `label` \| `counterfactual` \| `prescreen` \| `reflect` |
| agent_id | TEXT FK | |
| model_returned | TEXT | exact model/version string from the response |
| request_json | TEXT | full messages + params + schema |
| response_json | TEXT | full raw response |
| attempt | INTEGER | retry count |
| finish_reason | TEXT | |
| input_tokens / output_tokens | INTEGER | |
| cost_usd | REAL | |
| latency_ms | INTEGER | |
| created_at | TEXT | |

### `flips` (derived)
One row per computed answer change. Rebuilt by `analyze.py` from `turns`.

| column | type | notes |
|---|---|---|
| id | TEXT PK | |
| debate_id | TEXT FK | |
| seat | TEXT | |
| agent_id | TEXT | denormalized for easy grouping |
| from_turn_id / to_turn_id | TEXT FK | |
| from_answer / to_answer | TEXT | |
| flip_type | TEXT | `W→R` \| `R→W` \| `W→W` |
| moved_to_majority | INTEGER | 0/1 — did it join the current majority |
| moved_to_saboteur | INTEGER | 0/1 |
| confidence_delta | INTEGER | |
| self_reported_cause_turn_id | TEXT | from `influences_json` (primary) |
| verified_cause | TEXT | `confirmed` \| `not_causal` \| `spontaneous` \| `untested` — from counterfactuals |

### `argument_labels`
A labeler model's classification of each turn's argument (who used which persuasion tactic).

| column | type | notes |
|---|---|---|
| turn_id | TEXT FK | PK part |
| labeler_agent_id | TEXT FK | PK part (allows multiple labelers for agreement checks) |
| label_json | TEXT | full structured output |
| primary_type | TEXT | enum |
| new_information | INTEGER | 0/1 |
| has_calculation | INTEGER | 0/1 |
| appeals_to_consensus | INTEGER | 0/1 |
| appeals_to_authority | INTEGER | 0/1 |
| contains_factual_error | INTEGER | 0/1/NULL |

### `counterfactual_runs`
Re-runs of a flipping turn with its context modified, to test causation. Each variant is sampled several times.

| column | type | notes |
|---|---|---|
| id | TEXT PK | |
| flip_id | TEXT FK | |
| variant | TEXT | `original_replay` \| `remove_turn` \| `only_turn` \| `no_peers_reflect` |
| target_turn_id | TEXT | the peer turn removed / kept |
| n_samples | INTEGER | e.g. 5 |
| n_flipped | INTEGER | how many samples still flipped |
| llm_call_ids_json | TEXT | |

**Reading it:** if `original_replay` flips 5/5 but `remove_turn` flips 0/5, that turn caused the flip → `verified_cause = confirmed`. If `no_peers_reflect` also flips 4/5, the "persuasion" was mostly the model second-guessing itself → `spontaneous`.

## SQLite DDL

```sql
PRAGMA foreign_keys = ON;

CREATE TABLE experiments (
  id TEXT PRIMARY KEY, name TEXT NOT NULL, hypothesis TEXT,
  config_json TEXT NOT NULL, git_commit TEXT, created_at TEXT NOT NULL
);

CREATE TABLE conditions (
  id TEXT PRIMARY KEY,
  experiment_id TEXT NOT NULL REFERENCES experiments(id),
  name TEXT NOT NULL, n_rounds INTEGER NOT NULL,
  identity_mode TEXT NOT NULL CHECK (identity_mode IN ('anonymous','labeled','fake_labeled')),
  show_confidence INTEGER NOT NULL DEFAULT 1,
  speaking_mode TEXT NOT NULL CHECK (speaking_mode IN ('simultaneous','sequential')),
  order_policy TEXT NOT NULL DEFAULT 'shuffled',
  saboteur INTEGER NOT NULL DEFAULT 0,
  stop_rule TEXT NOT NULL DEFAULT 'fixed_rounds',
  params_json TEXT
);

CREATE TABLE questions (
  id TEXT PRIMARY KEY, source TEXT NOT NULL, source_id TEXT,
  text TEXT NOT NULL, choices_json TEXT, correct_answer TEXT NOT NULL,
  answer_type TEXT NOT NULL CHECK (answer_type IN ('multiple_choice','numeric','short_text')),
  domain TEXT, prescreen_json TEXT, initial_disagreement INTEGER
);

CREATE TABLE agents (
  id TEXT PRIMARY KEY, provider TEXT NOT NULL, model_id TEXT NOT NULL,
  role TEXT NOT NULL CHECK (role IN ('debater','saboteur','labeler','reflector','judge')),
  temperature REAL, max_tokens INTEGER,
  system_prompt TEXT NOT NULL, system_prompt_hash TEXT NOT NULL
);

CREATE TABLE debates (
  id TEXT PRIMARY KEY,
  experiment_id TEXT NOT NULL REFERENCES experiments(id),
  condition_id TEXT NOT NULL REFERENCES conditions(id),
  question_id TEXT NOT NULL REFERENCES questions(id),
  seed INTEGER NOT NULL, status TEXT NOT NULL DEFAULT 'running',
  rounds_completed INTEGER DEFAULT 0, consensus_reached INTEGER,
  final_answer TEXT, final_correct INTEGER, total_cost_usd REAL DEFAULT 0,
  started_at TEXT, finished_at TEXT
);

CREATE TABLE debate_seats (
  debate_id TEXT NOT NULL REFERENCES debates(id),
  seat TEXT NOT NULL, agent_id TEXT NOT NULL REFERENCES agents(id),
  display_name TEXT NOT NULL, speak_position INTEGER,
  PRIMARY KEY (debate_id, seat)
);

CREATE TABLE llm_calls (
  id TEXT PRIMARY KEY, purpose TEXT NOT NULL,
  agent_id TEXT REFERENCES agents(id), model_returned TEXT,
  request_json TEXT NOT NULL, response_json TEXT,
  attempt INTEGER DEFAULT 1, finish_reason TEXT,
  input_tokens INTEGER, output_tokens INTEGER, cost_usd REAL,
  latency_ms INTEGER, created_at TEXT NOT NULL
);

CREATE TABLE turns (
  id TEXT PRIMARY KEY,
  debate_id TEXT NOT NULL REFERENCES debates(id),
  round INTEGER NOT NULL, seat TEXT NOT NULL,
  llm_call_id TEXT REFERENCES llm_calls(id),
  parse_ok INTEGER NOT NULL, answer TEXT, is_correct INTEGER,
  confidence INTEGER CHECK (confidence BETWEEN 0 AND 100),
  argument TEXT, self_reported_changed INTEGER, computed_changed INTEGER,
  change_reason_type TEXT, influences_json TEXT, key_point TEXT,
  visible_turn_ids_json TEXT,
  UNIQUE (debate_id, round, seat)
);

CREATE TABLE flips (
  id TEXT PRIMARY KEY,
  debate_id TEXT NOT NULL REFERENCES debates(id),
  seat TEXT NOT NULL, agent_id TEXT NOT NULL,
  from_turn_id TEXT NOT NULL REFERENCES turns(id),
  to_turn_id TEXT NOT NULL REFERENCES turns(id),
  from_answer TEXT, to_answer TEXT,
  flip_type TEXT CHECK (flip_type IN ('W→R','R→W','W→W')),
  moved_to_majority INTEGER, moved_to_saboteur INTEGER,
  confidence_delta INTEGER, self_reported_cause_turn_id TEXT,
  verified_cause TEXT DEFAULT 'untested'
);

CREATE TABLE argument_labels (
  turn_id TEXT NOT NULL REFERENCES turns(id),
  labeler_agent_id TEXT NOT NULL REFERENCES agents(id),
  label_json TEXT NOT NULL, primary_type TEXT,
  new_information INTEGER, has_calculation INTEGER,
  appeals_to_consensus INTEGER, appeals_to_authority INTEGER,
  contains_factual_error INTEGER,
  PRIMARY KEY (turn_id, labeler_agent_id)
);

CREATE TABLE counterfactual_runs (
  id TEXT PRIMARY KEY,
  flip_id TEXT NOT NULL REFERENCES flips(id),
  variant TEXT NOT NULL CHECK (variant IN ('original_replay','remove_turn','only_turn','no_peers_reflect')),
  target_turn_id TEXT REFERENCES turns(id),
  n_samples INTEGER NOT NULL, n_flipped INTEGER NOT NULL,
  llm_call_ids_json TEXT
);

CREATE INDEX idx_turns_debate ON turns(debate_id, round);
CREATE INDEX idx_flips_agent ON flips(agent_id, flip_type);
CREATE INDEX idx_debates_cond ON debates(condition_id);
```

## Example analysis queries

**Persuadability by model** (share of turns after round 0 where the model changed its answer):
```sql
SELECT s.agent_id,
       AVG(t.computed_changed) AS flip_rate,
       COUNT(*) AS n_turns
FROM turns t
JOIN debate_seats s ON s.debate_id = t.debate_id AND s.seat = t.seat
WHERE t.round > 0 AND t.parse_ok = 1
GROUP BY s.agent_id;
```

**Does debate help or hurt?** (net flips):
```sql
SELECT agent_id,
       SUM(flip_type = 'W→R') AS fixed,
       SUM(flip_type = 'R→W') AS broken,
       SUM(flip_type = 'W→R') - SUM(flip_type = 'R→W') AS net
FROM flips GROUP BY agent_id;
```

**Self-report honesty** (claimed a change but didn't, or vice versa):
```sql
SELECT s.agent_id,
       AVG(t.self_reported_changed != t.computed_changed) AS mismatch_rate
FROM turns t
JOIN debate_seats s USING (debate_id, seat)
WHERE t.round > 0 AND t.parse_ok = 1
GROUP BY s.agent_id;
```

**Who persuades whom** (heatmap source; verified causes only):
```sql
SELECT ft.agent_id AS persuaded,
       cs.agent_id AS persuader,
       COUNT(*)    AS n
FROM flips ft
JOIN turns cause ON cause.id = ft.self_reported_cause_turn_id
JOIN debate_seats cs ON cs.debate_id = cause.debate_id AND cs.seat = cause.seat
WHERE ft.verified_cause = 'confirmed'
GROUP BY persuaded, persuader;
```

## Rules of thumb

- Never delete rows from `llm_calls`. Failed parses stay, with `parse_ok = 0`, and get reported as a rate.
- Record `model_returned` from the response, not just what you requested — providers silently update models.
- Pin a `seed` per debate and derive all randomness (seat assignment, order shuffles) from it.
- Rebuild `flips` from `turns` with a script; don't hand-edit derived tables.
