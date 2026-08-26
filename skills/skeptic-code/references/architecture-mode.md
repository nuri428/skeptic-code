# ARCHITECTURE Mode — Boundary and Dependency Review

This file is the ARCHITECTURE-mode protocol. SKILL.md's mode router sends you here on a
layering / boundary / dependency / "이 구조 괜찮아?" request — run it from this text. The core
invariants live in SKILL.md and apply unchanged, as does the Architecture Intent Check (SKILL.md
defines it; DEEP uses it as a gate, this mode uses it as a starting lens).

Priority stack:

```text
Boundary > Dependency Direction > Coupling > Simplicity > Abstraction Cost
```

ARCHITECTURE mode does not grade "too many layers / too few layers". It verifies six questions:

```text
1. Should this boundary exist at all?
2. Is the dependency direction right?
3. Are domain and infrastructure needlessly coupled?
4. Does one change propagate into too many areas?
5. Does a contract expose implementation detail?
6. Is the abstraction's cost higher than its current value?
```

## Architecture Suspects

| Tag | Meaning | Direction |
|---|---|---|
| `[BOUNDARY]` | Deliberate architecture boundary — value is separation, inversion, substitution, isolation, or contract stability. | KEEP |
| `[BLEED]` | Implementation detail leaking across a layer/domain boundary. | FIX |
| `[INVERSION]` | Dependency direction opposite to the architecture's intent. | FIX |
| `[COUPLING]` | One change fans out into needlessly many modules. | FIX |
| `[CONTRACT]` | Public/stable contract bound to an unstable implementation detail. | FIX |
| `[CYCLE]` | Module/package dependency cycle. | FIX |
| `[ISLAND]` | Abstraction with no role — pure indirection. The architecture-vocabulary sibling of `[PROPHET]`. | CUT |
| `[BYPASS]` | Code that skips a defined boundary and reaches infrastructure/domain directly. | FIX |

## Evidence sources

Search these before judging — for this mode's findings and for any architecture candidate:

```text
CLAUDE.md · README.md · ARCHITECTURE.md · ADR/ · docs/architecture/ · docs/design/
DI / container registrations · plugin and provider registrations
package/module boundaries · public interfaces · import directions · layer boundaries
tests using substitution/mocking · deployment boundaries · external service adapters
```

Absent documentation is not absent intent — the code structure itself (import directions, DI
registrations, test fakes) is evidence.

## Workflow

```text
AR0 — Read architecture evidence (sources above)
AR1 — Identify system/module boundaries
AR2 — Build the dependency-direction map (imports, package deps)
AR3 — Inspect public/stable contracts
AR4 — Find boundary violations ([BLEED] [BYPASS] [INVERSION] [CYCLE])
AR5 — Measure change propagation / coupling ([COUPLING] [CONTRACT])
AR6 — Evaluate abstraction value vs cost ([BOUNDARY] vs [ISLAND])
AR7 — Verify every finding with concrete imports/calls/files
AR8 — Propose the smallest structural correction
AR9 — Independent verification where material — read references/pass3-verification.md and
      dispatch per that protocol
```

## Finding shape

Every architecture finding must carry all of:

```yaml
architecture_finding:
  id: SC-00N
  tag: "[BLEED]"
  location: "<files/modules>"
  claimed_boundary: "<the boundary at stake>"
  current_dependency_direction: "<A -> B>"
  expected_dependency_direction: "<B -> A>"
  evidence: "<imports/calls/contracts/docs — concrete, with file paths>"
  consequence: "<why this matters now, not in a hypothetical future>"
  smallest_fix: "<minimal structural change>"
```

## Taste is not evidence

None of these phrases can carry a verdict:

```text
"too many layers" · "too abstract" · "this feels complex"
"clean architecture says so" · "one implementation only"
```

A verdict needs concrete cost:

```text
Valid:
  Domain package imports the SQLAlchemy model directly in 7 files,
  so persistence schema changes propagate into domain logic.

Invalid:
  Repository pattern seems cleaner.
```

Reproduction rarely applies here — architecture findings are evidenced by imports, callsites,
and contracts, not by firing a runtime failure (SKILL.md's non-runtime HIGH rule). A `[CYCLE]`
is proven by the import graph; a `[COUPLING]` by the list of files one change touches.

`[BOUNDARY]` verdicts land in the base report's `keep: N` count; the rest map onto CUT/FIX as
usual.
