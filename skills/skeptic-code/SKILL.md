---
name: skeptic-code
description: Adversarial code audit driven by YAGNI, KISS, and DRY. Every line is guilty until proven innocent. Two hunt passes — Existence (should this code be here?) and Safety (is this code dangerous?) — every candidate verified with evidence, then a third pass of Independent Verification, where a fresh checker that did not write the change tries to break it. Finds ghosts, prophets, liars, twins, strangers, oracles, cliffs, and wheels. HIGH verdicts require reproduction, not pattern matching. Before any new implementation, scans existing project packages for prior art. Clean result is a valid outcome — no forced findings. Produces a prioritized report with concrete before→after diffs.
allowed-tools: [Read, Grep, Glob, Bash, Edit, Write, Task, AskUserQuestion, TodoWrite]
---

# Skeptic-Code — Adversarial Code Audit

> "Innocent until proven guilty? Not here.  
>  **Deleted until proven necessary.**"

Code review asks: "Is this code correct?"  
Skeptic-code asks: "**Should this code exist at all? And if it should — can it survive reality?**"

Every line is a liability. Every abstraction is a bet. Every "just in case" is a cost.

## Three Commandments

| Principle | Question to ask |
|-----------|----------------|
| **YAGNI** — You Aren't Gonna Need It | "Is this required by the current spec, today?" If no → delete. |
| **KISS** — Keep It Simple | "Is there a simpler way that achieves the same result?" If yes → use it. |
| **DRY** — Don't Repeat Yourself | "Does this logic already exist — in this repo, or in a dependency?" If yes → reuse it. |

> **YAGNI beats KISS beats DRY in priority.** Code that doesn't exist is simpler than simple code.  
> DRY's scope extends beyond the file and beyond the repo: **check project packages first.**  
> YAGNI applies to abstractions and optimizations too — never introduce either unless explicitly requested.

## Usage

```
/skeptic-code:skeptic-code              # auto-detect scope
/skeptic-code:skeptic-code quick        # top-5 by severity (HIGH first), then by blast radius (most callsites affected)
/skeptic-code:skeptic-code deep         # full line-level audit
/skeptic-code:skeptic-code <path>       # specific file or directory
```

(Installed standalone rather than as a plugin, the command is `/skeptic-code` without the prefix.)

## The Eight Suspects

| Tag | Name | Crime | Direction |
|-----|------|-------|-----------|
| `[GHOST]` | Dead code | Was needed once. No longer. Still haunting. | CUT |
| `[PROPHET]` | Speculative feature | Written for a future that won't come. | CUT |
| `[LIAR]` | Silent failure | Claims to handle errors. Doesn't. Swallows them. | FIX |
| `[TWIN]` | Duplication | Same logic in two places. One shouldn't exist. | CUT |
| `[STRANGER]` | Scope creep | Nobody asked for this. Wasn't in spec. | CUT |
| `[ORACLE]` | Unverified assumption | Assumes the world cooperates — no validation, no fallback. | ADD |
| `[CLIFF]` | Unbounded failure path | Works until it doesn't. No limit, no retry, no floor. | ADD |
| `[WHEEL]` | Reinvented wheel | Hand-rolled what a project package already provides. | CUT |

## The Wanted List — 18 Known Offenders

### Pass 1A — Existence Hunt (items 1–12)
**Mindset: deletion bias.** "Can I justify this line against the current spec?"  
Leads to: **CUT** or **FIX** (LIAR items 11–12 always lead to FIX — replace the handler, do not simply delete it)

**GHOST · PROPHET · TWIN · STRANGER · WHEEL**

1. **Helper called once** — extracted function with a single callsite → inline it
2. **Interface / abstract class with one implementation or one subclass** — "in case we add more" → delete the interface or collapse the hierarchy into the single concrete class
3. **Config flag that's always the same value** — not a config, it's a constant.
   (URLs, IPs, ports, and file paths belong to item 13, direction ADD — not here.)
