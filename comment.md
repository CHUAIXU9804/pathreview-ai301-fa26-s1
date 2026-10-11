I reproduced the faithfulness checker issue in #59 and documented the results in my earlier repro comment.

The reproduction showed:

> **Expected:** The checker should recognize feedback mentioning Python, JavaScript, and Docker as supported by context containing those same skills, producing a score above 0.5.
>
> **Actual:** `test_multiple_context_chunks` returned `0.0` instead (`E assert 0.0 > 0.5`), with `claims_count=1 score=0.0 supported_count=0`.

Based on the reproduction and code inspection, the likely cause is the tokenization and fixed overlap threshold in `FaithfulnessChecker._is_supported`. Using `.split()` retains punctuation on tokens such as `python,`, preventing matches with `python` in the context. After stop-word filtering, the remaining overlap falls below the required two matching words.

The proposed fix is to normalize tokenization and use a length-aware overlap threshold in `rag/evaluator/faithfulness_checker.py`. I'll also adjust `_extract_claims` to retain short claims and update the affected tests in `tests/unit/test_faithfulness_checker.py`. The change will remain limited to lexical matching and claim extraction, without introducing semantic matching or changing the checker's return contract.

I'll verify the fix by rerunning the original reproduction, checking supported and unsupported control cases, and running the faithfulness checker and broader unit tests. I expect the original failing test to score `1.0`, while unsupported claims should remain unsupported.

One remaining question is whether the partial-support test, which would require clause-level splitting, belongs within #59. I'm leaving that larger change out of scope for now. The short-claim threshold also needs validation to avoid introducing false positives.

I'd appreciate any feedback on this approach before implementation.
