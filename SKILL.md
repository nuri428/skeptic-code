---
name: skeptic-code
description: Code skeptic audit. Every line of code is guilty until proven innocent — if it can't justify its existence, it gets cut. Finds ghosts (dead code), prophets (speculative features), liars (silent failures), twins (duplication), and strangers (unrequested additions). Two-pass verify-before-cut prevents false positives. Produces a prioritized cut list with concrete before→after diffs.
allowed-tools: [Read, Grep, Glob, Bash, Edit, Write, AskUserQuestion, TodoWrite]
---

# Skeptic-Code — 냉철한 코드 감사

> "Innocent until proven guilty? Not here.  
>  **Deleted until proven necessary.**"

코드 리뷰는 "이 코드가 올바른가?"를 묻는다.  
skeptic-code는 "**이 코드가 존재해야 하는가?**"를 묻는다.

모든 라인은 부채다. 모든 추상화는 도박이다. 모든 "혹시 몰라서"는 비용이다.

## 왜 기존 도구로는 부족한가

일반 코드 리뷰와 정적 분석 도구(linter, dead-code analyzer)는 공통된 전제를 가진다:  
**"동작하는 코드는 필요한 코드다."**

skeptic-code는 그 전제를 뒤집는다. 기존 도구가 잡지 못하는 세 가지가 있다:

- **YAGNI 위반** — 실행은 되지만 현재 요구사항과 무관한 코드. 정적 분석으로 플래그되지 않는다.
- **Silent failure** — `except: pass`, `catch (_) {}` 처럼 에러를 삼키는 코드. 린터는 패턴을 경고하지만 실제 hot path인지 판단하지 못한다.
- **Scope creep** — 스펙에 없는 기능이 자연스럽게 추가된 것. 도구는 코드가 사용되면 그냥 통과시킨다.

adversarial 프레이밍("유죄 추정")은 같은 코드를 봐도 다른 결론을 만든다.

## Usage

```
/skeptic-code              # 자동 범위 감지
/skeptic-code quick        # 상위 5개 발견, ~2분
/skeptic-code deep         # 전체 라인 수준 감사, ~10분
/skeptic-code <path>       # 특정 파일 또는 디렉토리
```

## The Five Suspects

| 태그 | 이름 | 죄목 |
|------|------|------|
| `[GHOST]` | Dead code | 한때 필요했다. 이제는 아니다. 아직도 떠돌고 있다. |
| `[PROPHET]` | Speculative feature | 오지 않을 미래를 위해 작성됨. "아마 필요할 것 같아서..." |
| `[LIAR]` | Silent failure | 에러를 처리한다고 주장한다. 실제로는 삼켜버린다. |
| `[TWIN]` | Duplication | 두 곳에 같은 로직. 하나는 존재해서는 안 된다. |
| `[STRANGER]` | Scope creep | 아무도 요청하지 않았다. 자연스럽게 추가됐다. 스펙에 없었다. |

## The Wanted List — 10 Known Offenders

즉시 사냥할 패턴들. 각각 발견이면 finding으로 기록한다.

1. **Helper called once** — 단일 callsite에서만 쓰이는 추출 함수 → 인라인
2. **Interface with one implementation** — "나중에 더 추가할 수도 있어서" → 인터페이스 삭제
3. **`except: pass` / `catch (_) {}`** — 거짓말. 제거하거나 실제로 처리하라.
4. **`or []` / `?? []` on a required field** — upstream 버그를 숨기는 패치
5. **Config flag that's always the same value** — config가 아니라 상수다
6. **`import X` where X is never called** — zombie import
7. **Commented-out code block** — 그게 git의 역할이다
8. **Null check on something that can't be null** — 아무것도 막지 않는 방어벽
9. **"We might want this later" abstraction** — YAGNI. 잘라라.
10. **UI/endpoint that wasn't in the spec** — Stranger. 자르거나 명시적 승인을 받아라.

## CRITICAL: Verify Before Cutting

**패턴만으로 플래그하지 않는다.** 모든 `[CUT]` 판정은 증거를 요구한다.

`[CUT]`으로 표시하기 전 반드시 확인:

```yaml
verification:
  read_full_file: true      # 의심 라인만이 아니라 전체 파일
  grep_all_callers: true    # 코드베이스 전체의 사용 횟수 확인
  check_scope: true         # tests/, __main__, dev-only인지 확인
  check_comment: true       # 존재 이유를 설명하는 주석이 있는지 확인

rule: 4가지 ALL 통과 시에만 [CUT].
      하나라도 "불확실" → [QUESTION] 또는 [FALSE_ALARM]으로 강등.
```

