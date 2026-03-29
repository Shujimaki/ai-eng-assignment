# Agent Trajectory — Recipe Enhancement Pipeline

**Tool:** Google Antigravity (Gemini-based in-IDE agent)  
**Session Duration:** ~45 minutes  
**Date:** 2026-03-29  
**Status:** Complete — all 6 bugs fixed, pipeline fully operational

---

## Overview

This file documents the full agent-assisted debugging and fix session for the Recipe Enhancement Pipeline take-home assessment. The workflow followed was: I reviewed the codebase independently first, identified all bugs, then directed the agent to fix them one at a time with confirmation gates between each fix.

---

## Pre-Session: Independent Code Review

Before starting the agent session, I manually reviewed the full codebase and identified 6 bugs across the pipeline. The agent was not involved in the diagnosis — it was given the findings and told to implement fixes in priority order.

---

## Session Log

### Opening Message — Bug Report

I opened the session by presenting all 6 bugs I had already identified, with the instruction to start with Bug 2 (the most critical), show a diff, and wait for my signal before moving on. Each fix was to get its own conventional commit.

**Bugs presented:**

- **Bug 1 (High):** `process_single_recipe()` calls `extract_single_modification()` which does `random.choice()` on modification reviews. Only one review ever gets processed per recipe regardless of how many exist. The cookies recipe has 5 modification reviews and we were randomly picking one. The stale files in `data/enhanced/` showed two modifications each with a `confidence_score` field that didn't exist in the current model, confirming this was a regression.

- **Bug 2 (Critical):** In `recipe_modifier.py`, `apply_edit()` uses fuzzy matching to find the closest line, but then calls `original_text.replace(edit.find, ...)` which is an exact string match. The fuzzy match finds the right line but the replace silently does nothing. A `ChangeRecord` still gets created falsely claiming a change was made.

- **Bug 3 (Medium):** When a recipe has no reviews with `has_modification=True`, the pipeline returns `None` and logs it as a processing failure. That's misleading — it's not broken, it just has no data. 4 out of 11 recipes hit this path every time.

- **Bug 4 (High):** The scraper extracts `featured_tweaks` as a separate field, but `parse_reviews_data()` in `pipeline.py` only reads from `reviews`. These are the highest-signal reviews on AllRecipes — curated, photo-highlighted — and the pipeline was discarding them entirely. Related to Bug 1 since both affect what reviews get processed.

- **Bug 5 (Medium):** `test_pipeline.py` uses `../data` which only works from inside `src/`. `pipeline.py` defaults `output_dir` to `"data/enhanced"` relative to CWD. They assumed different working directories. Also, two `data/enhanced/` directories existed: one at project root with stale schema-incompatible files, one at `src/data/enhanced/` as current output. Needed to consolidate to root-level.

- **Bug 6 (Low):** README referenced 6 recipes but `data/` actually has 11. 4 will always have no modifications (Bug 3), 7 are processable.

---

### Fix 1 — Bug 2: Replace Operation

**Agent actions:** Read `recipe_modifier.py`, `tweak_extractor.py`, `prompts.py`, and an existing enhanced output file for context. Made a 2-line edit in `apply_edit()` to replace the entire fuzzy-matched line with `edit.replace` directly instead of calling `original_text.replace(edit.find, ...)`.

**Commit:** `fix(recipe_modifier): replace entire fuzzy-matched line during replace operation`

**My signal:** Confirmed, moved on.

---

### Fix 2 — Bugs 1 + 4: Process All Reviews + Include featured_tweaks

**Agent actions:** Read `pipeline.py`, `models.py`, `enhanced_recipe_generator.py`, and the cookies recipe JSON. Made coordinated changes across three files:

- **`pipeline.py`** — `parse_reviews_data()` now reads both `reviews` and `featured_tweaks`, concatenates them, and deduplicates using a `seen_texts` set. `process_single_recipe()` now loops through all `modification_reviews` instead of calling `extract_single_modification()`, collects all successful extractions into `modifications_and_reviews`, and calls `apply_modifications_batch()` to apply them all sequentially.

- **`models.py`** — Restored the missing `confidence_score` field to `ModificationObject`, `ModificationApplied`, and `EnhancementSummary`. These existed in the stale `data/enhanced/` output files but had been removed from the models (a regression).

- **`enhanced_recipe_generator.py`** — Updated `generate_enhanced_recipe()` to accept a list of `(modification, review, change_records)` tuples instead of single objects. Updated `calculate_enhancement_summary()` to average confidence scores across all applied modifications.

I asked the agent for a detailed line-by-line explanation before confirming, since two bugs were fixed in one commit and the changes touched three files.

**Commit:** `fix(pipeline): process all available modification reviews and handle featured_tweaks`

