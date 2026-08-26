# BUG Mode — Symptom-First Defect Investigation

This file is the BUG-mode protocol. SKILL.md's mode router sends you here on an error, failure,
or regression request — run the protocol from this text, not from memory of it. The core
invariants (evidence before verdict, clean result is valid, maker ≠ checker) and the
reproduction rules ("Reproduction beats reading") live in SKILL.md and apply here unchanged.

Priority stack — this is not the audit stack:

```text
Correctness > Reproduction > Root Cause > Minimal Fix > Simplicity
```

BUG mode starts from the **symptom**, not from suspicious patterns:

```text
symptom → failing path → reproduction → violated invariant
        → first point of violation → minimal root-cause fix
        → regression test → independent verification
```

## Bug Suspects

| Tag | Defect | Typical shapes |
|---|---|---|
| `[MISMATCH]` | Contract mismatch between caller and callee | Optional consumed as required · bool return ignored · async not awaited · argument order · changed return schema, old caller · exception contract |
| `[EDGE]` | Boundary / edge case | off-by-one · empty collection · zero/negative · first/last/single element · max/min · missing key · EOF · empty response · pagination boundary |
| `[STATE]` | Invalid state transition | DONE→RUNNING · deleted object reused · status set before the operation succeeds · partial failure leaves an impossible state |
| `[ORDER]` | Initialization / call ordering | use before init · close before flush · commit before validation · event published before persistence · cleanup too early |
| `[LEAK]` | Resource / transaction / context leak | file handles · DB sessions · transactions · locks · temp files · connections · async tasks · context variables |
| `[RACE]` | Concurrency suspect — **candidate-only** | shared mutable state · check-then-act · non-atomic update · double init · unsynchronized cache mutation · concurrent file write |
| `[TYPE]` | Type / nullability / schema assumption | nullable treated as required · string/int mismatch · JSON shape · timezone · enum drift · schema vs DB nullable |
| `[DRIFT]` | Two definitions that must stay in sync, diverged | API response vs client model · DB schema vs ORM · env config vs code · protobuf/OpenAPI vs implementation · migration vs runtime model |

Suspect-specific verification:

- `[MISMATCH]`: read the callee's actual contract, read the caller's assumption, inspect **all**
  affected callsites, reproduce where possible.
- `[EDGE]`: the happy path passing is **not** evidence about the edge path.
- `[STATE]`: the finding must name expected transition vs actual transition.
- `[LEAK]`: never verdict on pattern alone — confirm actual ownership and the cleanup path.
- `[RACE]`: never confirmed by static grep alone. Pattern only → SUSPECT. Deterministic proof or
  runtime reproduction → BUG. Unable to verify → UNVERIFIED.

## Workflow

### B0 — Capture the symptom

```yaml
bug_symptom:
  expected: "<expected behavior>"
  actual: "<actual behavior>"
  trigger: "<input/event>"
  environment: "<relevant environment>"
```

Missing information is not a blocker — start collecting what evidence exists.

### B1 — Reproduce