4. **`import X` where X is never called** — zombie import
5. **Commented-out code block** — that's what git is for
6. **Null check on something that can't be null** — defensive wall against nothing
7. **UI/endpoint that wasn't in the spec** — Stranger. Cut or get explicit approval.
8. **Hand-rolled utility that a project package already provides** — `[WHEEL]`. Search existing deps first.
9. **Optimization applied without being asked** — premature. Revert to simplest correct form. `[PROPHET]`
10. **Abstraction layer added without being asked** — extra indirection, zero current benefit. Collapse it. `[PROPHET]`

**LIAR**

11. **`except: pass` / `catch (_) {}`** — the lie. Remove the try/catch or actually handle the error.
12. **`or []` / `?? []` on a required field** — patches over a bug upstream. Fix the source, not the symptom.

---

### Pass 1B — Safety Audit (items 13–18)
**Mindset: reinforcement bias.** "Is something missing that should guard this?"  
Leads to: **ADD**

**ORACLE — from Karpathy "Think Before Coding"**

13. **Hardcoded URL, IP, port, or file path** — assumes the environment never changes.
    Remedy shape: an environment variable or config entry with the current value as default.
14. **Return value ignored from a function that can fail** — the caller trusts blindly
15. **No input validation at a system boundary** — API handler, CLI arg, file read accepts anything

**CLIFF — from pre-mortem**

16. **External call (HTTP, DB, queue) with no timeout** — one slow dependency hangs everything
17. **Loop or recursion with no bound** — works until traffic or data size proves otherwise
18. **No retry / fallback on transient failure** — one blip kills the operation permanently

---

## CRITICAL: Verify Before Acting

**Do not flag based on pattern alone.** Every verdict requires evidence. The verification protocol differs by direction.

### Existence candidates (items 1–12) — before marking CUT or FIX (items 11–12: FIX only)

```yaml
verify_existence:
  read_full_file: true      # not just the suspicious line — the whole file
  grep_all_callers: true    # confirm usage count across the entire codebase
  check_scope: true         # is this in tests/, __main__, or dev-only?
  check_comment: true       # is there a comment explaining why it exists?
  reproduce_if_high: true   # rating HIGH → see "Reproduction beats reading" below
  reproduce_liar_fix: true  # items 11–12: make the swallowed error occur, at any severity

rule: ALL must pass.
      Any "unknown" → downgrade to [QUESTION] or [FALSE_ALARM].
```

Required evidence:
```yaml
cut:
  what: "Helper _build_prefix() at fuseki.py:45"
  evidence: "grep: 1 caller (line 89). No independent test. No comment."
```

### Safety candidates (items 13–18) — before marking ADD

```yaml
verify_safety:
  confirm_no_existing_guard: true   # grep call chain for existing timeout/validator — if found → FALSE_ALARM
  check_hot_path: true              # confirm path is reachable: not in disabled branch, not test-only, not behind a feature flag that's always off
  grep_similar_callsites: true      # same pattern elsewhere? fix all or none.
  reproduce_if_runnable: true       # make the missing guard's failure happen — see below

rule: ALL must pass.
      Guard found anywhere in call chain → [FALSE_ALARM].
```

Required evidence:
```yaml
add:
  what: "requests.get(url) at client.py:18"
  evidence: "No timeout= at callsite or session level. grep: 3 similar callsites at lines 42, 67, 91."
```

### Reproduction beats reading

grep evidence proves a pattern exists. It does not prove the pattern is dangerous, and it does not
prove a change works. **A guard is verified by making it fire, not by reading it.**

| Finding | Reproduction |
|---|---|
| Anything you intend to rate HIGH | Required |
| Any `ADD` on a runnable surface | Required |
| Any `FIX` on `[LIAR]` — the swallowed error | Required: make the error occur, watch what the code does with it |
| LOW / MEDIUM `CUT` settled by grep counts | Optional |

```yaml
reproduce:
  where: "outside the repository — mktemp -d, $TMPDIR"
  how: "copy the file out, break the copy, run the guard or test against it"
  never: "mutate the working tree to prove a point"
  after: "the working tree is exactly as you found it — restore anything you touched"
```

Worked shape:

```
Claim:   retry loop is unbounded                          [CLIFF]
Reading: `while not success:` — no counter visible
Running: cp client.py $T/ && stub the transport to always fail && run it
Result:  1.2M iterations in 30s, no exit, no backoff. Reproduced → HIGH.
```

