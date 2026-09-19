LOGEMICS — MEASUREMENT SYSTEM

**D1 Structural Analysis:** the broader critical-thinking dimension being assessed.
**— Indicator I1_1: Isolation of Premises and Conclusion:** one specific skill inside that dimension: can the learner identify which statements are premises, which one is the conclusion, and which are irrelevant?

**A Concrete Walkthrough — Task Template, Rubric, Exercise Format, End-to-End Example:** the document explains the full pipeline for that indicator: how the question is designed, how answers are scored, and how the final ability estimate is produced.

Version: v0.1-draft  |  Status: draft (pending psychometrician review)  Read by Alaa |  Date: 2026-07-06

_Scope note: this document is a per-indicator deliverable. artifacts here are DRAFT measurement content for indicator I1_1 only. Theta estimation unit is PER-INDICATOR_

# 0. Exercise Format — Decision and Rationale

Chosen format: Select statements (learner assigns a role — Premise / Conclusion / Neither — to each statement of a short argument).

Why this format, before how: the behaviour targeted by I1_1 is identification of argumentative roles, not production of text. A closed selection format captures exactly that behaviour and nothing else. A free-text format ("rewrite the conclusion in your own words") would import writing ability into the score — construct-irrelevant variance. This maps to "Analyse détaillée / Catégorisation" in Liu, Frankel & Roohr (2014) and to structure-identification tasks in argument-analysis batteries.

Consequence for the pipeline: scoring for this indicator is FULLY DETERMINISTIC. The rubric is still human-authored (INV-1 respected), but it is applied by CODE, not by the LLM scorer. No LLM in the scoring path means: no extraction_trace with model_id (replaced by extraction_version of the deterministic scorer), no Kappa B exposure for this template. Kappa A (editorial validation of generated instances) still applies in full.

> [!note]
> Alternatives considered and rejected:
> — Mark material in text (annotation): measures the same construct, but free-span annotation makes deterministic extraction fragile (span-boundary normalization). Statement-level selection is the same measurement with a cleaner data contract.
> — Short constructed-response: adds LLM scoring cost and fragility where a closed format measures the identical behaviour. Reserved for indicators where production is the construct (I1_5 enthymemes).
> — Multiple-choice (single answer 'which is the conclusion?'): only 1 bit of information; the per-statement role assignment yields a polytomous 0–3 signal, needed by the GRM.

OPEN — for psychometrician: the deterministic 0–3 mapping (section 1c) assumes the selection-pattern scale is ordinal in the GRM sense. This is a design hypothesis, not an established result. Must be checked against pilot response distributions.

# 1. The Psychometrician Writes the Measurement Material
_A **psychometrician** is a specialist in measuring human abilities, skills, knowledge, or psychological traits using scientifically designed assessments._
## 1a. The Claim

> [!info] DEFINITION — Claim
> A precise, testable statement of what observable behaviour the assessment is measuring. It binds the task to the construct.


> [!note]
> Claim C-I1_1: 'The learner can decompose a short natural-language argument into its functional parts — identifying which statements serve as premises, which single statement is the conclusion, and which statements play no argumentative role.'

Every task instance generated for I1_1 must produce evidence for or against this claim, and only this claim (mono-indicator rule, INV-5).

## 1b. The Task Template

> [!info] DEFINITION — Task Template
> A reusable skeleton for generating items. For I1_1: a slot-based argument of exactly 5 statements — 2 premises, 1 conclusion, 2 noise statements (contextually plausible, argumentatively inert). The learner assigns one role per statement.


What the psychometrician writes — the actual JSON file:

