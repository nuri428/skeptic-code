# Pass 3 — Independent Verification: the full protocol

This file is Step 8 of skeptic-code's workflow. SKILL.md's Step 8 sends you here **before you
dispatch a checker** — run the protocol from this text, not from memory of it. The trigger table
(when Pass 3 is mandatory, and when it takes two lanes) lives in SKILL.md Step 8.

## What makes a checker independent

All four must hold:

1. **Fresh context** — it starts from the diff and the goal, not from this audit's reasoning.
2. **Did not author the change** — a past maker is permanently disqualified as that change's checker.
3. **Not a fork of the maker** — a continuation of this session carries the assumptions under test.
4. **Read-only** — if it edits source it has joined the maker set and its verdict is void.

Independence is about what the checker reproduces, not about which model ran it. Nothing enforces
it mechanically. That is exactly why the honest report when no checker can be
reached is `UNVERIFIED`, not a verdict you were in no position to issue.

## Dispatch

Spawn a fresh general-purpose subagent (Task tool, `subagent_type: general-purpose`, or any
equivalent independent context). Independence condition 4 is not enforced by the harness — a
general-purpose subagent inherits Edit and Write — so the read-only restriction has to be stated
in the brief and checked in the verdict. Hand it exactly:

- the goal in one paragraph — which findings were applied, and what each was meant to change
- the scope — paths, or the diff range
- the commands you ran and their exit status
- the verdict envelope schema below, and which lane this checker is
- the revision under review

**On `change_ref`.** Step 7 edits files; it does not commit them, so the commit alone does not
identify what the checker is looking at. Record the commit **and** the tree state:

```bash
git rev-parse --short HEAD 2>/dev/null || echo NOCOMMIT   # baseline, if there is one
git status --porcelain=v1 -uall                           # every path with uncommitted state
git diff HEAD 2>/dev/null                                 # the change in tracked files
```

Hand the checker all three **commands verbatim and their outputs**, and have it re-run them
before it emits a verdict. Any difference — a new path, a changed hunk — means the revision
moved → `UNVERIFIED`. One exception: differences consisting only of untracked artifact paths
created by re-running the handed evidence commands (`__pycache__/`, `.pytest_cache/`, coverage
files) are noted in `evidence_checked`, not treated as drift — and avoided where possible
(`PYTHONDONTWRITEBYTECODE=1`, `pytest -p no:cacheprovider`).

For the compact `change_ref` label in reports and envelopes, both sides derive it the same way:

```bash
echo "$(git rev-parse --short HEAD 2>/dev/null || echo NOCOMMIT)+$( (git status --porcelain=v1 -uall; git diff HEAD 2>/dev/null) | shasum | cut -c1-12 )"
```

The digest is a label for matching reports to reviews; **drift detection compares the raw
outputs**, which also show *what* moved.

Two blind spots to state out loud rather than paper over: `git status` lists untracked paths but
never their **contents**, so a file Step 7 created can be rewritten without moving any of the three
(the maker lists a `shasum` for every untracked path Step 7 touched — created **or modified** —
in the handed material, and a checker that reads such a file re-hashes it against that list); and an untracked nested repo appears
as one `?? sub/` line hiding its whole subtree. Outside a git repo, or when `HEAD` does not resolve,
`shasum` the audited files instead and say so in `evidence_checked`.

Do **not** hand it your conclusion. "I verified this works" is the claim under test, not context.

Checker's brief:

```
You are the checker, not the implementer. You did not write this change, and you must
not start writing it now. Your job is to break the claim that the current code satisfies
the requirement.

- Read the goal, the diff, and the evidence you were handed. Judge the code as it is right
  now, not as the summary describes it — the summary is one of the things under test.
- Check that the evidence you were handed was produced from the revision you are reviewing.
  Evidence from an earlier revision is stale and cannot support APPROVE.
- Reproduce. Construct the input the change claims to handle — including the failure it
  was written to catch — and run it. Reading does not approve runtime behaviour.
- For each CUT: re-grep the removed symbol yourself. Confirm zero remaining references,
  including dynamic ones (getattr, string dispatch, templates, config).
- For each ADD: make the new guard fire. A guard never made to fire is unverified.
- For each FIX: trigger the error path the fix claims to handle.
- Hunt counter-examples in this order: security, data integrity, correctness, performance.
  One reproduced defect beats five speculative ones.
- Bash is for reproduction, never for repair. Never run `git push`, `git reset --hard`,
  `git checkout --`, `rm`, `mv`, or anything else that mutates the tree.
- Never print a secret, credential or token. Cite `file:line`; do not paste the value.
- READ-ONLY. Do not use Edit or Write on anything under the repository, and do not run a
  formatter or fixer. Scratch work goes in `mktemp -d` — copy a file out and break the copy.
  The working tree must be exactly as you found it when you finish; confirm it and say so.
  Findings go back as suggestions; if you edit the source your verdict is void.
- Re-run the three change_ref commands you were handed, at the start and again at the end.
  Any difference: UNVERIFIED — you reviewed a state that no longer exists.
- Confirm the exit statuses you were handed by re-running those commands yourself. Evidence
  that was never produced, or produced from an earlier state, cannot support APPROVE.
- Check that nothing beyond the named findings was changed — extra cleanup that crept into
  the same diff is part of what you are reviewing.
- If handed a shasum list for untracked paths, re-hash any such file you read against it.
- A tool error, timeout, rate limit, or missing runtime is never an APPROVE — name it as
  unrun. If what you could not run **is** the central claim, the verdict is UNVERIFIED, not
  REQUEST_CHANGES: "I could not judge this" is a different statement from "this is broken",
  and reporting it as broken sends the maker to fix a defect nobody demonstrated.
- If you authored or requested this change, or are a fork or continuation of that session,
  do not review it — return UNVERIFIED with that reason.
- Untracked artifact paths your own evidence re-runs created (caches, __pycache__) are
  noted in evidence_checked, not counted as drift — avoid creating them where you can.
- Verdict semantics: APPROVE = you tried to break the claim and failed, zero HIGH findings.
  REQUEST_CHANGES = at least one HIGH finding, reproduced. MEDIUM and LOW findings are
  reported, never blocking. HIGH means a reproduced wrong result, data loss, or security
  hole — not style. UNVERIFIED = you could not judge.
- Return exactly one verdict envelope, in the schema you were handed, naming your lane.
```