Where the table above says **Required**, a claim that could not be reproduced does not become
`CUT` / `FIX` / `ADD` — it becomes `[QUESTION]` at its original severity — except a HIGH claim, which
drops to MEDIUM, because an unreproduced claim cannot block — and the report names what could not be run and why — missing runtime,
credentials, network. Silence about an unrun check is
how a false pass gets written down as a verdict.

If you cannot fill `evidence` with grep results and line numbers → it is not a verdict.

---

## Workflow

### Step 0: Prerequisites

```bash
# 1. Test suite exists?
[ -d tests ] || [ -d test ] || [ -d spec ] || [ -d __tests__ ] \
  || echo "NO TESTS — post-edit evidence is reproduction, not the suite"

# 2. Skip vendored/generated files
# Exclude: node_modules/, vendor/, dist/, build/, *_generated.*, *.pb.go

# 3. Read CLAUDE.md / README for explicit design decisions
# → These become FALSE_ALARM justifications before hunting starts
```

### Step 0B: Scan Existing Packages (Before ANY New Implementation)

**Only run this step when asked to implement something new — not during a pure audit.**

```bash
# List what's already installed
cat package.json            # Node
cat requirements*.txt pyproject.toml  # Python
cat go.mod                  # Go
cat Cargo.toml              # Rust

# Search for the capability within the project
grep -rn "def <keyword>\|function <keyword>\|const <keyword>" src/
```

Decision rule:
- Project or package already covers the need → **reuse it**, adapt if needed.
- Nothing close exists → only then write new code.

### Step 1: Read Everything

Do not skim. Read the full file(s). Map what exists:

```bash
wc -l <files>                           # size
grep -rn --exclude-dir={.git,node_modules,vendor,dist,build} "def \|class \|function " .   # all defined symbols
grep -rn --exclude-dir={.git,node_modules,vendor,dist,build} "<symbol>" .                    # all callers
```

### Step 2: Pass 1A — Existence Hunt (items 1–12)

Go through items 1–12. For each file, note every matching pattern.  
Record: what, where (file:line), which tag, direction (CUT or FIX).

### Step 3: Pass 1B — Safety Hunt (items 13–18)

**Switch mindset.** You are no longer cutting — you are looking for unguarded danger.  
Go through items 13–18. Note: what path, what guard is missing, what the failure scenario is.

### Step 4: Pass 2 — Verify All Candidates

Apply the matching verification protocol (existence or safety) to each candidate.  
Read ±30 lines around each finding.

- All checks pass → assign verdict (CUT / FIX / ADD)
- Any check fails or is unknown → `[QUESTION]` or `[FALSE_ALARM]`
- Rating HIGH, a `[LIAR]` FIX, or adding a guard to a runnable surface → **reproduce it first**
  (see *Reproduction beats reading*). A pattern match is a candidate, not a verdict.

### Step 5: Build Report

```yaml
skeptic_code:
  scope: "<what was audited>"
  verdict_counts: {cut: N, fix: N, add: N, question: N, false_alarms: N}

  findings:
    - id: SC-001
      tag: "[GHOST]"
      direction: CUT
      verdict: CUT
      location: "file.py:42"
      title: "Helper called once — inline it"
      evidence: "grep: 1 caller at file.py:89. 0 independent tests."
      before: |
        def _fmt(x): return f"prefix:{x}"
        label = _fmt(value)
      after: |
        label = f"prefix:{value}"
      lines_removed: 2
      lines_added: 0

    - id: SC-002
      tag: "[CLIFF]"
      direction: ADD
      verdict: ADD
      location: "client.py:18"
      title: "HTTP call with no timeout — 3 callsites affected"
      evidence: "No timeout= at callsite or session level. grep: lines 18, 42, 67, 91."
      before: |
        resp = requests.get(url)
      after: |
        resp = requests.get(url, timeout=10)
      lines_removed: 0   # in-place edit: count only net line-count change
      lines_added: 0
      note: "Apply to all 4 callsites"

    - id: SC-003
      tag: "[PROPHET]"
      direction: CUT
      verdict: QUESTION
      location: "base.py:10-30"
      title: "Abstract class with one concrete implementation"
      question: "Is a second implementation planned within the current phase?"

  false_alarms:
    - what: "No auth on /api endpoints"
      reason: "CLAUDE.md line 47: 'single-user assumption, no auth in v0.1'"

  verification:            # Pass 3 — filled in after Step 8, omit if no edits were applied
    required: true         # per the trigger table in Step 8
    rounds: 1              # at 2 REQUEST_CHANGES on the same finding → escalate, do not loop
    lanes:                 # one entry per checker; 2 entries for the high-risk row
      - lane: correctness
        checker: "fresh subagent, read-only, did not author the change"
        verdict: APPROVE   # APPROVE | REQUEST_CHANGES | UNVERIFIED
        change_ref: "<short sha>+<state-hash>"   # what THIS lane reviewed
    result: APPROVE        # worst lane wins: UNVERIFIED > REQUEST_CHANGES > APPROVE
```