```json
{
  "template_id": "TT-D1-I1_1-01",
  "template_version": "1.0",
  "dimension_id": "D1",
  "indicator_id": "I1_1",
  "claim_id": "C-I1_1",
  "name": "Argument decomposition — role assignment (P / C / Neither)",
  "language": "EN",
  "exercise_format": "select_statements",
  "response_format": "CLOSED",
  "prompt_structure": {
    "slots": [
      "topic",
      "premise_1",
      "premise_2",
      "conclusion",
      "noise_1",
      "noise_2"
    ],
    "stimulus_template": "Read the following five statements about {topic}. They are presented in shuffled order.\nS1..S5 = shuffle({premise_1}, {premise_2}, {conclusion}, {noise_1}, {noise_2})\nThe conclusion must be logically derivable from the two premises. Noise statements must be on-topic but must neither support nor follow from the argument.",
    "instructions": "For each statement S1–S5, select its role: (A) Premise — it supports the conclusion, (B) Conclusion — it is what the argument tries to establish, (C) Neither — it is context only. Exactly one statement is the conclusion.",
    "presentation_rules": [
      "Statement order is randomized per instance (seed-recorded).",
      "No connective words ('therefore', 'because') may appear in the rendered statements — slot filling MUST strip them. Rationale: connectives are lexical shortcuts; keeping them would measure keyword spotting, not structural analysis."
    ]
  },
  "answer_key": {
    "role_map": {
      "premise_1": "premise",
      "premise_2": "premise",
      "conclusion": "conclusion",
      "noise_1": "neither",
      "noise_2": "neither"
    }
  },
  "expected_error_patterns": [
    {
      "id": "EP-I11-a",
      "description": "Premise/conclusion inversion: marks a premise as the conclusion (often the most general-sounding statement)."
    },
    {
      "id": "EP-I11-b",
      "description": "Noise absorption: marks a noise statement as a premise because it is topically related."
    },
    {
      "id": "EP-I11-c",
      "description": "Position heuristic: assigns 'conclusion' to the last-displayed statement regardless of content."
    }
  ],
  "evidence_requirements": [
    "rubric_level"
  ],
  "item_parameters": {
    "a": "TBD (design-fixed, template level — OD-SYS-009)",
    "b": "TBD"
  },
  "status": "draft"
}
```

Status is 'draft'. Only the psychometrician can move it to 'approved'. Code refuses to instantiate a draft template in production (INV-7).

Note the constraint doing real measurement work: stripping connectives. With 'therefore' left in, pilot literature on argument-identification tasks shows learners keyword-match instead of analysing structure. I cannot cite a specific effect size from memory — flag for Lucas: verify against the argument-analysis literature before approving.

## 1c. The Rubric (deterministic mapping)

> [!info] DEFINITION — Rubric — deterministic variant
> For a closed format, the rubric is a human-authored MAPPING from response patterns to levels 0–3. It plays the same contractual role as an LLM rubric: it defines what each level means. It is applied by code, versioned, and amendable through the same supersession workflow.


```json
{
  "rubric_id": "RUB-D1-I1_1-01",
  "rubric_version": "0.1",
  "dimension_id": "D1",
  "indicator_id": "I1_1",
  "applied_by": "code_deterministic",
  "levels": [
    {
      "level": 0,
      "label": "No structural reading",
      "descriptor": "Conclusion not identified AND fewer than 2 statements correctly classified overall."
    },
    {
      "level": 1,
      "label": "Partial structure",
      "descriptor": "Conclusion not identified, BUT at least one premise correctly classified — some structural signal, wrong anchor."
    },
    {
      "level": 2,
      "label": "Anchored",
      "descriptor": "Conclusion correctly identified; at least one classification error among the remaining four statements."
    },
    {
      "level": 3,
      "label": "Full decomposition",
      "descriptor": "Conclusion correctly identified AND all four remaining statements correctly classified (2 premises, 2 neither)."
    }
  ],
  "interpretation_rules": [
    "G-1: The conclusion is the pivot. Levels 2–3 require it; no combination of correct premise picks reaches Level 2 without it.",
    "G-2: Marking more than one statement as 'conclusion' is blocked by the UI; if it occurs via data error, score Level 0 and flag the response for engineering review.",
    "G-3: A blank/incomplete response (any statement left unassigned) = Level 0, flagged 'incomplete', scoring_eligible stays true. OPEN: Lucas to confirm incomplete responses should enter the GRM rather than be excluded.",
    "G-4: Error pattern is derived, not scored: EP-I11-a if a premise was marked conclusion; EP-I11-b if any noise marked premise; EP-I11-c if the last-displayed statement was marked conclusion and is not the true conclusion. Patterns feed item diagnostics, never the level."
  ],
  "status": "draft"
}
```

Why the conclusion is the pivot (G-1): identifying the conclusion is the logically prior act — premises are only definable relative to what they support. A learner who classifies premises 'correctly' without locating the conclusion is pattern-matching, not decomposing. This ordering claim is defensible from argumentation theory (a premise IS a premise only relative to a conclusion) but the resulting 0<1<2<3 ordinality must still be verified empirically (see Open Decisions).

