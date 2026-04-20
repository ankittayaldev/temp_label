# Confidence Score Framework — Implementation Report

---

## 1. Background —

Amiyanshu's mail proposed five candidate approaches. We evaluated each for cost/feasibility:

| # | Approach                         | Feasibility                       | Status        |
| - | -------------------------------- | --------------------------------- | ------------- |
| 1 | **Self-consistency (GPT)**       | High cost/time but implementable  | **Done**      |
| 2 | Validator model                  | High cost/time, extra model in loop | Dropped     |
| 3 | Rule validation                  | Already enforced inside the prompt  | Not needed  |
| 4 | **Retrieval grounding score**    | Feasible using existing pgvector    | **Done**    |
| 5 | Token probability                | Not available from the Azure Responses API | Dropped |

Final score shipped = **weighted average (50% GPT + 50% RAG)**.

---

## 2. High-Level Flow

```
          ┌─────────────────────────────────────────────────────────────┐
 claim ──▶│  Hybrid retrieval (RRF)  →  top-k source chunks      │
          └─────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
          ┌─────────────────────────────────────────────────────────────┐
          │  GPT-5.2 verification call (VERIFICATION_SYSTEM_PROMPT)     │
          │   → valid, valid_confidence, replacement_text,              │
          │     replacement_confidence  ← Component A                   │
          └─────────────────────────────────────────────────────────────┘
                                 │
                    (only if invalid & replacement_text present)
                                 ▼
          ┌─────────────────────────────────────────────────────────────┐
          │  Embed replacement_text (Azure text-embedding, 3072-dim)    │
          │  SQL: MAX(cosine_sim) over retrieved point_ids (pgvector)   │
          │   → retrieval_grounding_score  ← Component B                │
          └─────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
          ┌─────────────────────────────────────────────────────────────┐
          │  Weighted average (0.5 · GPT + 0.5 · RAG)  →  score         │
          └─────────────────────────────────────────────────────────────┘
```

Key property: the grounding score adds **one extra embedding call + one SQL query** to the existing verification path. No new DB round-trips beyond that.

---

## 3. Component A — GPT Self-Consistency Score (`replacement_confidence`)

### 3.1 System prompt (verbatim)

`label/processors/prompt_templates/rag_prompt.py` — `VERIFICATION_SYSTEM_PROMPT`:

> You are a regulatory expert verifying marketing claims strictly against the provided PI sources. For additional context, especially for infographic-related content or when a claim appears split improperly on the current page, check previous conversion history. The page image or PDF is available in the first response, USE VISION to read the image and recover the correct context (like surrounding text/graphics or headings) before making the decision. Apply this decision order: (1) normalize equivalent terminology, (2) check factual support, (3) decide valid/invalid. Equivalent wording/normalization: treat "adults" and "patients" as equivalent wording for indication matching. Equivalent mutation wording: treat BRAF V600 and BRAF V600E or V600K as equivalent notation, one form is umbrella wording and the other is subtype wording. If a claim uses qualitative wording where the source requires a specific value, mark it invalid unless the exact value is stated (for example, 'room temperature' is not exact and must be replaced with the specific temperature value from the source). Mark valid=true only if the claim is explicitly and completely supported by the sources, with no assumptions. A claim must remain VALID when the only difference is equivalent wording/notation after normalization. Do not infer missing details and do not introduce stricter wording not required by the source. If invalid, provide a compliant replacement_text derived only from the sources. The replacement_text should correct only unsupported factual parts and preserve the original claim structure and intent. If the claim is valid, replacement_text must be empty and replacement_confidence must be null. Do not rewrite valid claims for style, terminology preference, or notation normalization. Do not automatically add regulatory qualifiers (e.g., 'as detected by an FDA-approved test') unless the original claim explicitly refers to testing/diagnostics. Respond with a strict JSON object containing keys: valid (boolean), valid_confidence (number 0-1), replacement_text (string or empty), replacement_confidence (number 0-1, null when no replacement is provided or valid=true), explanation (string), reasoning (string), sources (list of strings), sources_list (list of integers). pass none in replacement text if not-sufficient documents or irrelevant sources are found.

### 3.2 Output schema (strict JSON)

