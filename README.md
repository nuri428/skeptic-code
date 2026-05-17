# skeptic-code

> "Innocent until proven guilty? Not here. **Deleted until proven necessary.**"

A Claude Code skill for adversarial code auditing.

Code review asks: "Is this code correct?"  
`skeptic-code` asks: **"Should this code exist at all?"**

## Why Existing Tools Are Not Enough

Linters and dead-code analyzers share a hidden assumption: **"if it runs, it belongs."**

`skeptic-code` rejects that assumption. Three failure classes that existing tools miss:

- **YAGNI violations** — code that executes but serves no current requirement
- **Silent failures** — `except: pass` / `catch (_) {}` patterns that eat errors on hot paths
- **Scope creep** — features nobody asked for that felt natural to add

Adversarial framing ("guilty until proven innocent") changes the conclusion even when looking at the same code.

## Scope and Limitations

This skill applies **Karpathy's Simplicity First principle** and a **pre-mortem lens**, narrowed to one question: *does this code need to exist?*

It does **not** cover:
- Runtime failure scenarios (load, concurrency, dependency outages)
- Wrong assumptions in business logic
- Architectural failure modes
- Coding process guidance (when to ask, how to plan)

For those, use a dedicated architecture review or pre-mortem session.

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

## The Five Suspects

| Tag | Name | Crime |
|-----|------|-------|
| `[GHOST]` | Dead code | Was needed once. No longer. Still haunting. |
| `[PROPHET]` | Speculative feature | Written for a future that won't come. "We'll probably need..." |
| `[LIAR]` | Silent failure | Claims to handle errors. Doesn't. Swallows them. |
| `[TWIN]` | Duplication | Same logic in two places. One of them shouldn't exist. |
| `[STRANGER]` | Scope creep | Nobody asked for this. Felt natural to add. Wasn't in spec. |

## How It Works

1. **Read** — full file(s), no skimming
2. **Pass 1: Hunt** — identify candidates against 10 known offender patterns
3. **Pass 2: Verify** — grep confirms actual usage before any `[CUT]` verdict
4. **Report** — prioritized findings with concrete before→after diffs
5. **Apply** — edits only after user approval, test suite runs after each cut

**Key guarantee**: no `[CUT]` without grep evidence. Uncertain findings become `[QUESTION]` or `[FALSE_ALARM]`, never a silent deletion.

---

## Acknowledgements

`skeptic-code` was shaped by two sources:

**[Andrej Karpathy's LLM coding guidelines](https://x.com/karpathy/status/2015883857489522876)**  
The *Simplicity First* principle — no features beyond what was asked, no abstractions for single-use code, no "flexibility" that wasn't requested — is the direct ancestor of the `[PROPHET]`, `[STRANGER]`, and helper-called-once checks. Thank you for articulating what senior engineers instinctively enforce but rarely write down.

**Pre-mortem methodology** (Gary Klein / Daniel Kahneman)  
Assuming failure has already occurred and working backwards forces a different quality of scrutiny than optimistic review. The adversarial "guilty until proven innocent" stance in this skill is pre-mortem applied at the line level. Thank you for the framework.

Both approaches are far broader than what this skill covers — see *Scope and Limitations* above.
