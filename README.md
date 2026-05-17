# skeptic-code

> "Innocent until proven guilty? Not here. **Deleted until proven necessary.**"

A Claude Code skill for adversarial code auditing.

| | |
|--|--|
| 🇰🇷 | [한국어 README](./README.ko.md) |

---

Code review asks: "Is this code correct?"  
`skeptic-code` asks: **"Should this code exist at all? And if it should — can it survive reality?"**

## Why Existing Tools Are Not Enough

Linters and dead-code analyzers share a hidden assumption: **"if it runs, it belongs."**

`skeptic-code` rejects that assumption. Four failure classes that existing tools miss:

- **YAGNI violations** — code that executes but serves no current requirement
- **Silent failures** — `except: pass` / `catch (_) {}` patterns that eat errors on hot paths
- **Scope creep** — features nobody asked for that felt natural to add
- **Unverified assumptions** — hardcoded values, ignored return results, missing timeouts

Adversarial framing ("guilty until proven innocent") changes the conclusion even when looking at the same code.

## Installation

```bash
/install-skill nuri428/skeptic-code
```

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

`[ORACLE]` and `[CLIFF]` are the two additions beyond the original five. See *Design Rationale* below for why only these two were added and what was deliberately left out.

## How It Works

1. **Read** — full file(s), no skimming
2. **Pass 1: Hunt** — identify candidates against 15 known offender patterns
3. **Pass 2: Verify** — grep confirms actual usage before any `[CUT]` verdict
4. **Report** — prioritized findings with concrete before→after diffs
5. **Apply** — edits only after user approval, test suite runs after each cut

**Key guarantee**: no `[CUT]` without grep evidence. Uncertain findings become `[QUESTION]` or `[FALSE_ALARM]`, never a silent deletion.

## Scope and Limitations

This skill does **not** cover:
- Race conditions — cannot be reliably detected via static grep; false positives here destroy trust
- Runtime failure scenarios requiring profiling or dynamic analysis
- Architectural failure modes
- Coding process guidance (when to ask, how to plan)

For those, use a dedicated architecture review or pre-mortem session.

---

## Design Rationale

The expansion from 5 to 7 suspects was itself validated through the two methodologies this skill draws from.

**Karpathy "Think Before Coding" applied to the expansion:**  
What assumptions was the expansion making?

- `[RACE]` assumed grep can detect race conditions reliably → **rejected**: static analysis produces too many false positives on concurrency issues
- Multi-mode command structure (`exist` / `assume` / `fail` / `full`) — Simplicity First: the request was broader coverage, not a new command interface → **rejected as [PROPHET]**

**Pre-mortem on the expansion:**  
"The expanded skill failed after six months — why?"

- High false positive rate on `[RACE]` wasted dev time and destroyed trust in findings
- Four-mode structure confused users: "which mode should I run?" led to running none
- `[ORACLE]` / `[BLIND]` / `[TIGHTROPE]` overlapped conceptually → collapsed into `[ORACLE]`

What survived: two clearly distinct suspects with grep-verifiable patterns, added to the existing Wanted List without changing the command interface.

---

## Acknowledgements

`skeptic-code` was shaped by two sources:

**[Andrej Karpathy's LLM coding guidelines](https://x.com/karpathy/status/2015883857489522876)**  
The *Simplicity First* principle — no features beyond what was asked, no abstractions for single-use code — is the direct ancestor of `[PROPHET]`, `[STRANGER]`, and the helper-called-once check. *Think Before Coding* — surface assumptions, ask before implementing — is the ancestor of `[ORACLE]`. Thank you for articulating what senior engineers instinctively enforce but rarely write down.

**Pre-mortem methodology** (Gary Klein / Daniel Kahneman)  
Assuming failure has already occurred and working backwards forces a different quality of scrutiny than optimistic review. The adversarial "guilty until proven innocent" stance is pre-mortem applied at the line level. `[CLIFF]` — unbounded growth, missing timeout, no retry floor — is pre-mortem's "how does this fail under real load?" compressed into a grep-verifiable pattern. Thank you for the framework.

Both approaches are far broader than what this skill covers. The limitations above are intentional, not oversights.