# 2. The Code Generates a Task Instance

The generator fills slots (code + LLM assistance for surface content), records the seed, and emits the instance WITH its answer key. Because the answer key is slot-dependent, it is a HYPOTHESIS until the human editorial validator confirms it: the validator must check that the intended conclusion really is derivable from the two premises and that the noise statements are truly inert.

```json
{
  "task_instance_id": "ti_i11_000108",
  "template_id": "TT-D1-I1_1-01",
  "template_version": "1.0",
  "generator_version": "taskgen-0.4",
  "seed": 108,
  "dimension_id": "D1",
  "indicator_id": "I1_1",
  "slots_filled": {
    "topic": "the school garden project",
    "premise_1": "Every class that maintains a garden plot this term earns project credit.",
    "premise_2": "Ms. Rivera's class maintains a garden plot this term.",
    "conclusion": "Ms. Rivera's class earns project credit.",
    "noise_1": "The garden was first planted three years ago near the east wing.",
    "noise_2": "Tomatoes and herbs are the most popular crops among students."
  },
  "display_order": [
    "noise_1",
    "premise_2",
    "conclusion",
    "noise_2",
    "premise_1"
  ],
  "prompt_rendered": "Read the five statements about the school garden project (shuffled order). For each, choose: Premise / Conclusion / Neither. Exactly one is the conclusion.\nS1: The garden was first planted three years ago near the east wing.\nS2: Ms. Rivera's class maintains a garden plot this term.\nS3: Ms. Rivera's class earns project credit.\nS4: Tomatoes and herbs are the most popular crops among students.\nS5: Every class that maintains a garden plot this term earns project credit.",
  "answer_key": {
    "S1": "neither",
    "S2": "premise",
    "S3": "conclusion",
    "S4": "neither",
    "S5": "premise"
  },
  "qa_status": "approved",
  "created_at": "2026-07-06T08:00:00Z"
}
```

Editorial validation (2b) applies unchanged: human annotates 0–3 for item quality; the AI validator gives a blind second opinion; Kappa A is computed on the batch (~30–50 items), never per item. Nothing in this indicator changes the Kappa A machinery.

# 3. The Learner Responds — Zone RAW

Canonical learner: Sarah. She sees the rendered prompt and assigns:

> [!note]
> S1 → Neither   S2 → Premise   S3 → Conclusion   S4 → Premise   S5 → Premise
> (She correctly finds the conclusion, but absorbs a noise statement — S4 — as a premise.)

```json
{
  "response_id": "rsp_002214",
  "learner_id": "lrn_sarah01",
  "task_instance_id": "ti_i11_000108",
  "session_id": "ses_20260706_003",
  "form_id": "FORM_A",
  "selections": {
    "S1": "neither",
    "S2": "premise",
    "S3": "conclusion",
    "S4": "premise",
    "S5": "premise"
  },
  "submitted_at": "2026-07-06T09:12:40Z",
  "raw_telemetry": {
    "time_on_task_ms": 74000,
    "revision_count": 2
  }
}
```

No score, no level, no accuracy in this object. RAW is sealed (INV-2). If rubric G-rules change tomorrow, this exact object is re-scored — never edited.

# 4. Extraction — Zone DERIVED (deterministic only)

- **CODE** — Compare selections to answer_key. Apply RUB-D1-I1_1-01 v0.1 mapping. Derive error pattern (G-4). No LLM involved.
- **IN** — Response rsp_002214 + TaskInstance ti_i11_000108 + Rubric v0.1
- **OUT** — EvidenceFeature: rubric_level = 2, error_pattern = EP-I11-b

Trace of the mapping: conclusion S3 correct → Level ≥ 2. Remaining four: S1 ✓, S2 ✓, S5 ✓, S4 ✗ (noise marked premise) → one error → Level 2, not 3. Pattern: EP-I11-b.

```json
{
  "evidence_id": "ev_002214_rub",
  "response_id": "rsp_002214",
  "dimension_id": "D1",
  "indicator_id": "I1_1",
  "feature_name": "rubric_level",
  "feature_value": 2,
  "error_pattern": "EP-I11-b",
  "signal_category": "primary_measurement",
  "scoring_eligible": true,
  "extractor": "deterministic",
  "extraction_version": "EXTR-D1-I11-v0.1",
  "extraction_trace": {
    "rubric_id": "RUB-D1-I1_1-01",
    "rubric_version": "0.1",
    "scorer": "code",
    "model_id": null
  },
  "extracted_at": "2026-07-06T09:12:41Z"
}
```