**My signal:** Confirmed after review, moved on.

---

### Clarification — Variable Naming

Before proceeding to Bug 3, I asked the agent to clarify the difference between `modification_reviews` and `modifications_and_reviews` in `pipeline.py` to make sure I understood the logic correctly.

- `modification_reviews` — input list of raw `Review` objects filtered to those with `has_modification=True`
- `modifications_and_reviews` — output list of `(ModificationObject, Review)` tuples after LLM extraction, keeping source review paired with structured edit for attribution in Step 3

Confirmed, moved on.

---

### Fix 3 — Bug 3: Graceful Handling of Zero-Modification Recipes

**Agent actions:** Read `pipeline.py`. Updated `process_single_recipe()` to log an info message and proceed gracefully when no modification reviews are found, rather than returning `None` and logging a failure. The recipe still gets saved as an `EnhancedRecipe` with zero modifications applied.

**Commit:** `fix(pipeline): gracefully handle recipes with zero modification reviews`

**My signal:** Confirmed, moved on.

---

### Fix 4 — Bugs 5 + 6: Path Consistency + Output Directory + README

Combined since they were closely related.

**Agent actions:**

- **`pipeline.py`** — `output_dir` now resolves to an absolute path anchored at project root using `Path(__file__).resolve().parent.parent.parent`. No longer dependent on CWD.

- **`test_pipeline.py`** — Replaced `../data` relative paths with `Path(__file__).resolve().parent.parent / "data"` so the script works regardless of where it's invoked from.

- Cleared the stale `src/data/enhanced/` directory. All output now goes to `data/enhanced/` at project root.

- **`README.md`** — Updated to reflect 11 recipes in `data/`, correct output directory, and accurate run instructions.

Ran `uv run python3 src/test_pipeline.py all` from project root to verify everything worked with the new paths.

**Commit:** `fix: resolve conflicting output directories and update readme counts`

---

### Validation Error Mid-Run

During the full pipeline run, I noticed validation errors in the logs for one recipe:

```
WARNING: Validation error: Input should be 'ingredient_substitution', 'quantity_adjustment',
'technique_change', 'addition' or 'removal' [input_value='quantity_adjustment|ingredient_substitution']
```

The LLM returned a pipe-delimited string combining two modification types, which Pydantic correctly rejected. The pipeline retried 3 times, failed all attempts, and then gracefully fell back to proceeding without enhancement — exactly the behavior introduced in Bug 3's fix.

I confirmed with the agent that this was intentional, not a new bug.

---

### Final Verification

After all fixes were committed, I asked the agent to do a full sweep and run `uv run python3 src/test_pipeline.py all`.

**Result:** 11/11 recipes processed, 0 failures, all output in unified `data/enhanced/` directory. All fixes working in harmony with no lingering runtime or logic errors.

---

## Summary of Changes

| Bug | Severity | Files Changed | Commit |
|-----|----------|---------------|--------|
| Bug 2: Silent replace after fuzzy match | Critical | `recipe_modifier.py` | `fix(recipe_modifier): replace entire fuzzy-matched line during replace operation` |
| Bug 1: Only one random review processed | High | `pipeline.py`, `models.py`, `enhanced_recipe_generator.py` | `fix(pipeline): process all available modification reviews and handle featured_tweaks` |
| Bug 4: `featured_tweaks` ignored | High | `pipeline.py` | (same commit as Bug 1) |
| Bug 3: Zero-mod recipes logged as failures | Medium | `pipeline.py` | `fix(pipeline): gracefully handle recipes with zero modification reviews` |
| Bug 5: Inconsistent paths, split output dirs | Medium | `pipeline.py`, `test_pipeline.py` | `fix: resolve conflicting output directories and update readme counts` |
| Bug 6: README says 6 recipes, actually 11 | Low | `README.md` | (same commit as Bug 5) |

---

## Key Decisions

**Why Bug 2 first?** Everything else depended on the replace operation working correctly. Fixing it first meant subsequent test runs would give accurate results.

**Why Bugs 1 and 4 together?** They were two sides of the same problem — what reviews feed into the pipeline. Fixing Bug 1 without Bug 4 would still silently discard the highest-signal data. The fix in `parse_reviews_data()` naturally covered both.

**Why OpenRouter instead of OpenAI?** The original codebase required an OpenAI API key which requires a paid account. OpenRouter provides an OpenAI-compatible API, so the swap was two lines (`base_url` and `api_key` env var) with zero changes to the rest of the code.

**Why keep `confidence_score`?** The stale output files in `data/enhanced/` had it. The current models didn't. Restoring it was the right call to maintain schema consistency and preserve a useful signal for downstream consumers.
