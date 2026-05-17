---
name: skeptic-code
description: Code skeptic audit. Every line of code is guilty until proven innocent — if it can't justify its existence, it gets cut. Finds ghosts (dead code), prophets (speculative features), liars (silent failures), twins (duplication), strangers (unrequested additions), oracles (unverified assumptions), and cliffs (unbounded failure paths). Two-pass verify-before-cut prevents false positives. Produces a prioritized cut list with concrete before→after diffs.
allowed-tools: [Read, Grep, Glob, Bash, Edit, Write, AskUserQuestion, TodoWrite]
---

# Skeptic-Code — Adversarial Code Audit

> "Innocent until proven guilty? Not here.  
>  **Deleted until proven necessary.**"

Code review asks: "Is this code correct?"  
Skeptic-code asks: "**Should this code exist at all? And if it should — can it survive reality?**"

Every line is a liability. Every abstraction is a bet. Every "just in case" is a cost.

## Why Existing Tools Are Not Enough

Linters and dead-code analyzers share a hidden assumption:  
**"If it runs, it belongs."**

Skeptic-code rejects that assumption. Four failure classes that existing tools miss:

- **YAGNI violations** — Code that executes but serves no current requirement. Static analysis passes it because it compiles and runs.
- **Silent failures** — `except: pass` / `catch (_) {}` patterns that eat errors. Linters warn on style but cannot judge whether this is on a hot path.
- **Scope creep** — Features nobody asked for that felt natural to add. Tools pass them because the code is used.
- **Unverified assumptions** — Hardcoded values, ignored return results, missing timeouts. The code assumes the world cooperates. It won't.

Adversarial framing ("guilty until proven innocent") changes the conclusion even when looking at the same code.

## Design Rationale

This skill was validated through two lenses before expansion:

**Karpathy's "Think Before Coding"** — applied retrospectively to existing code: what assumptions did the author make without documenting or testing them? This produced `[ORACLE]`.

**Pre-mortem methodology** — "this code caused a production incident; why?" applied at line level: what grows without bound, what has no fallback, what silently swallows failure? This produced `[CLIFF]`.

Both lenses were also applied to the expansion itself. Two suspects were rejected:
- `[RACE]` — race conditions cannot be reliably detected by static grep; false positives would destroy trust in the tool.
- Multi-mode structure (`exist` / `assume` / `fail`) — Karpathy's Simplicity First: the user asked for more coverage, not a new command interface.

## Usage

```
/skeptic-code              # auto-detect scope
/skeptic-code quick        # top-5 findings, ~2 min
/skeptic-code deep         # full line-level audit, ~10 min
/skeptic-code <path>       # specific file or directory
```

## The Seven Suspects

| Tag | Name | Crime |
|-----|------|-------|
| `[GHOST]` | Dead code | Was needed once. No longer. Still haunting. |
| `[PROPHET]` | Speculative feature | Written for a future that won't come. "We'll probably need..." |
| `[LIAR]` | Silent failure | Claims to handle errors. Doesn't. Swallows them. |
| `[TWIN]` | Duplication | Same logic in two places. One of them shouldn't exist. |
| `[STRANGER]` | Scope creep | Nobody asked for this. Felt natural to add. Wasn't in spec. |
| `[ORACLE]` | Unverified assumption | Assumes the world cooperates — no validation, no fallback, no test. |
| `[CLIFF]` | Unbounded failure path | Works until it doesn't. No limit, no retry, no floor. |

## The Wanted List — 15 Known Offenders

Hunt these immediately. Each match earns a finding.

**Existence (GHOST · PROPHET · TWIN · STRANGER)**

1. **Helper called once** — extracted function with a single callsite → inline it
2. **Interface with one implementation** — "in case we add more" → delete the interface
3. **`except: pass` / `catch (_) {}`** — the lie. Remove it or actually handle it. `[LIAR]`
4. **`or []` / `?? []` on a required field** — patches over a bug upstream `[LIAR]`
5. **Config flag that's always the same value** — not a config, it's a constant
6. **`import X` where X is never called** — zombie import
7. **Commented-out code block** — that's what git is for
8. **Null check on something that can't be null** — defensive wall against nothing
9. **"We might want this later" abstraction** — YAGNI. Cut it.
10. **UI/endpoint that wasn't in the spec** — Stranger. Cut or get explicit approval.

**Assumptions (ORACLE) — from Karpathy "Think Before Coding"**

11. **Hardcoded URL, IP, port, or file path** — assumes the environment never changes
12. **Return value ignored from a function that can fail** — the caller trusts blindly
13. **No input validation at a system boundary** — API handler, CLI arg, file read accepts anything

**Failure paths (CLIFF) — from pre-mortem**

14. **External call (HTTP, DB, queue) with no timeout** — one slow dependency hangs everything
15. **Loop or recursion with no bound, or no retry/fallback on transient failure** — works until traffic, data size, or a blip proves otherwise

