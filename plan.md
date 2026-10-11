# Implementation Plan: Faithfulness checker scores claims unsupported when the context uses different words

**Issue:** https://github.com/codepath/pathreview-ai301-fa26-s1/issues/59#issuecomment-5984246709

## Diagnosis

The reported bug occurs when when feedback contains punctuation attached to key skill words (for example "Python, JavaScript, and Docker experience") and the context uses the same words without that punctuation. The checker scores the claim as unsupported even though every skill is present in the context.`test_multiple_context_chunks` returns `0.0` instead of a score above 0.5.
The likely root cause is the tokenization and threshold logic in `FaithfulnessChecker._is_supported` in `rag/evaluator/faithfulness_checker.py`.
The claim and context are tokenized with a bare `.split()`, so punctuation stays attached to words (`"python,"` ≠ `"python"`, `"javascript,"` ≠ `"javascript"`). Those tokens never match the context. The stop-word filter then removes `"and"`, which leaves a single meaningful overlap (`docker`). That is below the hard-coded threshold of 2 (`len(meaningful_overlap) >= 2`), so the claim is rejected. The threshold has no fallback for short claims that have few content words. A secondary contributor is `_extract_claims`, which drops fragments of 10 characters or fewer, so very short claims such as "Knows Rust" never reach scoring.

### Reproduction Evidence

From my Unit 2 reproduction report:

> **Steps:** Ran `pytest tests/unit/test_faithfulness_checker.py::TestFaithfulnessChecker::test_multiple_context_chunks --runxfail -v` to check details specifically for the test_multiple_context_chunks function.

> **Expected:** The test provides context describing Python, JavaScript, and Docker experience. The faithfulness checker should recognize the feedback describing those same skills as supported, resulting in a score greater than 0.5.

> **Actual:** The test failed because the checker returned a score of `0.0`: `E assert 0.0 > 0.5`. The captured output also reported: `claims_count=1 score=0.0 supported_count=0`.

Control case:

> The same claim with the commas removed ("The developer has Python JavaScript and Docker experience") against the same context has overlap `['and', 'docker', 'python', 'javascript']`, and `_is_supported` returns `True`. Separately, `test_feedback_fully_supported_by_context` ("...strong Python skills and experience with Django.") passes, because its matching words carry no trailing punctuation.

These observations support the diagnosis because the only difference between the failing and passing inputs is the attached punctuation. The context still supports the claim, so the matching logic, not the data, causes the false negative. With the comma-attached tokens the overlap is `['and', 'docker']`, which leaves one meaningful token after stop-word removal and falls under the threshold of 2. With punctuation stripped, the overlap is four tokens and the claim is supported.

## Scope

**In scope:**

- Tokenize claims and context so punctuation does not prevent word matches (`_is_supported`).
- Make the support threshold account for short claims that have few content words, so they can be marked supported.
- Lower the minimum claim length in `_extract_claims` so short claims such as "Knows Rust" are scored instead of silently dropped.
- Remove the `xfail` markers from the tests this fix makes pass (`test_multiple_context_chunks`, `test_multiple_claims_varying_support`).

**Out of scope:**

- `test_partial_support_returns_middle_score`. It checks a single-sentence claim, which can only score 0.0 or 1.0, so it needs clause-level claim splitting. That is a larger design change than #59 (see Unknowns). Its `xfail` marker stays.
- Semantic or synonym matching (embeddings, stemming, LLM-based checks), and changes to callers of `FaithfulnessChecker.check` or its 0.0-1.0 return contract.

## Files to Change

- `rag/evaluator/faithfulness_checker.py`: in `_is_supported`, tokenize with `re.findall(r"\w+", text.lower())` and replace the fixed `>= 2` threshold with a length-aware one (1 meaningful overlap for claims with 3 or fewer content words, otherwise 2). In `_extract_claims`, lower the `len(s.strip()) > 10` filter so short claims are kept.
- `tests/unit/test_faithfulness_checker.py`: remove the `xfail` markers from `test_multiple_context_chunks` and `test_multiple_claims_varying_support`. Add tests for punctuation-attached tokens, a short claim with one matching content word, and a short claim that is correctly unsupported. Keep the `xfail` on `test_partial_support_returns_middle_score`.

## Implementation Approach

1. Locate `FaithfulnessChecker._is_supported` and `_extract_claims`, which are responsible for punctuation-bound tokens, the fixed threshold and the dropped short claims.
2. Modify `_is_supported` so tokens come from `re.findall(r"\w+", ...)` on the lowercased text. Compute the claim's content words (tokens minus stop words) and require 1 meaningful overlap if there are 3 or fewer content words, otherwise 2. Lower the claim-length filter in `_extract_claims` (for example keep any fragment of 3+ characters).
3. Preserve existing behavior: the 0.0 return for empty feedback or empty context, the 0.5 default when no claims are extracted, the 10-claim cap, the stop-word list, and the `supported / len(claims)` formula. Long claims must still need 2 meaningful overlaps, so one shared generic word (for example "developer") does not make an unrelated claim supported.
4. Add or update tests to verify the corrected behavior. The markers are `xfail(strict=True)`, so the fixed tests must have their markers removed in the same change or they will fail as XPASS.

## Test Plan

1. **Original reproduction**
   - Input/steps: `pytest tests/unit/test_faithfulness_checker.py::TestFaithfulnessChecker::test_multiple_context_chunks --runxfail -v` (feedback "The developer has Python, JavaScript, and Docker experience." with the three context chunks).
   - Expected after fix: the test passes with a score of 1.0 (> 0.5) and `supported_count=1` for the one extracted claim.

2. **Control case**
   - Input/steps: `test_feedback_with_no_support_in_context` ("This developer is an expert in Rust systems programming." against Python/JavaScript context), plus `test_feedback_fully_supported_by_context`.
   - Expected after fix: the unsupported feedback still scores below 0.5 (the only shared content word is "developer", below the 2-word threshold for a 6-content-word claim), and the fully supported feedback still scores above 0.5.

3. **Regression tests**
   - Run: `.venv/bin/python -m pytest tests/unit/test_faithfulness_checker.py -v -rx`, then `.venv/bin/python -m pytest tests/unit -q`.
   - Expected: all tests pass; `test_multiple_claims_varying_support` scores 2/3 (about 0.67, inside 0.2-0.8); `test_partial_support_returns_middle_score` remains `XFAIL`; no other unit test regresses.

## Risks and Unknowns

**Risks:**

- Lowering the threshold to 1 for short claims could raise false positives, for example a short claim sharing one generic word with the context.
- Lowering the claim-length filter changes scores for existing feedback (more claims counted), which could shift faithfulness scores that callers or evaluation thresholds rely on.

**Unknowns:**

- Whether maintainers want `test_partial_support_returns_middle_score` fixed under #59. The prototype shows tokenization and threshold changes alone leave it at 0.0 because it is one sentence. It would need clause-level splitting (on commas or "and").
- The right cut-off for a "short claim" (3 content words is my prototype's choice) and whether words like `with` and `has` belong in the stop-word list.
- Which other modules consume the faithfulness score (for example thresholds in `rag/` or `agent/`) and whether any depend on the current scoring.

## Deviations
