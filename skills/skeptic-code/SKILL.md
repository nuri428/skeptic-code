---
name: skeptic-code
description: Adversarial engineering skill that verifies claims and changes with evidence, in five modes — AUTO (lightweight guard while implementing any coding request), QUICK (top-5 focused review), DEEP (full YAGNI/KISS/DRY line-level audit of the 18 offenders), BUG (symptom-first behavioural defect investigation with reproduction, invariant tracing and regression tests), ARCHITECTURE (boundary, dependency-direction and coupling review). Nothing gets a verdict without evidence — a pattern match is only a candidate, runtime HIGH claims require reproduction, clean result is valid, and material changes get independent verification by a fresh checker (Pass 3). Use when implementing or refactoring, reviewing code, running a full audit, investigating an error or regression, or evaluating structure and layering.
allowed-tools: [Read, Grep, Glob, Bash, Edit, Write, Task, AskUserQuestion, TodoWrite]
---

# Skeptic-Code — Adversarial Engineering Audit

> **Nothing gets a verdict without evidence.**
>
> Code must justify its existence. Abstractions must justify their boundaries.
> Guards must justify their failure model. Bug fixes must restore a demonstrated invariant.
> Architecture must justify its dependency direction.

Code review asks: "Is this code correct?"
Skeptic-code asks: "**Can this claim survive evidence — and this code, reality?**"

One epistemology, five modes. The deletion creed — *"innocent until proven guilty? Not here.
Deleted until proven necessary"* — is DEEP's law, not a universal one: a bug hunt puts
correctness before parsimony, and an architecture review weighs boundaries before line count.

## One Epistemology, Five Modes

Every mode obeys the same invariants:

- **Evidence before verdict.** A pattern match is only a candidate.
- **Reproduction beats speculation** — runtime HIGH claims require it where a runtime exists
  (`[UNREPRODUCED-HIGH]` where it does not).
- **Clean result is valid.** No forced findings, ever.
- **Fix the source, not the symptom.**
- **Maker ≠ checker.** Material changes get independent verification (Pass 3).

What changes per mode is the **priority stack** — which concern wins when they conflict:

```text
AUTO          Scope > Correctness > Simplicity
QUICK         Severity > Blast Radius > Simplicity
DEEP          YAGNI > KISS > DRY
BUG           Correctness > Reproduction > Root Cause > Minimal Fix > Simplicity
ARCHITECTURE  Boundary > Dependency Direction > Coupling > Simplicity > Abstraction Cost
```

| Mode | Is | Protocol lives in |
|---|---|---|
| `AUTO` | Prevention layer while implementing a coding request — cheap, diff-scoped | this file, below |
| `QUICK` | Focused review — the DEEP pipeline, top-5 by severity then blast radius | this file |
| `DEEP` | Full line-level audit — 18 offenders, both hunt passes | this file |
| `BUG` | Symptom-first behavioural defect investigation | `references/bug-mode.md` — **read on entry** |
| `ARCHITECTURE` | Boundary / dependency-direction / coupling review | `references/architecture-mode.md` — **read on entry** |

### Mode router

With no explicit mode, route by the shape of the request:

```text
"이 기능 구현해줘" — any implementation or refactor request      → AUTO (while you build)
"이 코드 이상한 데 찾아줘" — review this change/file             → QUICK
"전체적으로 싹 감사해줘" — full audit                            → DEEP
"왜 여기서 NoneType 에러가 나지?" — error, failure, regression   → BUG
"이 구조 괜찮아?" — layering, boundaries, dependencies           → ARCHITECTURE
```

An explicit mode or path argument always wins. BUG and ARCHITECTURE are entered only through
their reference file — never run them from memory of this table.

## Three Commandments

| Principle | Question to ask |
|-----------|----------------|
| **YAGNI** — You Aren't Gonna Need It | "Is this required by the current spec, today?" If no → delete. |
| **KISS** — Keep It Simple | "Is there a simpler way that achieves the same result?" If yes → use it. |
| **DRY** — Don't Repeat Yourself | "Does this logic already exist — in this repo, or in a dependency?" If yes → reuse it. |