**If no findings after Pass 2 — clean result:**

```yaml
skeptic_code:
  scope: "src/auth/"
  verdict: CLEAN
  evidence: "18 patterns checked across Pass 1A and 1B. 0 candidates survived Pass 2."
```

> Clean result is valid. **Do not generate forced findings.**

### Step 6: Present and Decide

**Present all findings before making any edit. Never edit without approval.**

Say up front whether Pass 3 will run — the table in Step 8 decides. In short: applying anything
that changes behaviour makes independent verification mandatory before the audit is called done.

Group findings by direction (CUT / FIX / ADD), then ask via AskUserQuestion:
- Apply all verdicts now (Recommended)
- Walk through each finding first
- Apply specific IDs only (this option triggers a follow-up question listing the IDs)
- Report only — no changes

Approving the verdicts also authorizes the fixes a Pass 3 checker demands for those same findings.
Anything the checker raises **outside** the approved findings comes back to the user as a new
finding, not as an edit.

### Step 7: Apply Approved Changes

**CUT:**
1. Show exact before → after
2. Edit the file
3. Run tests if available (`pytest -q` or equivalent)
4. Confirm green, move to next

**FIX / ADD:**
1. Show what's being changed and where
2. Edit the file
3. Run tests if available
4. Grep for similar callsites — fix all, or flag remaining as `[QUESTION]`

### Step 8: Pass 3 — Independent Verification

> The audit is not done when the auditor is satisfied.
> It is done when **someone who did not run it** tried to break it and failed.

An auditor that applied its own verdicts re-runs the assumption that produced them, so a mistaken
verdict survives its own review. Self-review produces an envelope that looks exactly like a real
one — which is why this step is a **dispatch**, not a re-read.

#### When Pass 3 is mandatory

| What Step 7 did | Verification |
|---|---|
| Any `FIX` or `ADD` — behaviour changed | Required — 1 independent checker |
| `CUT` that removed a callsite, export, or public symbol | Required — 1 independent checker |
| `CUT` of a comment, unused import, or dead local | Skip — no behaviour to break |
| Touched auth, authorization, payments, access control, secrets, or a destructive data path | Required — 2 checkers in separate lanes, dispatched independently: **correctness** (did the change do what it claims?) and **security** (what does the change now let through?) |
| Report-only run — no edits applied | Skip — nothing to verify |

"Behaviour" is wider than application code: shell, SQL, CI/CD, hooks, commands, agent and skill
definitions, runtime prompts, and configuration that changes what the system does. Prose, comments
and formatting that cannot change behaviour are exempt. Diff size is not an exemption in either
direction — a one-line change to an authorization check is behavioural, a 900-line doc reflow is
not. When a change is ambiguous, treat it as behavioural.

#### What makes a checker independent

All four must hold:

1. **Fresh context** — it starts from the diff and the goal, not from this audit's reasoning.
2. **Did not author the change** — a past maker is permanently disqualified as that change's checker.
3. **Not a fork of the maker** — a continuation of this session carries the assumptions under test.
4. **Read-only** — if it edits source it has joined the maker set and its verdict is void.

