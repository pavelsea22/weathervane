# Weathervane — Structured Output Formats

Every model call returns JSON that matches a schema. This doc defines the schemas, how a turn is built, and how to handle providers that don't comply perfectly.

## Principles

1. **Argument before answer.** Models generate JSON fields in schema order. Put the reasoning field first so the answer is conditioned on the reasoning, not the other way around.
2. **Constrain the answer.** For multiple choice, build the schema per question with `"enum": ["A","B","C","D"]`. Unparseable answers then become impossible instead of a cleanup job.
3. **Public vs. private fields.** Peers see only `argument`, `answer` and (when the condition allows it) `confidence`. The introspection fields (`position_changed`, `influences`, `change_reason_type`, `key_point`) are logged but **never shown to peers**, so they can't become a persuasion channel themselves.
4. **Self-reports are data, not truth.** The harness computes whether the answer changed. The model's claim is stored next to it and checked by counterfactual reruns.
5. **Strict-mode compatible.** Every field is `required`, `additionalProperties: false`, and nullable fields use `["type","null"]`. Range checks (confidence 0–100, length limits) happen in code (Pydantic), because strict modes differ in which keywords they support.

---

## 1. Debater turn (rounds ≥ 1)

### JSON Schema

```json
{
  "name": "debater_turn",
  "strict": true,
  "schema": {
    "type": "object",
    "additionalProperties": false,
    "required": [
      "argument", "answer", "confidence",
      "position_changed", "change_reason_type", "influences", "key_point"
    ],
    "properties": {
      "argument": {
        "type": "string",
        "description": "Your reasoning, addressed to the other participants. Engage with their specific points. This is the only text they will see besides your answer."
      },
      "answer": {
        "type": "string",
        "enum": ["A", "B", "C", "D"],
        "description": "Your current answer. (Enum is generated per question.)"
      },
      "confidence": {
        "type": "integer",
        "description": "Probability (0-100) that your answer is correct."
      },
      "position_changed": {
        "type": "boolean",
        "description": "True if your answer differs from your answer in the previous round."
      },
      "change_reason_type": {
        "type": "string",
        "enum": [
          "none",
          "new_evidence_from_peer",
          "found_error_in_own_reasoning",
          "found_error_in_peer_reasoning",
          "peer_calculation_or_proof",
          "peer_agreement_or_majority",
          "peer_confidence",
          "reinterpreted_question",
          "other"
        ],
        "description": "Main reason your answer or confidence moved this round. 'none' if neither changed meaningfully."
      },
      "influences": {
        "type": "array",
        "description": "Peer messages that affected your position this round. Empty if none.",
        "items": {
          "type": "object",
          "additionalProperties": false,
          "required": ["seat", "round", "weight"],
          "properties": {
            "seat": { "type": "string", "description": "e.g. 'B'" },
            "round": { "type": "integer" },
            "weight": { "type": "string", "enum": ["primary", "secondary"] }
          }
        }
      },
      "key_point": {
        "type": ["string", "null"],
        "description": "One sentence: the single point that most affected your position this round, or null."
      }
    }
  }
}
```

### Example output

```json
{
  "argument": "Agent B's unit conversion is right — I treated 1.5 hours as 1h50m. Redoing it: 90 km / 1.5 h = 60 km/h, which is C. Agent C's claim that the return trip matters doesn't apply; the question asks only about the outbound leg.",
  "answer": "C",
  "confidence": 88,
  "position_changed": true,
  "change_reason_type": "found_error_in_own_reasoning",
  "influences": [{ "seat": "B", "round": 1, "weight": "primary" }],
  "key_point": "B pointed out my hours-to-minutes conversion error."
}
```

## 2. Independent answer (round 0)

Same shape minus the change fields. No peers exist yet.

```json
{
  "name": "independent_answer",
  "strict": true,
  "schema": {
    "type": "object",
    "additionalProperties": false,
    "required": ["argument", "answer", "confidence"],
    "properties": {
      "argument":   { "type": "string" },
      "answer":     { "type": "string", "enum": ["A", "B", "C", "D"] },
      "confidence": { "type": "integer" }
    }
  }
}
```

The **reflector** control condition (one model, no peers, "reconsider your answer") uses the debater schema with `influences` always empty. That makes its output directly comparable.

## 3. Saboteur turn

Same `debater_turn` schema, so the saboteur's turns look identical to peers. The difference is entirely in its private system prompt (argue for a fixed wrong answer, sound confident, never concede). Saboteur introspection fields are ignored in analysis. `agents.role = 'saboteur'` marks it.

## 4. Argument labeler

A separate model (ideally not one of the debaters) classifies each debater turn. It sees the question, the correct answer, the turn's `argument`, and the peer turns it was responding to. It does **not** see any model names.