> **YAGNI beats KISS beats DRY in priority.** Code that doesn't exist is simpler than simple code.  
> DRY's scope extends beyond the file and beyond the repo: **check project packages first.**  
> YAGNI applies to abstractions and optimizations too — never introduce either unless explicitly requested.

This stack governs DEEP and QUICK (and AUTO's prior-art rule is DRY's). BUG and ARCHITECTURE
rank differently — see the priority stacks above.

## Usage

```
/skeptic-code:skeptic-code                  # route by request shape (see Mode router)
/skeptic-code:skeptic-code auto             # prevention guard while implementing
/skeptic-code:skeptic-code quick            # top-5 by severity (HIGH first), then blast radius
/skeptic-code:skeptic-code deep             # full line-level audit
/skeptic-code:skeptic-code bug              # symptom-first defect investigation
/skeptic-code:skeptic-code architecture     # boundary and dependency review
/skeptic-code:skeptic-code <path>           # audit a specific file or directory
```

(Installed standalone rather than as a plugin, the command is `/skeptic-code` without the prefix.)

## AUTO Mode — the Prevention Layer

AUTO is not an audit. It runs **while implementing a coding request** and stops new debt at the
door: unrequested features, speculative abstraction, reimplementing what exists, new duplication,
silent failure, unguarded new external boundaries. It must not slow implementation down — no
18-item scan, no repo-wide line audit, no reproduction runs per request.

Scope = `git diff` + directly affected symbols + direct callers/callees when needed. Whole-repo
search only to answer four questions: does this already exist (code or dependency)? does it
duplicate something? does it cross an architecture boundary? which callsites does this symbol
change?

**A0 — before coding.** Identify the requested scope. Search existing implementations and
dependencies (`package.json`, `requirements*.txt`, `pyproject.toml`, `go.mod`, `Cargo.toml`;
existing helpers, services, adapters, providers). Read `CLAUDE.md` / `README` /
`ARCHITECTURE.md` / `ADR/` for constraints. Existing project code covers the need → reuse or
adapt. A dependency covers it → prefer the dependency unless project constraints say otherwise.
Nothing fits → the simplest correct implementation.

**A1 — while coding.** Watch only these six, applied to what you are writing:

| | Ask |
|---|---|
| `[STRANGER]` | Did I add anything outside the requested scope? |
| `[PROPHET]` | Did I add an abstraction, optimization, or config the request doesn't need today? |
| `[WHEEL]` | Did I reimplement what the repo or a dependency already provides? |
| `[TWIN]` | Did this change duplicate existing logic? |
| `[LIAR]` | Does my error handling actually handle — or swallow? |
| `[CLIFF]` | Does a **new** HTTP/DB/queue/file/external boundary lack an obvious failure bound? |

**A2 — after coding.** Inspect the diff and changed symbols only; run targeted tests; confirm
the result matches the requested scope — nothing more. **"While I'm here" cleanup of unrelated
existing code is forbidden unless explicitly requested.**

**Verdicts in AUTO** apply to the code being written, not to the pre-existing codebase:

```yaml
auto_mode:
  low:    {action: silently avoid or fix in your own new code, report: optional}
  medium: {action: avoid or fix, report: mention in the completion summary}
  high:   {action: stop or surface before proceeding, report: required}
```

No per-finding approval workflow — except that these always surface to the user before
proceeding: a conflict with the stated requirement; a public API, schema, or data-migration
change; a destructive operation; an auth/security semantics change; an architecture boundary
change; an unavoidable change to existing behaviour. A defect noticed in *unrelated existing*
code is reported as a candidate for a later QUICK/DEEP run — never fixed silently.

---

## The Core Suspects (DEEP · QUICK · AUTO)

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
| `[BOUNDARY]` | Deliberate seam | Not a crime — a boundary whose value is separation, substitution, or contract stability, not implementation count. | KEEP |

BUG and ARCHITECTURE carry their own suspect sets — `[MISMATCH]` `[EDGE]` `[STATE]` `[ORDER]`
`[LEAK]` `[RACE]` `[TYPE]` `[DRIFT]`, and `[BLEED]` `[INVERSION]` `[COUPLING]` `[CONTRACT]`
`[CYCLE]` `[ISLAND]` `[BYPASS]` — defined in their reference files.