Schema note: extraction_trace stays mandatory, with model_id = null and scorer = 'code'. Keeping the same trace shape for deterministic and LLM features means the supersession/rescoring machinery is identical for both. One EvidenceFeature per response — local independence preserved.

# 5. Inference — Zone INFERRED (per indicator)

> [!info] DEFINITION — IndicatorEstimate
> PRIMARY estimation unit (pipeline v0.2.4): one GRM engine per INDICATOR. Sarah's rubric_level on I1_1 items feeds the I1_1 engine and no other. A per-dimension theta, if ever wanted, is a downstream read aggregate — never estimated here.


- **CODE** — Run GRM (grid approximation) on all scoring-eligible I1_1 features for this learner in this session. Item parameters design-fixed at template level (OD-SYS-009).
- **IN** — EvidenceFeatures for indicator I1_1 (here: 1 feature, ev_002214_rub)
- **OUT** — IndicatorEstimate: theta = +0.24, SD = 0.90 (ILLUSTRATIVE)

```json
{
  "estimate_id": "est_i11_sarah_s003",
  "learner_id": "lrn_sarah01",
  "indicator_id": "I1_1",
  "dimension_id": "D1",
  "session_id": "ses_20260706_003",
  "theta": 0.24,
  "uncertainty": {
    "type": "sd",
    "value": 0.9
  },
  "evidence_ids": [
    "ev_002214_rub"
  ],
  "evidence_counts": {
    "rubric_level": 1
  },
  "prior_estimate_id": null,
  "model_version": "infer-grid-0.1",
  "computed_at": "2026-07-06T09:20:00Z"
}
```

Numbers are ILLUSTRATIVE. The one verified property carried over from the v0.1 harness: a single observation leaves SD near the prior (~0.9) — one response cannot condemn a learner. FORM_A-style composition with 3 I1_1 items would bring SD toward ~0.65 (order of magnitude from the harness, not a guarantee).

SessionReport line the teacher would see: "I1_1 Argument decomposition: signal developing, limited evidence." Never a raw score, ranking, or percentile.

# 6. Audit Loop — What Changes for a Deterministic Indicator

Kappa B (LLM-vs-human scoring audit) does NOT apply here: there is no LLM judgment to audit. What replaces it is a cheaper, stronger check: the psychometrician reviews the RUBRIC MAPPING itself against a stratified sample of response patterns (do Level-1 patterns really reflect more competence than Level-0 patterns?). Mapping amendments follow the standard supersession workflow: new rubric_version, re-score RAW, recompute IndicatorEstimates, nothing deleted.

Item-level diagnostics still apply in full: per-instance difficulty drift (slot topics shifting real difficulty — the OD-SYS-009 risk), DIF across groups, and error-pattern frequencies (a high EP-I11-c rate signals a position-heuristic exploit → randomization or template fix).

# 7. Open Decisions — I1_1 (for Lucas)

- **OD-IND-I11-01** Ordinality of the 0–3 mapping: is 'conclusion wrong + a premise right' (L1) genuinely more competent than L0, and less than L2? Design hypothesis; check against pilot distributions before trusting GRM fit.
- **OD-IND-I11-02** Incomplete responses (G-3): include in GRM as Level 0, or exclude as missing? Measurement decision, not engineering.
- **OD-IND-I11-03** Statement count (5) and noise count (2): chosen for cognitive load at 15–17, NOT validated. More noise = harder + longer; interacts with item parameter b.
- **OD-IND-I11-04** Connective stripping rule: confirm against argument-analysis literature that keeping connectives collapses the task into keyword spotting (I flag this as likely but uncited).
- **OD-IND-I11-05** Design-fixed a/b for this template: plausible values needed from expert judgment before pilot (ties to OD-SYS-009).
- **OD-IND-I11-06** UI blocks multiple 'conclusion' picks (G-2): confirm this constraint doesn't leak the answer structure (learner learns 'exactly one conclusion' — acceptable scaffold or cue?).

Document generated for the Logemics measurement system — indicator I1_1 deliverable, v0.1-draft, July 2026. JSON artifacts are draft measurement content pending psychometrician approval; theta/SD values are illustrative.