Independence is about what the checker reproduces, not about which model ran it. Nothing enforces
it mechanically. That is exactly why the honest report when no checker can be
reached is `UNVERIFIED`, not a verdict you were in no position to issue.

#### Dispatch

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
(the maker lists a `shasum` for every file Step 7 created in the handed material, and a checker
that reads such a file re-hashes it against that list); and an untracked nested repo appears
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

#### The three verdicts

There are exactly three. "Approve with comments" is a category error — decide which one it is.

| Verdict | Meaning | Obligation |
|---|---|---|
| `APPROVE` | The checker tried to break the claim against the current revision and failed. No HIGH finding. | Done — for **this** revision only. Any further edit voids it. |
| `REQUEST_CHANGES` | At least one HIGH finding, reproduced. | Fix exactly those findings — nothing more — re-run the evidence, dispatch a **fresh** checker. In a two-lane run, re-dispatch both lanes. |
| `UNVERIFIED` | No judgement was reached: the checker never ran, returned empty or malformed output, timed out, was rate-limited, or reviewed a revision that has since moved. | Retry with a different checker, runtime, or strategy. **Never convert it to a pass.** |

With two lanes, the worst verdict wins: `UNVERIFIED` > `REQUEST_CHANGES` > `APPROVE`, and the
combined result carries that verdict's obligation. Any edit after a lane reported voids **every**
lane — all required lanes re-dispatch together against the new `change_ref`, because an APPROVE
belongs to one revision only.

Severity uses this skill's own three levels — HIGH / MEDIUM / LOW (see *Severity* below). MEDIUM and
LOW findings are reported, not blocking; rating a naming nit HIGH trains the loop to ignore you.

#### Verdict envelope

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

#### Failure modes that void a verdict

- Approving because the diff *looks* right, without ever executing it.
- Trusting the maker's summary about what the code does.
- Inventing a fourth verdict word: "APPROVE with comments" is one of the three — decide which.
- Fixing the defect yourself: the maker must fix it, or the loop learns nothing.

Before accepting an envelope, the maker checks one thing the checker cannot: was the central claim
**reproduced** rather than read, and does every finding carry `file:line`, a reproduction, and a
concrete suggestion? A tidy envelope around an unexecuted review is the failure this step exists to
catch.

#### Re-dispatch etiquette

Every round gets a checker with a fresh context — reusing the previous one is asking someone to
re-read their own conclusion. When you come back for round N+1, say plainly:

- what changed since the last round, and which finding each edit addresses
- that it should re-reproduce the original defect rather than take your word that it is gone
- that it is invited to attack **the fix itself** — a fix written under review pressure is exactly
  where the next defect hides

#### Escalate after two

When the same finding comes back `REQUEST_CHANGES` twice in a row, **stop**. Two failed fixes on
one defect mean the diagnosis is wrong, and the third attempt usually enlarges the damage rather
than the understanding.

Surface to the user: the finding, both attempted fixes, the reproduction command and its output,
and what you now believe is actually wrong. Do not keep looping, do not lower the severity, and do
not restate the goal so that the current code satisfies it.

#### Never report the audit complete when

- there is no successful evidence produced **after** the last edit — the test suite where one
  exists, and otherwise the reproduction that shows the change does what it claims
- any condition in the `APPROVE` row above is unmet, or a required lane is missing

In either case the honest report is what is missing — not "done".

---

## Severity

| Severity | Examples |
|----------|---------|
| HIGH | `[LIAR]` on a hot path, `[CLIFF]` call without timeout, 100+ lines of `[STRANGER]` scope creep, anything the Pass 3 security lane reproduces — auth/authz bypass, secret exposure, injection, data reachable that should not be |
| MEDIUM | `[PROPHET]` unsolicited abstraction/optimization, `[WHEEL]` hand-rolled utility ≥20 lines, `[ORACLE]` hardcoded environment value |
| LOW | `[GHOST]` zombie import, stale comment, unnecessary null check, `[WHEEL]` trivial reimplementation <20 lines |

---

## Example