```json
{
  "valid":                 "boolean",
  "valid_confidence":      "number 0-1",
  "replacement_text":      "string | '' | 'none'",
  "replacement_confidence":"number 0-1 | null",
  "explanation":           "string",
  "reasoning":             "string",
  "sources":               "string[]   (≥1 citation)",
  "sources_list":          "integer[]  (1-based source numbers)"
}
```

`replacement_confidence` is required to be `null` when `valid=true` or no replacement is produced — enforced by JSON Schema (`strict: true`).

### 3.3 Parameters the model weighs for `replacement_confidence`

Not a deterministic formula; these are the signals the prompt + schema push the model toward.

**Source-grounding signals**
1. **Source coverage** — how many of the claim's facts are explicitly stated in `sources_list`.
2. **Directness of support** — verbatim / near-verbatim match vs. paraphrased or inferred.
3. **Number of corroborating sources** — single vs. multiple agreeing sources.
4. **Source specificity** — exact values/percentages vs. qualitative wording.
5. **Relevance of sources** — on-topic (same drug/indication/trial) vs. tangential.

**Claim-vs-replacement signals**
6. **Extent of rewrite** — minimal correction (high confidence) vs. large restructuring (lower).
7. **Preservation of original structure/intent** per the instructions.
8. **Ambiguity in normalisation** — equivalent terminology (BRAF V600 ≡ V600E/K, adults ≡ patients) applied cleanly or not.
9. **Presence of unsupported qualifiers** removed vs. retained.

**Decision-context signals**
10. `valid_confidence` — low confidence in the invalidity decision implies lower confidence in the replacement.
11. **Completeness of context** — whether vision / prior-page context was needed and recoverable.
12. **Conflicts between sources** — agreement vs. contradiction.
13. **Whether `"none"` would be more appropriate** (insufficient sources edge case).

**Output-level signals**
14. **Token-level likelihood** of the generated `replacement_text` under the model.
15. **Schema constraint** — must be `null` if `valid=true` or no replacement given.

### 3.4 Worked example (from `gpt_response.json`)

- `valid = false`, `valid_confidence = 0.9` → high-confidence invalidity.
- Two sources cited (`[1, 3]`).
- Replacement covers **COLUMBUS** incidences but drops **skin papilloma / PHAROS** — partial coverage.
- Result: `replacement_confidence = 0.74` (dragged down from the ~0.9 range by partial coverage).

### 3.5 Limitations

- Self-reported number → subject to model calibration drift.
- No token log-probabilities exposed via the Azure Responses API (Option 5 from the mail was blocked for this reason).

---

## 4. Component B — RAG Retrieval Grounding Score (`retrieval_grounding_score`)

### 4.1 Intuition

The GPT number is the **model's own opinion** about its replacement. The grounding score is an **independent geometric check**: does the replacement actually live close to the same source chunks that were retrieved for the claim? If the replacement drifts into unsupported space, the cosine similarity drops even when GPT is confident.

### 4.2 Algorithm

```
Step 1:  embed(replacement_text)                         → 3072-dim vector (Azure text-embedding)
Step 2:  point_ids ← ids of all chunks passed to GPT     (from the RRF result set)
Step 3:  score = MAX cosine_similarity(embedding, chunk.dense_embedding)
                  over chunks where collection = C
                        AND  index_status = 'completed'
                        AND  point_id ∈ point_ids
Step 4:  clamp to [0, 1]
```

**One DB call.** The entire aggregation is a single SQL with `MAX(...)` — no application-side loop over chunks.

### 4.3 Storage

`label/DAL/models/tables.py` — `RagChunkPg`:

| Column             | Type                   | Role                                           |
| ------------------ | ---------------------- | ---------------------------------------------- |
| `point_id`         | `String` PK            | UUIDv5 of `{id_seed}:{chunk_no}`               |
| `collection`       | `String`               | Tenant / document-set namespace                |
| `dense_embedding`  | `Vector(3072)`         | pgvector column, cast to `halfvec` at query time |
| `index_status`     | `String`               | Only `'completed'` rows are scored             |
| `text`, `headings` | `String`               | Displayed chunk body and section path          |
| `sha256`, `page_number`, `tokens` | —       | Provenance                                     |

### 4.4 SQL (from `label/db.py::calculate_retrieval_grounding_score_pg`)