**모든 `[CUT]`에 필수 증거:**
```yaml
cut:
  what: "Helper _build_prefix() at fuseki.py:45"
  evidence: "grep: 1 caller (line 89). 독립 테스트 없음. 주석 없음."
```

`evidence`를 grep 결과와 라인 번호로 채울 수 없으면 → `[CUT]`이 아니다.

## Workflow

### Step 1: Read Everything

스키밍 금지. 전체 파일을 읽는다. 무엇이 존재하는지 파악한다:

```bash
wc -l <files>                           # 크기
grep -rn "def \|class \|function " .    # 정의된 모든 심볼
grep -rn "<symbol>" .                   # 모든 호출처
```

### Step 2: Pass 1 — Hunt

Wanted List를 순회한다. 각 파일에서 매칭되는 패턴을 기록한다.  
기록 항목: 무엇인지, 어디인지 (file:line), 어떤 태그인지.

### Step 3: Pass 2 — Verify

각 후보에 대해 검증 체크리스트를 실행한다.  
발견 주변 ±30 라인을 읽는다.  
4가지 모두 통과 → `[CUT]`.  
하나라도 실패 → `[QUESTION]` 또는 `[FALSE_ALARM]`.

### Step 4: Build Report

```yaml
skeptic_code:
  scope: "<감사 대상>"
  verdict_counts: {cut: N, question: N, false_alarms: N}

  findings:
    - id: SC-001
      tag: "[GHOST]"
      verdict: CUT
      location: "file.py:42"
      title: "Helper called once — inline it"
      evidence: "grep: 1 caller at file.py:89. 테스트 0개."
      before: |
        def _fmt(x): return f"prefix:{x}"
        label = _fmt(value)
      after: |
        label = f"prefix:{value}"
      lines_removed: 2

    - id: SC-002
      tag: "[PROPHET]"
      verdict: QUESTION
      location: "base.py:10-30"
      title: "Abstract Protocol with one concrete implementation"
      question: "현재 phase 내에 두 번째 backend가 계획되어 있는가?"

  false_alarms:
    - what: "No auth on /api endpoints"
      reason: "CLAUDE.md line 47: 'single-user assumption, no auth in v0.1'"
```

### Step 5: Present and Decide

**반드시 찾은 결과를 먼저 제시한 후에만 수정한다. 사전 승인 없이 편집하지 않는다.**

발견 요약과 가장 큰 절감 항목을 제시한다.  
AskUserQuestion으로 다음 선택지를 제공한다:
- 모든 CUT 항목 즉시 적용 (권장)
- 각 발견을 하나씩 검토
- 특정 ID만 적용
- 보고서만 — 변경 없음

### Step 6: Apply Approved Cuts

승인된 각 cut에 대해:
1. 정확한 before → after 제시
2. 파일 편집
3. 테스트 실행 (`pytest -q` 또는 동등한 명령)
4. 통과 확인 후 다음으로 이동

## Severity

| 심각도 | 예시 |
|--------|------|
| HIGH | hot path의 silent failure, 100줄 이상의 scope creep |
| MEDIUM | 미사용 추상화, 한 곳에서만 호출되는 helper, config ghost |
| LOW | zombie import, 오래된 주석, 불필요한 null check |

## Example

```
User: /skeptic-code src/web/router.py

Reading router.py (239 lines)...
Scanning for the 10 known offenders...

Pass 1 — 4 candidates found
Pass 2 — Verifying each:

  SC-001 [LIAR]: save_config() collects errors[] but always returns HTTP 200
    grep: errors list built but status never set. No test for partial failure.
    Verdict: CUT

  SC-002 [PROPHET]: _MAX_TTL_BYTES constant at module level
    grep: used in exactly one Form() call. No config, no override, no test.
    Verdict: QUESTION (or inline as literal)

  FA-001: "No auth on endpoints"
    Checked CLAUDE.md — single-user assumption is explicit design decision.
    Verdict: FALSE_ALARM

skeptic-code complete: 1 CUT, 1 QUESTION, 1 false alarm.

[AskUserQuestion: 발견 요약 제시, 적용 방식 선택 요청]

User: Apply all CUT items.

Applying SC-001...
  Before: return templates.TemplateResponse(...) always 200
  After:  status_code=207 when errors is non-empty
Running tests... 116 passed. ✓
```