## The Wanted List — 18 Known Offenders

### Pass 1A — Existence Hunt (items 1–12)
**Mindset: deletion bias.** "Can I justify this line against the current spec?"  
Leads to: **CUT** or **FIX** (LIAR items 11–12 always lead to FIX — replace the handler, do not simply delete it)

**GHOST · PROPHET · TWIN · STRANGER · WHEEL**

1. **Helper called once** — candidate, not a verdict. KEEP when it earns its place: a meaningful
   semantic name, a tested contract, isolation of a side effect, boundary adaptation, or real
   complexity reduction at the callsite. Otherwise → inline it
2. **Interface / abstract class with one implementation or one subclass** — run the
   **Architecture Intent Check** (below) first. `[BOUNDARY]` role evidenced → KEEP. No boundary
   role and no current need → collapse into the single concrete class. `[PROPHET]`
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
18. **No retry / fallback on transient failure** — inspect failure semantics **before**
    proposing retry. Transient + retry-safe (idempotent, duplicate-side-effect-safe, no existing
    retry layer) → ADD candidate. Non-idempotent or semantics unknown — payment, order creation,
    publish without idempotency key, destructive mutation — → `[QUESTION]` or a different guard:
    timeout, fail-fast, idempotency key, dedup, transaction, queue retry policy, explicit
    propagation. A retry that duplicates a side effect is worse than the blip it papers over.

---

## Architecture Intent Check

Items 1, 2 and 10 misfire on deliberate architecture when applied mechanically. Before tagging
an interface, abstract class, adapter, repository, provider, gateway, port, or a
plugin/domain/transport/persistence boundary as `[PROPHET]` or `[GHOST]`, check for a role that
implementation count cannot measure:

```yaml
verify_architecture:
  cross_boundary_role: true        # separates domain from infrastructure?
  dependency_inversion_role: true  # does the import direction depend on it?
  test_substitution_role: true     # do tests substitute or mock through it?
  external_isolation_role: true    # does it isolate an external system?
  stable_contract_role: true       # do consumers rely on it staying put?
  documented_intent: true          # CLAUDE.md / ARCHITECTURE.md / ADR / DI or plugin registrations
```

Any role evidenced → verdict `KEEP`, tag `[BOUNDARY]` — not a CUT target.
**Single implementation ≠ unnecessary abstraction.** `[PROPHET]` requires ALL of: single
implementation, no boundary role, no substitution need, no external isolation, no
stable-contract requirement, no documented intent, no current-phase requirement. And absent
documentation is not absent intent — the code structure itself (import directions, DI
registrations, test fakes) is evidence.

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
  architecture_intent: true # items 1, 2, 10: run the Architecture Intent Check first

rule: ALL must pass.
      Any "unknown" → [QUESTION] or [FALSE_ALARM] — except a HIGH claim whose required
      reproduction could not run, which becomes [UNREPRODUCED-HIGH], severity kept
      (see "Reproduction beats reading").
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
  retry_safety: true                # item 18: idempotency and side-effect safety before proposing retry

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
| A **runtime-defect claim** you intend to rate HIGH | Required |
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

A HIGH claim with **no runtime failure to fire** — 100+ lines of `[STRANGER]` scope creep, a
`[TWIN]` pair — is evidenced by grep against the spec and the codebase instead; reproduction does
not apply to it, and nothing below follows from its absence. An environment-assumption `[ORACLE]`
(hardcoded URL, IP, path) sits at MEDIUM per the Severity table unless its failure is actually
fired — pattern alone never makes it HIGH.

Where the table above says **Required**, a claim that could not be reproduced does not become
`CUT` / `FIX` / `ADD` — and it is **not downgraded** either. "Could not judge" is a different
statement from "less severe"; rounding the first into the second is the same mistake Pass 3
forbids when it refuses to convert `UNVERIFIED` into a pass. A HIGH claim becomes
`[UNREPRODUCED-HIGH]`: severity kept, verdict suspended, and the report names what could not be
run and why — missing runtime, credentials, network (routine in an air-gapped audit). Whether it
blocks is **the user's call at Step 6**, not the auditor's. Anything below HIGH becomes
`[QUESTION]` at its original severity. Silence about an unrun check is how a false pass gets
written down as a verdict.