```sql
SELECT MAX(
  GREATEST(
    0.0,
    LEAST(
      1.0,
      1.0 - (
        dense_embedding::public.halfvec(3072)
        OPERATOR(public.<=>)
        CAST(:query_vector AS public.halfvec(3072))
      )
    )
  )
) AS retrieval_grounding_score
FROM rag_chunks_pg
WHERE collection     = :collection
  AND index_status   = 'completed'
  AND point_id       IN :point_ids;
```

Notes:

- `<=>` is pgvector's **cosine distance** operator; `1 − distance = cosine similarity`.
- Cast to `halfvec(3072)` keeps the comparison fast on the same storage format used by the **HNSW index**.
- `GREATEST/LEAST` clamp guarantees the output stays in `[0, 1]` (protects against floating-point drift).
- `IN :point_ids` — bound via SQLAlchemy `expanding=True` bindparam; post-filtering over the exact set GPT saw, so we score against the **same evidence** GPT used.

### 4.5 When the score is computed

In `_verify_single_claim_hybrid_from_results` (paraphrased):

```python
should_embed_replacement = (
    embed_client is not None
    and isinstance(replacement_text, str)
    and replacement_text.strip() != ""
    and replacement_text.strip().lower() != "none"
    and replacement_confidence is not None
)
```

So the grounding score is **only** computed for invalid claims that produced a real replacement. Valid claims (`replacement_text == ""`) and `"none"` fallbacks are skipped — matches GPT's schema rule that `replacement_confidence` is `null` there.

### 4.6 Cost profile

- +1 embedding call (replacement text — usually 20–60 tokens).
- +1 SQL round-trip (aggregation over 5–15 `point_ids`, HNSW-backed storage).
- Wrapped in try/except — any failure falls back to `replacement_confidence` as the final score, so grounding is a **strictly additive** signal.

---

## 5. Weighted Combination — Final `score`

```python
# label/routes/rag.py (called from _verify_single_claim_hybrid_from_results)
weighted_average_score = _calculate_weighted_average_score(
    replacement_confidence,        # GPT self-consistency
    retrieval_grounding_score,     # RAG cosine grounding
)
```

Formula:

```
score  =  0.5 · replacement_confidence  +  0.5 · retrieval_grounding_score
```

### 5.1 Fallback rules

| Condition                                                   | Returned `score`                  |
| ----------------------------------------------------------- | --------------------------------- |
| `valid = true`                                              | `null` (no replacement, no score) |
| `replacement_text ∈ {"", "none", null}`                     | `null`                            |
| GPT returns number, RAG fails/disabled                      | `replacement_confidence`          |
| Both present                                                | `0.5·GPT + 0.5·RAG` (4-dp rounded) |

### 5.2 Why 50/50

- **GPT** knows *semantics* — it knows whether the rewrite preserves intent and removes only unsupported parts.
- **RAG** knows *geometry* — it knows whether the rewrite sits on top of the retrieved evidence.
- Neither dominates; equal weighting is the simplest defensible prior before we have labelled data to fit a better weight.
- Both factors are on the same `[0, 1]` scale, so the average is directly interpretable as a confidence number.

### 5.3 What this buys us operationally

- Reviewers can sort the queue by `score` → lowest first = most likely to need human attention.
- Divergence between GPT and RAG is a diagnostic signal:
  - **GPT high, RAG low** → model is confident in a rewrite that drifts off-evidence (possible hallucination or over-generalisation).
  - **RAG high, GPT low** → replacement is lexically close to sources but the model is hedging (often because `valid_confidence` was only moderate).

---

## 6. Sample-data Comparison — OLD vs. NEW

All 21 claims from `sample_verified_json.json`. `Δ = NEW − OLD`.