## CRITICAL: Verify Before Cutting

**Do not flag based on pattern alone.** Every `[CUT]` verdict requires evidence.

Before marking anything `[CUT]`, confirm all four:

```yaml
verification:
  read_full_file: true      # not just the suspicious line — the whole file
  grep_all_callers: true    # confirm usage count across the entire codebase
  check_scope: true         # is this in tests/, __main__, or dev-only?
  check_comment: true       # is there a comment explaining why it exists?

rule: ALL four must pass to cut.
      Any "unknown" → downgrade to [QUESTION] or [FALSE_ALARM].
```

**Required evidence for every `[CUT]`:**
```yaml
cut:
  what: "Helper _build_prefix() at fuseki.py:45"
  evidence: "grep: 1 caller (line 89). No independent test. No comment."
```

If you cannot fill `evidence` with grep results and line numbers → it is not a `[CUT]`.

## Workflow

### Step 1: Read Everything

Do not skim. Read the full file(s). Map what exists:

```bash
wc -l <files>                           # size
grep -rn "def \|class \|function " .    # all defined symbols
grep -rn "<symbol>" .                   # all callers
```

### Step 2: Pass 1 — Hunt

Go through all 15 items on the Wanted List. For each file, note every matching pattern.  
Record: what, where (file:line), which tag.

### Step 3: Pass 2 — Verify

For each candidate, run the verification checklist.  
Read ±30 lines around the finding.  
All four checks pass → `[CUT]`.  
Any check fails → `[QUESTION]` or `[FALSE_ALARM]`.

### Step 4: Build Report

```yaml
skeptic_code:
  scope: "<what was audited>"
  verdict_counts: {cut: N, question: N, false_alarms: N}

  findings:
    - id: SC-001
      tag: "[GHOST]"
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

    - id: SC-002
      tag: "[ORACLE]"
      verdict: CUT
      location: "client.py:18"
      title: "HTTP call with no timeout"
      evidence: "requests.get(url) — no timeout= parameter. grep: no timeout set at call site or session level."
      before: |
        resp = requests.get(url)
      after: |
        resp = requests.get(url, timeout=10)
      lines_removed: 0

    - id: SC-003
      tag: "[PROPHET]"
      verdict: QUESTION
      location: "base.py:10-30"
      title: "Abstract Protocol with one concrete implementation"
      question: "Is a second backend planned within the current phase?"

  false_alarms:
    - what: "No auth on /api endpoints"
      reason: "CLAUDE.md line 47: 'single-user assumption, no auth in v0.1'"
```

### Step 5: Present and Decide

**Present all findings before making any edit. Never edit without approval.**

Summarize findings and the biggest wins, then ask the user to choose via AskUserQuestion:
- Apply all CUT items now (Recommended)
- Walk through each finding first
- Apply specific IDs only
- Report only — no changes

### Step 6: Apply Approved Cuts

For each approved cut:
1. Show exact before → after
2. Edit the file
3. Run tests (`pytest -q` or equivalent)
4. Confirm green, move to next

## Severity

| Severity | Examples |
|----------|---------|
| HIGH | Silent failure on a hot path, HTTP call without timeout, 100+ lines of scope creep |
| MEDIUM | Unused abstraction, helper called once, hardcoded environment value |
| LOW | Zombie import, stale comment, unnecessary null check |

## Example

```
User: /skeptic-code src/web/router.py

Reading router.py (239 lines)...
Scanning 15 known offenders...

Pass 1 — 5 candidates found
Pass 2 — Verifying each:

  SC-001 [LIAR]: save_config() collects errors[] but always returns HTTP 200
    grep: errors list built but status never set. No test for partial failure.
    Verdict: CUT

  SC-002 [ORACLE]: requests.get(external_api) has no timeout parameter
    grep: no timeout= anywhere in call chain. One slow upstream hangs the worker.
    Verdict: CUT

  SC-003 [CLIFF]: retry loop has no maximum attempt count
    grep: while not success — no counter, no sleep backoff, no circuit breaker.
    Verdict: CUT

  SC-004 [PROPHET]: _MAX_TTL_BYTES constant at module level
    grep: used in exactly one Form() call. No config, no override, no test.
    Verdict: QUESTION (or inline as literal)

  FA-001: "No auth on endpoints"
    Checked CLAUDE.md — single-user assumption is explicit design decision.
    Verdict: FALSE_ALARM

skeptic-code complete: 3 CUT, 1 QUESTION, 1 false alarm.

[AskUserQuestion: present summary, ask how to proceed]

User: Apply all CUT items.

Applying SC-001...
  Before: return templates.TemplateResponse(...) always 200
  After:  status_code=207 when errors is non-empty
Running tests... 116 passed. ✓

Applying SC-002...
  Before: resp = requests.get(url)
  After:  resp = requests.get(url, timeout=10)
Running tests... 116 passed. ✓
```
