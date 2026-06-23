# skeptic-code

> "Innocent until proven guilty? Not here. **Deleted until proven necessary.**"

A Claude Code skill for adversarial code auditing.

| | |
|--|--|
| 🇰🇷 | [한국어 README](./README.ko.md) |

---

Code review asks: "Is this code correct?"  
`skeptic-code` asks: **"Should this code exist at all? And if it should — can it survive reality?"**

## Three Commandments

| Principle | Question to ask |
|-----------|----------------|
| **YAGNI** — You Aren't Gonna Need It | "Is this required by the current spec, today?" If no → delete. |
| **KISS** — Keep It Simple | "Is there a simpler way that achieves the same result?" If yes → use it. |
| **DRY** — Don't Repeat Yourself | "Does this logic already exist — in this repo, or in a dependency?" If yes → reuse it. |

> YAGNI beats KISS beats DRY in priority. Code that doesn't exist is simpler than simple code.  
> YAGNI applies to abstractions and optimizations too — never introduce either unless explicitly requested.

## Installation

**Via marketplace (recommended):**

```
/plugin marketplace add nuri428/skeptic-code
/plugin install skeptic-code@skepticcode
```

## Usage

```
/skeptic-code:skeptic-code              # auto-detect scope
/skeptic-code:skeptic-code quick        # top-5 by severity (HIGH first), then blast radius
/skeptic-code:skeptic-code deep         # full line-level audit
/skeptic-code:skeptic-code <path>       # specific file or directory
```

## The Eight Suspects

| Tag | Name | Crime | Direction |
|-----|------|-------|-----------|
| `[GHOST]` | Dead code | Was needed once. No longer. Still haunting. | CUT |
| `[PROPHET]` | Speculative feature | Written for a future that won't come. "We'll probably need..." | CUT |
| `[LIAR]` | Silent failure | Claims to handle errors. Doesn't. Swallows them. | FIX |
| `[TWIN]` | Duplication | Same logic in two places. One of them shouldn't exist. | CUT |
| `[STRANGER]` | Scope creep | Nobody asked for this. Felt natural to add. Wasn't in spec. | CUT |
| `[ORACLE]` | Unverified assumption | Assumes the world cooperates — no validation, no fallback, no test. | ADD |
| `[CLIFF]` | Unbounded failure path | Works until it doesn't. No limit, no retry, no floor. | ADD |
| `[WHEEL]` | Reinvented wheel | Hand-rolled what a project package already provides. | CUT |

## How It Works

Two distinct passes with opposite mindsets:

**Pass 1A — Existence Hunt (items 1–12)**  
*Deletion bias.* "Can I justify this line against the current spec?" → CUT or FIX

**Pass 1B — Safety Hunt (items 13–18)**  
*Reinforcement bias.* "Is something missing that should guard this?" → ADD

After both passes, every candidate is verified with grep evidence before a verdict is assigned. No `[CUT]` or `[ADD]` without grep results and line numbers.

| Step | Action |
|------|--------|
| 0 | Prerequisites check (test suite, vendored files, CLAUDE.md baseline) |
| 1 | Read all files — no skimming |
| 2 | Pass 1A: hunt 12 existence patterns |
| 3 | Pass 1B: hunt 6 safety patterns |
| 4 | Pass 2: verify all candidates with grep |
| 5 | Build report |
| 6 | Present findings — user approves before any edit |
| 7 | Apply approved changes, run tests after each |

**Clean result is valid.** If 18 patterns return nothing, the output is `CLEAN` — no forced findings.

## Scope and Limitations

This skill does **not** cover:
- Race conditions — cannot be reliably detected via static grep; false positives here destroy trust
- Runtime failure scenarios requiring profiling or dynamic analysis
- Architectural failure modes

---

## Acknowledgements

`skeptic-code` was shaped by two sources:

**[Andrej Karpathy's LLM coding guidelines](https://x.com/karpathy/status/2015883857489522876)**  
*Simplicity First* — no features beyond what was asked, no abstractions for single-use code — is the ancestor of `[PROPHET]`, `[STRANGER]`, and the helper-called-once check. *Think Before Coding* — surface assumptions before implementing — is the ancestor of `[ORACLE]`.

**Pre-mortem methodology** (Gary Klein / Daniel Kahneman)  
Assuming failure has already occurred and working backwards forces a different quality of scrutiny than optimistic review. The adversarial "guilty until proven innocent" stance is pre-mortem applied at the line level. `[CLIFF]` is pre-mortem's "how does this fail under real load?" compressed into a grep-verifiable pattern.

Both approaches are far broader than what this skill covers. The limitations above are intentional, not oversights.
