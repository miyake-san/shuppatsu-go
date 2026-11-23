---
name: shp-doubt-check
description: Adversarial second look at a non-trivial Go decision before it stands - state the claim, list assumptions, attack them, test the cheapest ones, and record a verdict. Use for high-stakes or irreversible changes (auth, data migrations, permissions, concurrency, money), unfamiliar code, confident-but-unverified conclusions, and before approving a design or merging a risky diff.
license: MIT
metadata:
  kit: shuppatsu-go
  version: "1.3.0"
  inspired_by: "https://github.com/addyosmani/agent-skills (doubt-driven-development, MIT)"
---

# shp-doubt-check

Confidence is cheap; verification is not. For decisions that are expensive to reverse, run a short, structured attempt to prove yourself wrong **before** presenting the result.

## When to run it

Run it when any of these hold: security or auth logic, access-control policies, data migration or deletion, concurrency or locking, money or quotas, public API or event contracts, unfamiliar codebase areas, or a conclusion you reached without running anything. Skip it for trivial, reversible edits.

## Procedure

1. **Claim.** State the decision in one sentence ("This handler is idempotent because ...").
2. **Assumptions.** List every assumption it rests on (inputs, ordering, who calls it, platform behaviour, library semantics, Go version).
3. **Attack.** For each assumption ask: *what if it is false?* Use these lenses:
   - **Inputs:** nil, empty, huge, duplicated, malicious, wrong encoding, wrong time zone.
   - **Concurrency:** two callers at once, retry after partial success, goroutine outlives its context.
   - **Failure:** dependency slow, down or returning a partial response; process killed mid-write.
   - **Scale and cost:** 1000x volume, unbounded growth, hot partition, throttling.
   - **Security:** who can call this, what can they reach, what is logged.
   - **Time:** clock skew, retries days later, function or request timeout versus downstream timeout.
4. **Test the cheapest ones.** Prefer evidence over argument: a failing test, `go test -race`, `go doc` to confirm semantics, a read of the official docs, a dry run. Label every assumption *verified*, *unverified* or *refuted*.
5. **Fresh eyes when stakes are high.** Re-read the diff as if reviewing someone else's, or hand it to the read-only `shp-go-reviewer` agent with only the code and the claim (not your reasoning), so it is not anchored on your conclusion.
6. **Verdict.** `holds`, `holds with changes` (list them) or `does not hold` (say what to do instead).

## Output

```markdown
## Doubt check: <decision>
Claim: ...
| # | Assumption | Attack | Evidence | Status |
|---|---|---|---|---|
| 1 | ... | ... | test / doc / run | verified / unverified / refuted |
Verdict: holds | holds with changes | does not hold
Residual risk: <what remains unverified and why it is acceptable>
```

## Anti-rationalization

| Excuse | Rebuttal |
|---|---|
| "I am confident" | Confidence is not evidence. Name the check that would change your mind. |
| "It would take too long" | Compare with the cost of a production incident or a rollback. Check only the riskiest assumption if time is short. |
| "The tests pass" | Tests prove the cases written, not the assumptions behind them. |
| "Reviewers will catch it" | Reviewers see the diff, not your unstated assumptions. |

## Verification gate

At least one assumption was actually tested (not just argued), unverified ones are listed as residual risk, and the verdict is explicit. Do not downgrade a refuted assumption to "minor".

## Sources

- Go memory model and data races: https://go.dev/ref/mem , https://go.dev/doc/articles/race_detector
- Idempotency and safe retries (Stripe engineering write-up): https://stripe.com/blog/idempotency
- Pre-mortem technique: https://hbr.org/2007/09/performing-a-project-premortem
- OWASP Top Ten: https://owasp.org/www-project-top-ten/

## Inspired by

Claim-assumption-attack-verdict loop inspired by `doubt-driven-development` in addyosmani/agent-skills (MIT). Lenses and Go mapping are original. See `THIRD_PARTY_NOTICES.md`.