| Claim # | Valid | OLD (`replacement_confidence`, GPT only) | RAG (`retrieval_grounding_score`) | **NEW (`score` — weighted avg)** | Δ        |
| ------: | :---: | ---------------------------------------: | --------------------------------: | -------------------------------: | -------: |
|       1 | false |                                     0.70 |                            0.6983 |                       **0.6991** |  −0.0009 |
|       2 | false |                                     0.72 |                            0.8224 |                       **0.7712** |  +0.0512 |
|       3 | false |                                     0.74 |                            0.8024 |                       **0.7712** |  +0.0312 |
|       4 | false |                                     0.83 |                            0.7385 |                       **0.7842** |  −0.0458 |
|       5 | true  |                                     null |                              null |                         **null** |        — |
|       6 | false |                                     0.74 |                            0.8807 |                       **0.8103** |  +0.0703 |
|       7 | false |                                     0.74 |                            0.8250 |                       **0.7825** |  +0.0425 |
|       8 | false |                                     0.72 |                            0.7258 |                       **0.7229** |  +0.0029 |
|       9 | false |                                     0.00 |                            0.0000 |                       **0.0000** |   0.0000 |
|      10 | true  |                                     null |                              null |                         **null** |        — |
|      11 | false |                                     0.77 |                            0.7189 |                       **0.7445** |  −0.0255 |
|      12 | false |                                     0.78 |                            0.8110 |                       **0.7955** |  +0.0155 |
|      13 | true  |                                     null |                              null |                         **null** |        — |
|      14 | false |                                     0.64 |                            0.5124 |                       **0.5762** |  −0.0638 |
|      15 | false |                                     0.90 |                            0.9651 |                       **0.9325** |  +0.0325 |
|      16 | false |                                     0.74 |                            0.9467 |                       **0.8434** |  +0.1034 |
|      17 | false |                                     0.78 |                            0.8559 |                       **0.8179** |  +0.0379 |
|      18 | false |                                     null |                              null |                         **null** |        — |
|      19 | false |                                     0.80 |                            0.6980 |                       **0.7490** |  −0.0510 |
|      20 | false |                                     0.73 |                            0.8224 |                       **0.7762** |  +0.0462 |
|      21 | false |                                     0.78 |                            0.7821 |                       **0.7811** |  +0.0011 |

### 6.1 Reading the table

- **Valid claims (5, 10, 13):** no replacement → `null` everywhere. Correct per schema.
- **Claim 18 (invalid, no replacement):** GPT returned `"none"` (insufficient/irrelevant sources) → grounding skipped → `score = null`. Correctly flagged for manual attention.
- **Claim 9:** GPT self-assigned `replacement_confidence = 0`, RAG also `0` → perfect agreement that this replacement is unreliable.
- **Largest positive Δ — Claim 16 (+0.1034):** GPT was conservative (0.74), but the replacement was **very close** to retrieved CYP3A4-inducer language (RAG = 0.9467). New score surfaces the reviewer's confidence that the rewrite is well-grounded.
- **Largest negative Δ — Claim 14 (−0.0638):** GPT was moderately confident (0.64), but the replacement drifted off-evidence (RAG = 0.5124) — new score correctly flags it for closer review than GPT alone suggested.
- **Near-zero Δ — Claims 1, 8, 21:** the two signals agree, confidence is unchanged; weighting doesn't have to "do anything" in these cases and that is fine.

### 6.2 Aggregate observations (16 claims with both scores, excluding claim 9's 0/0 edge)

- **RAG > GPT** in 11/16 cases → model tends to be slightly more conservative than raw geometric grounding.
- **RAG < GPT** in 5/16 cases → when the model rewrites with more paraphrase, geometric distance from chunks grows.
- Average absolute Δ ≈ 0.04 — **the weighted score is a meaningful adjustment, not noise**, without being destabilising.

---

## 7. What Changed

| Aspect                  | Before                                  | After                                                      |
| ----------------------- | --------------------------------------- | ---------------------------------------------------------- |
| Confidence exposed      | `replacement_confidence` (GPT self-report) | `score` = 50/50 weighted avg; **plus** raw GPT and RAG kept for audit |
| Independence            | Single source (GPT)                     | Two independent signals (GPT semantics + pgvector geometry) |
| Cost per claim          | 1 chat call                             | 1 chat call + 1 embedding + 1 SQL                          |
| Reviewer queue sortable? | Coarse (GPT only)                       | Finer — divergence between signals is itself a flag        |
| Schema coverage         | `replacement_confidence` only           | `replacement_confidence`, `retrieval_grounding_score`, `score` |

---

## 8. One-line Summary

> *For every invalid claim we now emit two independent, bounded-in-[0,1] confidence numbers — GPT's self-assessed `replacement_confidence` and a pgvector `retrieval_grounding_score` — and expose their 50/50 weighted average as `score`. Same pipeline, one extra embedding + one extra SQL, no new infra. Sample run on 21 claims shows the combined score is a meaningful, non-destabilising adjustment to the GPT-only baseline.*