## The three verdicts

There are exactly three. "Approve with comments" is a category error — decide which one it is.

| Verdict | Meaning | Obligation |
|---|---|---|
| `APPROVE` | The checker tried to break the claim against the current revision and failed. No HIGH finding. | Done — for **this** revision only. Any further edit voids it. |
| `REQUEST_CHANGES` | At least one HIGH finding, reproduced. | Fix exactly those findings — nothing more — re-run the evidence, dispatch a **fresh** checker. In a two-lane run, re-dispatch both lanes. |
| `UNVERIFIED` | No judgement was reached: the checker never ran, returned empty or malformed output, timed out, was rate-limited, or reviewed a revision that has since moved. | Retry **once**, with a different checker, runtime, or strategy; a second consecutive `UNVERIFIED` → escalate (below). **Never convert it to a pass.** |

With two lanes, the worst verdict wins: `UNVERIFIED` > `REQUEST_CHANGES` > `APPROVE`, and the
combined result carries that verdict's obligation. Any edit after a lane reported voids **every**
lane — all required lanes re-dispatch together against the new `change_ref`, because an APPROVE
belongs to one revision only.

Severity uses this skill's own three levels — HIGH / MEDIUM / LOW (see the Severity table in SKILL.md). MEDIUM and
LOW findings are reported, not blocking; rating a naming nit HIGH trains the loop to ignore you.

## Verdict envelope

```json
{
  "schema": "skeptic.review/v1",
  "verdict": "APPROVE|REQUEST_CHANGES|UNVERIFIED",
  "task_ref": "skeptic-code run on <scope> — applied SC-001, SC-002",
  "change_ref": "<short sha>+<state-hash>",
  "lane": "correctness",
  "working_tree": "unchanged",
  "findings": [
    {
      "severity": "HIGH|MEDIUM|LOW",
      "finding_ref": "SC-002 — or NEW-1 for a defect outside the applied findings",
      "location": "file:line",
      "description": "what is wrong, and how you reproduced it",
      "suggestion": "the smallest change that fixes it"
    }
  ],
  "evidence_checked": ["commands you ran and what they returned", "what you could not run, and why"],
  "summary": "one paragraph: what you tried to break, and why the verdict follows"
}
```

The envelope is the contract, not decoration: a checker response that does not end with a parseable
`skeptic.review/v1` object, or that reports `working_tree` as anything but `unchanged`, is
`UNVERIFIED` — not a verdict you read the tone of.

## Failure modes that void a verdict

- Approving because the diff *looks* right, without ever executing it.
- Trusting the maker's summary about what the code does.
- Inventing a fourth verdict word: "APPROVE with comments" is one of the three — decide which.
- Fixing the defect yourself: the maker must fix it, or the loop learns nothing.

Before accepting an envelope, the maker checks one thing the checker cannot: was the central claim
**reproduced** rather than read, and does every finding carry `file:line`, a reproduction, and a
concrete suggestion? A tidy envelope around an unexecuted review is the failure this step exists to
catch.

## Re-dispatch etiquette

Every round gets a checker with a fresh context — reusing the previous one is asking someone to
re-read their own conclusion. When you come back for round N+1, say plainly:

- what changed since the last round, and which finding each edit addresses
- that it should re-reproduce the original defect rather than take your word that it is gone
- that it is invited to attack **the fix itself** — a fix written under review pressure is exactly
  where the next defect hides

## Escalate after two

When the same finding comes back `REQUEST_CHANGES` twice in a row, **stop**. Two failed fixes on
one defect mean the diagnosis is wrong, and the third attempt usually enlarges the damage rather
than the understanding.

The same cap bounds `UNVERIFIED`: two consecutive `UNVERIFIED` on the same change — each retry
already made with a different checker, runtime, or strategy — means the blocker is environmental,
not procedural. **Stop.** Report what could not be run and why, and let the user decide: accept
the change explicitly unverified, supply the missing runtime, or hold it. Retrying without a cap
is this skill's own item 17, `[CLIFF]` — and it never converts to a pass either way.

Surface to the user: the finding, both attempted fixes, the reproduction command and its output,
and what you now believe is actually wrong. Do not keep looping, do not lower the severity, and do
not restate the goal so that the current code satisfies it.

## Never report the audit complete when

- there is no successful evidence produced **after** the last edit — the test suite where one
  exists, and otherwise the reproduction that shows the change does what it claims
- any condition in the `APPROVE` row above is unmet, or a required lane is missing

In either case the honest report is what is missing — not "done".

---
