# Provenance Guard — Planning Document

## Architecture Narrative

A piece of text enters the system via a `POST /submit` request carrying the content and a `creator_id`. The submission endpoint validates the input, assigns a UUID (`content_id`), and passes the text to two independent detection signals running in sequence. Signal 1 (LLM-based classification) sends the text to Groq's `llama-3.3-70b-versatile` and receives an `ai_probability` float. Signal 2 (stylometric heuristics) computes four measurable statistics—sentence-length uniformity, filler-word absence, AI-transition phrase density, and type-token ratio—and returns a combined float. The confidence scorer applies a weighted average (LLM 60%, stylometric 40%) to produce a single `ai_probability` score, maps that score to one of three attribution categories, and derives a confidence value. A label generator converts attribution + confidence into plain-language text for the reader. The result is returned to the caller and simultaneously written as a structured JSON entry in the audit log. The content's status is stored in a JSON file so the appeal endpoint can look it up.

An appeal enters via `POST /appeal` with a `content_id` and `creator_reasoning`. The endpoint looks up the content ID, updates its status to `"under_review"` in the content store, and appends an appeal event to the audit log. No automated re-classification occurs — the appeal is queued for human review.

## Architecture

```
POST /submit
    │
    ▼
[Input Validation]
    │ (text, creator_id)
    ▼
[Signal 1: LLM Classification]───────────────────────────────────────────┐
    │ groq llama-3.3-70b-versatile                                        │
    │ → ai_probability float [0,1]                                        │
    ▼                                                                     │
[Signal 2: Stylometric Heuristics]                                        │
    │ sentence uniformity, filler words, AI phrases, TTR                 │
    │ → stylo_score float [0,1]                                           │
    ▼                                                                     │
[Confidence Scorer]                                                       │
    │ ai_prob = 0.60 × llm + 0.40 × stylo                                │
    │ → attribution, confidence, ai_probability                          │
    ▼                                                                     │
[Label Generator]                                                         │
    │ → plain-language label string                                      │
    ▼                                                                     │
[Audit Log] ◄────────────────────────────────────────────────────────────┘
    │ content_id, attribution, confidence, both signal scores, timestamp
    ▼
[Content Store] ← stores status per content_id
    │
    ▼
JSON Response → (content_id, attribution, confidence, label, scores, status)


POST /appeal
    │
    ▼
[Input Validation]
    │ (content_id, creator_reasoning)
    ▼
[Content Store Lookup]
    │ verify content_id exists, check not already under_review
    ▼
[Status Update] → "under_review" written to content store
    │
    ▼
[Audit Log] → appeal event with creator_reasoning
    │
    ▼
JSON Response → (content_id, status: under_review, message, timestamp)
```

---

## Detection Signals

### Signal 1: LLM-Based Classification (Groq / llama-3.3-70b-versatile)

**What it measures:** Holistic semantic and stylistic patterns in the text. The LLM evaluates vocabulary consistency, sentence construction balance, the presence of overly formal transitions, emotional authenticity, and whether the writing has the natural imperfections (run-ons, contractions, personal voice) that characterize human prose.

**Why AI writing differs here:** AI models are trained to produce fluent, consistent, well-structured output. This produces a recognizable register: uniform formality, completion of every thought, avoidance of fragments, and consistent use of formal transition phrases. Human writing is more irregular — it trails off, uses colloquialisms inconsistently, and has an idiosyncratic voice.

**Output format:** A float in `[0.0, 1.0]` where `1.0` = very likely AI-generated. Derived by prompting the model with a structured assessment task and parsing its JSON response (`{"ai_probability": 0.87, "reasoning": "..."}`). Falls back to `0.5` on parse failure.

**Blind spots:** A formally trained human writer (academic, legal, technical) may score high because their writing lacks casual markers. An AI deliberately prompted to write sloppily (with typos, contractions, and fragments) may score low. The signal is also subject to the LLM's own training distribution — if the model hasn't seen much of a particular domain's human writing, it may misjudge it.

---

### Signal 2: Stylometric Heuristics (pure Python)

**What it measures:** Four measurable statistical properties of the text at the character, word, and sentence level:

1. **Sentence length uniformity** (coefficient of variation of sentence word counts): AI text tends to produce sentences of similar length; human writing has more variance.
2. **Filler/hedge word absence**: Human writing frequently includes hedges ("honestly," "kinda," "I guess," "tbh"). AI writing almost never does.
3. **AI transition and buzzword density**: AI writing overuses phrases like "Furthermore," "It is important to note," "paradigm shift," "stakeholder," "nuanced." High density signals AI authorship.
4. **Type-token ratio (TTR)**: Unique words / total words. AI text clusters in a mid-range (0.55–0.78) — more varied than repetitive casual speech, less varied than rich creative prose.

**Why AI writing differs here:** These are structural regularities that emerge from how language models generate text token-by-token, optimizing for coherence and fluency rather than for the natural messiness of spontaneous human composition.

**Output format:** A float in `[0.0, 1.0]` where `1.0` = strong AI structural indicators. Computed as a weighted combination of the four sub-signals: uniformity (30%), filler absence (25%), transition density (30%), TTR zone (15%).

**Blind spots:** Academic or technical human writing has low filler-word density and uses formal transitions legitimately. Highly polished human prose (edited essays, published articles) can resemble AI output on these metrics. Very short texts (< 3 sentences) reduce the reliability of sentence-length variance.

---

## Uncertainty Representation

### Score Interpretation

The weighted average `ai_probability = 0.60 × llm_score + 0.40 × stylo_score` represents the system's estimate of the probability that the content was AI-generated.

### Threshold Definitions

| ai_probability | Attribution | Confidence Derivation |
|---|---|---|
| ≥ 0.70 | `likely_ai` | `confidence = ai_probability` |
| ≤ 0.35 | `likely_human` | `confidence = 1.0 − ai_probability` |
| 0.36 – 0.69 | `uncertain` | `confidence = |ai_prob − 0.525| / 0.175` |

### Why These Thresholds

A false positive — labeling a human's work as AI-generated — is the more serious error on a writing platform. It damages the creator's reputation without recourse. To reduce false positives:

- The `likely_ai` threshold is set high (0.70) so the system only commits to that label when evidence is strong from both signals.
- The `uncertain` band is deliberately wide (0.36–0.69) to capture borderline cases rather than forcing a binary verdict.
- `likely_human` requires `ai_probability ≤ 0.35`, giving the system a meaningful floor of evidence before confirming human authorship.

A score of 0.51 sits in the uncertain band, near its center (0.525), and produces a low confidence score (~0.09). A score of 0.65 sits near the uncertain-to-AI boundary and produces a higher confidence (~0.71). These map to meaningfully different label text — the 0.51 case clearly communicates "we couldn't tell," while the 0.65 case explains what signals were mixed.

---

## Transparency Label Variants

All three variants use plain language — no jargon. Confidence is expressed as a percentage.

**High-confidence AI-generated** (`attribution = "likely_ai"`, e.g., confidence = 0.85):
> "AI-Generated Content (85% confidence): Our system's analysis strongly suggests this content was produced by an AI writing tool. Multiple signals — including writing style, sentence structure, and vocabulary patterns — indicate AI authorship. If you are the creator and believe this is a mistake, you can submit an appeal."

**High-confidence human-written** (`attribution = "likely_human"`, e.g., confidence = 0.91):
> "Human-Written Content (91% confidence): Our system's analysis suggests this content was written by a person. The writing shows natural variation, personal voice, and characteristics consistent with human authorship. If you believe this classification is incorrect, you can submit an appeal."

**Uncertain** (`attribution = "uncertain"`, ai_probability = 0.54, human_pct = 46):
> "Origin Unclear: Our system could not confidently determine whether this content was written by a human or generated by AI. The writing contains a mix of characteristics from both (54% AI indicators, 46% human indicators). This content has not been marked either way — the creator may submit an appeal to provide additional context for reviewers."

---

## Appeals Workflow

**Who can appeal:** Any creator who submitted content via `POST /submit`. The only identifier required is the `content_id` returned at submission time.

**What they provide:** A `creator_reasoning` string — a free-text explanation of why they believe the classification is wrong. No length limit enforced, but a meaningful explanation is expected (the field is required and non-empty).

**What the system does on receipt:**
1. Validates `content_id` exists in the content store.
2. Checks the content is not already `under_review` (prevents duplicate appeals).
3. Updates the content's status to `"under_review"` in the content store.
4. Appends an `appeal_submitted` event to the audit log with: `content_id`, `event`, `timestamp`, `creator_id`, `original_attribution`, `status`, and `appeal_reasoning`.
5. Returns a confirmation JSON including the new status and a human-readable message.