```
User: /skeptic-code src/web/router.py

Step 0: Prerequisites...
  Tests found: tests/test_router.py ✓
  No vendored/generated files in scope ✓
  CLAUDE.md baseline: "single-user, no auth in v0.1" → noted as FALSE_ALARM seed

Reading router.py (239 lines)...
Mapping all defined symbols...

Pass 1A — Existence Hunt (items 1–12)...
  3 candidates found

Pass 1B — Safety Hunt (items 13–18)...
  2 candidates found (one of them — missing auth — seeded from the CLAUDE.md baseline)

Pass 2 — Verifying all 5 candidates...

  SC-001 [LIAR] FIX: save_config() collects errors[] but always returns HTTP 200
    grep: errors list built but status code never set. No test for partial failure.
    Verdict: FIX

  SC-002 [CLIFF] ADD: requests.get(external_api) has no timeout
    grep: no timeout= at callsite or session level. 3 similar callsites: lines 42, 67, 91.
    Verdict: ADD (all 4 callsites)

  SC-003 [CLIFF] ADD: retry loop has no maximum attempt count
    grep: while not success — no counter, no backoff, no circuit breaker.
    Verdict: ADD

  SC-004 [PROPHET] CUT: _MAX_TTL_BYTES constant defined at module level
    grep: used in exactly one Form() call. No config, no override, no test.
    Verdict: QUESTION — inline as literal or justify as named constant?

  FA-001: "No auth on endpoints"
    CLAUDE.md: single-user assumption is explicit design decision.
    Verdict: FALSE_ALARM

skeptic-code complete: 0 CUT, 1 FIX, 2 ADD, 1 QUESTION, 1 false alarm.

[AskUserQuestion → present summary, ask how to proceed]

User: Apply all.

Applying SC-001 (FIX)...
  Before: return TemplateResponse(...) — always 200
  After:  status_code=207 when errors is non-empty
Running tests... 116 passed. ✓

Applying SC-002 (ADD — 4 callsites)...
  Before: resp = requests.get(url)
  After:  resp = requests.get(url, timeout=10)
  Applied to lines 18, 42, 67, 91.
Running tests... 116 passed. ✓

Applying SC-003 (ADD)...
  Before: while not success:
  After:  for attempt in range(MAX_ATTEMPTS):
Running tests... 116 passed. ✓

Pass 3 — Independent Verification (1 FIX + 2 ADD applied → required, 1 lane)
  change_ref: 4b1c9de+a91f30c2e118   (label derived per "On change_ref"; raw outputs handed too)
  Dispatching fresh checker — read-only, no audit context, not handed my conclusion...

  Round 1 → REQUEST_CHANGES
    HIGH · SC-003 · router.py:88 — the loop is now bounded at 5 attempts, but has no backoff.
    Reproduced: copied router.py to $TMP, stubbed the transport to always fail, ran it —
    5 attempts fired in 0.4s against a rate-limited endpoint. The bound was added; the
    cliff moved rather than closed. Working tree unchanged.

  Fixing SC-003 only (exponential backoff, cap 30s)... tests 116 passed. ✓
  Re-dispatching a fresh checker — told what changed, asked to attack the fix itself...

  Round 2 → APPROVE
    change_ref: 4b1c9de+7d02be44a1c9 (three recorded outputs re-run — no drift start to end)
    evidence_checked: made the new guard fire (5 attempts, 0.4s → 24.8s elapsed);
      forced save_config() to partially fail → HTTP 207 (SC-001); grepped requests.get(
      → 4/4 callsites carry timeout= (SC-002); pytest -q → 116 passed.

skeptic-code complete: verified APPROVE on 4b1c9de+7d02be44a1c9.
```

---

## Acknowledgements

The Pass 3 verification loop — maker ≠ checker, the three verdicts, reviewer-death-is-not-approval,
and escalate-after-two — is adapted from [claude-forge](https://github.com/sangrokjung/claude-forge)
(MIT), specifically its `adversarial-reviewer` agent, `review-loop` skill, and
`rules/adversarial-review.md`.

`[PROPHET]`, `[STRANGER]` and the helper-called-once check descend from Andrej Karpathy's
*Simplicity First*; `[ORACLE]` from his *Think Before Coding*. `[CLIFF]` is pre-mortem methodology
(Klein / Kahneman) compressed into a grep-verifiable pattern.