If you cannot fill `evidence` with grep results and line numbers → it is not a verdict.

---

## DEEP Workflow — the default audit

(QUICK runs this same pipeline, reporting only the top-5 by severity, then blast radius.
AUTO, BUG, and ARCHITECTURE have their own workflows — AUTO above, the others in `references/`.)

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
- Any check fails or is unknown → `[QUESTION]` or `[FALSE_ALARM]` — except a HIGH claim whose
  required reproduction could not run, which becomes `[UNREPRODUCED-HIGH]`, severity kept
- Rating HIGH on a **runtime-defect claim**, a `[LIAR]` FIX, or adding a guard to a runnable
  surface → **reproduce it first** (see *Reproduction beats reading*; non-runtime HIGHs are
  grep-evidenced instead). A pattern match is a candidate, not a verdict.

### Step 5: Build Report

```yaml
skeptic_code:
  mode: DEEP               # AUTO | QUICK | DEEP | BUG | ARCHITECTURE
  scope: "<what was audited>"
  verdict_counts: {cut: N, fix: N, add: N, keep: N, question: N, unreproduced_high: N, false_alarms: N}
  # BUG mode adds bug/suspect/unverified counts and per-finding invariant/root_cause/regression_test
  # fields; ARCHITECTURE mode uses its own finding shape — see the reference files.

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

    - id: SC-004
      tag: "[CLIFF]"
      direction: ADD
      verdict: UNREPRODUCED-HIGH
      location: "worker.py:71"
      title: "Unbounded queue consumer — reproduction requires the production broker"
      evidence: "grep: no max batch, no backpressure. Could not run: broker unreachable from audit env."
      note: "Severity kept at HIGH; blocking is the user's call at Step 6."

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

Group findings by direction (CUT / FIX / ADD). `KEEP` and `FALSE_ALARM` entries are
report-only — severity-less by design, no approval question, excluded from QUICK's top-5
ranking. `[UNREPRODUCED-HIGH]` findings get their own
group, presented first, with their own question per finding — **block** (treat as HIGH until it
can be run), **apply anyway** (accept the proposed change unverified), or **defer** (leave in the
report). "Apply all verdicts now" never covers them — a suspended verdict is not a verdict. Then
ask via AskUserQuestion:
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

The full protocol — independence conditions, dispatch and the checker's brief, `change_ref`,
the three verdicts, the envelope, re-dispatch etiquette, and both escalation caps — lives in
**`references/pass3-verification.md`. Read that file before dispatching a checker**; do not run
Pass 3 from memory of it. On a report-only run nothing is dispatched, so nothing is read — that
is why the protocol is not inlined here.

In one line: a fresh checker that did not write the change tries to break it — reproducing, not
reading — and returns exactly one of `APPROVE` / `REQUEST_CHANGES` / `UNVERIFIED`. No completion
claim without an `APPROVE` for the code as it stands now.

## Severity

| Severity | Examples |
|----------|---------|
| HIGH | `[LIAR]` on a hot path, `[CLIFF]` call without timeout, 100+ lines of `[STRANGER]` scope creep, anything the Pass 3 security lane reproduces — auth/authz bypass, secret exposure, injection, data reachable that should not be |
| MEDIUM | `[PROPHET]` unsolicited abstraction/optimization, `[WHEEL]` hand-rolled utility ≥20 lines, `[ORACLE]` hardcoded environment value |
| LOW | `[GHOST]` zombie import, stale comment, unnecessary null check, `[WHEEL]` trivial reimplementation <20 lines |

---

## Example (DEEP)

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
  change_ref: 4b1c9de+a91f30c2e118   (label per the reference's "On change_ref"; raw outputs handed too)
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

The mode architecture (AUTO/QUICK/DEEP/BUG/ARCHITECTURE), the Architecture Intent Check and
`[BOUNDARY]`, the BUG-mode suspects and invariant-first debugging flow, and the retry-safety
gate on item 18 were contributed as a community improvement proposal (2026-08).