**What a human reviewer sees:** Calling `GET /log` surfaces all events including the original classification entry and the subsequent `appeal_submitted` entry for the same `content_id`. The reviewer can see the original attribution, confidence, signal scores, and the creator's stated reasoning side by side.

**No automated re-classification:** Appeals are queued for human review only. The system does not re-run detection on appeal because the signals may not have changed and automated re-classification could create an exploitable loop.

---

## Anticipated Edge Cases

**Edge case 1: Non-native English speaker with formal writing style**
A creator whose first language is not English may write in a structured, formal register that avoids contractions and slang — not because they used AI, but because that is their natural register in English. The stylometric signal would score their filler-word absence and formal sentence structure as AI-like, potentially pushing the combined score into the `uncertain` or `likely_ai` band. This is the most likely source of false positives. The wide uncertain band and appeal workflow are the primary mitigations.

**Edge case 2: Short poetry or lyrical text (< 50 words)**
Short poems often use repetition (low TTR), unconventional punctuation, and fragmented sentences. These characteristics confuse both signals: the stylometric signal sees low sentence-count variance as a data-quality problem (returning 0.5 for uniformity when there are fewer than 2 sentences), and the LLM may struggle to distinguish a deliberately spare style from AI minimalism. The system should return `uncertain` for most very short texts, which is the correct epistemic stance.

**Edge case 3: AI-assisted writing (human edits AI output)**
A creator who uses AI to draft a paragraph and then extensively edits it produces text that is neither purely human nor purely AI. Both signals will produce mid-range scores because the structural regularities are partially retained while the personal edits introduce irregularity. The system honestly produces an `uncertain` result — which is arguably the correct label for this genuinely mixed authorship scenario.

---

## AI Tool Plan

### Milestone 3 — Submission endpoint + Signal 1 (LLM)

**Spec sections to provide:** Detection Signals (Signal 1 description + output format) and Architecture diagram (submission flow).

**What to ask for:** Flask app skeleton with `POST /submit` route stub (accepting `text` + `creator_id`, returning a hardcoded response), a `llm_signal(text: str) -> float` function using Groq's chat completions API with a structured JSON-response prompt, and a `GET /log` endpoint stub.

**How to verify:** Call `llm_signal()` directly with 3 test inputs (clearly AI, clearly human, borderline). Check that the returned value is a float in `[0,1]`, that the prompt instructs the model to return only JSON, and that the fallback handles parse errors gracefully. Wire into the endpoint and confirm `content_id` appears in the response.

### Milestone 4 — Signal 2 + Confidence Scoring

**Spec sections to provide:** Detection Signals (Signal 2 sub-signals and weighting), Uncertainty Representation (thresholds table), Architecture diagram.

**What to ask for:** `stylometric_signal(text: str) -> float` function implementing the four sub-signals with specified weights, and `compute_confidence(llm_score, stylo_score) -> dict` function applying the threshold table.

**How to verify:** Run `stylometric_signal()` on all four test inputs from the spec and confirm the clearly human text scores noticeably lower than the clearly AI text. Run `compute_confidence()` across a range of score pairs to confirm all three attribution labels are reachable. Check that `uncertain` scores near 0.525 produce low confidence values and scores near 0.69 produce higher confidence values.

### Milestone 5 — Production layer (labels, appeals, rate limiting, full audit log)

**Spec sections to provide:** Transparency Label Variants (all three written out), Appeals Workflow, Architecture diagram (both submission and appeal flows).

**What to ask for:** `generate_label(attribution, confidence, ai_probability) -> str` function matching the three written-out variants, `POST /appeal` endpoint implementing the workflow steps, Flask-Limiter setup with `10 per minute; 100 per day` applied to the submit route, and an extended audit log schema capturing all required fields.

**How to verify:** Submit inputs that produce all three label categories and confirm the returned label text matches the variants in this spec. Submit an appeal, call `GET /log`, and verify the `appeal_submitted` event appears with `status: under_review` and `appeal_reasoning` populated. Run 12 rapid POST requests and confirm the 11th and 12th return HTTP 429.
