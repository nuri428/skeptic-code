---
name: skeptic-code
description: Adversarial code audit driven by YAGNI, KISS, and DRY. Every line is guilty until proven innocent. Two distinct audit passes — Existence (should this code be here?) and Safety (is this code dangerous?). Finds ghosts, prophets, liars, twins, strangers, oracles, cliffs, and wheels. Before any new implementation, scans existing project packages for prior art. Clean result is a valid outcome — no forced findings. Produces a prioritized report with concrete before→after diffs.
allowed-tools: [Read, Grep, Glob, Bash, Edit, Write, AskUserQuestion, TodoWrite]
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
/skeptic-code              # auto-detect scope
/skeptic-code quick        # top-5 by severity (HIGH first), then by blast radius (most callsites affected)
/skeptic-code deep         # full line-level audit
/skeptic-code <path>       # specific file or directory
```

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
3. **Config flag that's always the same value** — not a config, it's a constant
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

13. **Hardcoded URL, IP, port, or file path** — assumes the environment never changes
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

rule: ALL four must pass.
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

rule: ALL three must pass.
      Guard found anywhere in call chain → [FALSE_ALARM].
```

Required evidence:
```yaml
add:
  what: "requests.get(url) at client.py:18"
  evidence: "No timeout= at callsite or session level. grep: 3 similar callsites at lines 42, 67, 91."
```

If you cannot fill `evidence` with grep results and line numbers → it is not a verdict.

---

## Workflow

### Step 0: Prerequisites

```bash
# 1. Test suite exists?
ls tests/ test/ spec/ __tests__/ 2>/dev/null || echo "NO TESTS — Step 7 will be skipped"

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
grep -rn "def \|class \|function " .    # all defined symbols
grep -rn "<symbol>" .                   # all callers
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
      lines_removed: 0
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

Group findings by direction (CUT / FIX / ADD), then ask via AskUserQuestion:
- Apply all verdicts now (Recommended)
- Walk through each finding first
- Apply specific IDs only
- Report only — no changes

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

---

## Severity

| Severity | Examples |
|----------|---------|
| HIGH | `[LIAR]` on a hot path, `[CLIFF]` call without timeout, 100+ lines of `[STRANGER]` scope creep |
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
  4 candidates found

Pass 1B — Safety Hunt (items 13–18)...
  2 candidates found

Pass 2 — Verifying all 6 candidates...

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
```