Build a failing reproduction **first**. **Do not patch before understanding the failure.**
Preference order: existing failing test → minimal new test → minimal isolated script →
temporary copy outside the working tree → controlled stub/mock. Never destroy production data
or mutate the working tree to reproduce (SKILL.md's `reproduce:` rules apply).

### B2 — Minimize the failing case

Strip unrelated variables, shrink the input, narrow the path — until what remains is the
invariant in miniature.

### B3 — Trace data and control flow

Do not patch at the symptom site by reflex. Trace backward
(`failure site ← caller ← transformed value ← state transition ← boundary input`)
or forward when needed
(`input → normalization → validation → transformation → side effect → failure`).

### B4 — State the invariant

Before any patch, write the broken invariant in one sentence:

```text
"A required patent ID must never become None after normalization."
"A failed DB write must not transition the job to COMPLETED."
"Every external HTTP call must terminate within the operation deadline."
```

If you cannot state the invariant, you do not yet understand the root cause.

### B5 — Locate the FIRST invariant violation

Fix the first incorrect state, not the last visible exception.

```text
Bad:    foo().strip() raises on None → add `or ""` before .strip()
Better: find out why required foo() returned None → fix the contract/source bug
```

(This is SKILL.md item 12's rule — "fix the source, not the symptom" — applied to debugging.)

### B6 — Minimal root-cause fix

Was the request a question ("why does this fail?") → **present the diagnosis and proposed fix,
and get approval before editing**, per DEEP's Step 6 rule. Was it a fix request ("fix this bug")
→ apply directly, with AUTO's always-surface exceptions (public API, schema, migration,
destructive operation, auth semantics, behaviour change beyond the bug).

The smallest change that restores the invariant. Forbidden in the same change: unrelated
refactoring, future-proof abstraction, large cleanup, new frameworks, broad rewrites.
AUTO-mode guards (`[STRANGER]` `[PROPHET]` `[TWIN]`) apply to the fix you write.

### B7 — Regression test

Leave the reproduction behind as a test that proves: **before fix → fails, after fix → passes**.
The existing suite passing is not, by itself, proof of the fix.

### B8 — Independent verification

A material bug fix goes through Pass 3 — read `references/pass3-verification.md` and dispatch
per that protocol. Hand the checker only: the symptom, the reproduction, the diff, the claimed
invariant, and the verification commands — **not your conclusion**. Ask it to re-run the
original reproduction, attack the fix and nearby edge cases, validate the regression test, and
check for unrelated behaviour regressions. Verdicts: `APPROVE` / `REQUEST_CHANGES` / `UNVERIFIED`.

## Verdict policy

```yaml
bug_verdict:
  runtime_reproduced:               BUG
  failing_test_proves_behavior:     BUG
  deterministic_contract_violation: BUG   # provable from code alone, e.g. types cannot match
  pattern_only:                     SUSPECT
  plausible_but_unverified:         SUSPECT
  reproduction_impossible:          UNVERIFIED
```

**"This code looks dangerous" ≠ "this is a bug."** Calling something a bug requires evidence.
SUSPECT and UNVERIFIED are honest verdicts — never inflate them to BUG to make a report look
decisive, and never drop them from the report to make it look clean.

How this maps onto SKILL.md's audit statuses: in BUG mode, `SUSPECT` plays `[QUESTION]`'s role,
and a finding-level `UNVERIFIED` subsumes `[UNREPRODUCED-HIGH]` — **record the severity and keep
it**, and for a HIGH claim blocking remains the user's call, exactly as in the audit modes. A
finding-level `UNVERIFIED` is a statement about one claim; a Pass 3 **checker's** `UNVERIFIED`
is a statement about a whole review, with its own retry-once-then-escalate obligation — do not
mix the two in one sentence.

## Regressions — use git history

When regression is suspected, do not guess from the current code alone. Ask **"when did this
become wrong?"**:

```bash
git log --oneline -- <path>
git blame <path>
git diff <known-good>..HEAD -- <path>
git show <commit> -- <path>
```

Compare against a known-good revision when one exists, and read how the recent change shifted
the invariant.

## Report additions

BUG-mode findings extend the SKILL.md report shape with these fields:

```yaml
- id: SC-001
  tag: "[MISMATCH]"
  verdict: BUG            # BUG | SUSPECT | UNVERIFIED
  severity: HIGH
  location: "service.py:82"
  title: "Optional return consumed as required string"
  invariant: "lookup_user() result must be validated before string operations"
  evidence: |
    lookup_user() returns Optional[str].
    Caller at service.py:82 invokes .strip() without checking None.
    Reproduced with a missing user ID: AttributeError.
  root_cause: "Caller assumes a stronger contract than the callee provides."
  smallest_fix: "Handle the missing user at the contract boundary."
  regression_test: "tests/test_service.py::test_missing_user"
```

`verdict_counts` gains `bug: N, suspect: N, unverified: N`.