```json
{
  "name": "argument_label",
  "strict": true,
  "schema": {
    "type": "object",
    "additionalProperties": false,
    "required": [
      "primary_type", "secondary_types", "new_information",
      "has_calculation", "appeals_to_consensus", "appeals_to_authority",
      "contains_factual_error", "engages_with_seats", "rationale"
    ],
    "properties": {
      "primary_type": {
        "type": "string",
        "enum": [
          "new_evidence", "calculation_or_proof", "error_correction",
          "rebuttal", "restatement", "bare_assertion",
          "consensus_appeal", "concession", "other"
        ]
      },
      "secondary_types": {
        "type": "array",
        "items": { "type": "string", "enum": [
          "new_evidence", "calculation_or_proof", "error_correction",
          "rebuttal", "restatement", "bare_assertion",
          "consensus_appeal", "concession", "other"
        ]}
      },
      "new_information":        { "type": "boolean", "description": "Introduces a fact or step not present in earlier turns." },
      "has_calculation":        { "type": "boolean" },
      "appeals_to_consensus":   { "type": "boolean", "description": "Argues from 'most of us agree'." },
      "appeals_to_authority":   { "type": "boolean", "description": "Argues from who said it, or claimed expertise." },
      "contains_factual_error": { "type": ["boolean", "null"], "description": "null if unsure." },
      "engages_with_seats":     { "type": "array", "items": { "type": "string" } },
      "rationale":              { "type": "string", "description": "One or two sentences justifying the labels." }
    }
  }
}
```

To validate the labeler, hand-label ~50 turns yourself and report agreement. Run two different labeler models and store both (`argument_labels` is keyed by labeler).

## 5. Counterfactual rerun

No new schema. A counterfactual is a replay of the exact `debater_turn` request with the context edited:

| variant | context change |
|---|---|
| `original_replay` | identical request, re-sampled (measures baseline flip stability) |
| `remove_turn` | the claimed-cause peer turn is deleted |
| `only_turn` | all peer turns deleted except the claimed cause |
| `no_peers_reflect` | all peer turns deleted; "reconsider your previous answer" added |

Sample each variant N times (e.g. 5) at the debate's temperature. Record `n_flipped`.

---

## How a turn request is assembled

```
[system]  Agent's system prompt (role, rules, "respond only with JSON matching the schema")

[user]    QUESTION
          <question text>
          A) …  B) …  C) …  D) …

          YOUR PREVIOUS ANSWER (round N-1)
          Answer: B   Confidence: 70
          Argument: …

          OTHER PARTICIPANTS (round N-1)
          --- Agent B ---
          Answer: C   Confidence: 85       ← confidence omitted if show_confidence = 0
          Argument: …
          --- Agent C ---
          …

          This is round N of M. Respond to the other participants' arguments,
          then give your current answer. Change your answer only if you are
          persuaded it is wrong.
```

Notes:
- Display names come from `debate_seats.display_name`: `Agent B` (anonymous), `GPT` (labeled), or a deliberately wrong brand (`fake_labeled`).
- Peers are listed in a seed-shuffled order each round, so list position doesn't confound results. Store the order in `visible_turn_ids_json`.
- Only include **the previous round** of peer turns plus the agent's own history. Full transcripts grow fast and make prompts expensive. Make this a condition parameter (`context_window: last_round | full`) if you want to test it.
- Keep the prompt neutral. The phrase "change your answer only if persuaded" should be identical across conditions. The wording of that one sentence alone can swing flip rates, so it is worth its own experiment later.

## Provider handling

- **Through OpenRouter:** pass `response_format: {type: "json_schema", json_schema: {...}}`. Support for strict schemas varies by provider and model. Check per model in a pilot run, and record which mode was used in `llm_calls.request_json`.
- **Fallback when strict mode isn't available:** put the schema in the system prompt, parse the reply, and validate with Pydantic. On failure, retry once with the validation error appended. After that, store with `parse_ok = 0`. Never repair answers by hand.
- **Reasoning models:** the hidden "thinking" isn't part of `argument`. Log reasoning tokens when the API reports them, but analyze only the visible output.
- **Report parse-failure rate per model.** If one model fails much more often, that's a bias in your sample, so disclose it.

## Pydantic models (validation layer)

```python
from typing import Literal, Optional
from pydantic import BaseModel, Field, conint

ChangeReason = Literal[
    "none", "new_evidence_from_peer", "found_error_in_own_reasoning",
    "found_error_in_peer_reasoning", "peer_calculation_or_proof",
    "peer_agreement_or_majority", "peer_confidence",
    "reinterpreted_question", "other",
]

class Influence(BaseModel):
    seat: str = Field(pattern=r"^[A-Z]$")
    round: conint(ge=0)
    weight: Literal["primary", "secondary"]

class IndependentAnswer(BaseModel):
    argument: str = Field(min_length=1, max_length=4000)
    answer: str
    confidence: conint(ge=0, le=100)

class DebaterTurn(IndependentAnswer):
    position_changed: bool
    change_reason_type: ChangeReason
    influences: list[Influence]
    key_point: Optional[str] = Field(default=None, max_length=300)

def validate_answer(turn: IndependentAnswer, choices: list[str]) -> None:
    if turn.answer not in choices:
        raise ValueError(f"answer {turn.answer!r} not in {choices}")
```

Post-parse checks the harness runs and stores (not errors, just flags):
- `position_changed` disagrees with the computed change → `self_reported_changed != computed_changed`.
- `influences` names a seat that wasn't visible to this agent → invalid attribution; count it per model.
- `change_reason_type = "none"` but the answer changed → unexplained flip.
