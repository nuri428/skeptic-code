# skeptic-code

> "무죄 추정? 여기선 아닙니다. **필요하다고 증명되기 전까지는 삭제 대상입니다.**"

적대적 엔지니어링을 위한 Claude Code 스킬입니다 — 하나의 인식론(*증거 없이는 어떤 판정도
없다*), 다섯 개의 모드: 구현 가드, 집중 리뷰, 전체 감사, 버그 조사, 아키텍처 리뷰.

| | |
|--|--|
| 🇺🇸 | [English README](./README.md) |

---

코드 리뷰는 "이 코드가 올바른가?"를 묻습니다.  
`skeptic-code`는 묻습니다: **"이 코드가 존재해야 하는가? 그리고 존재해야 한다면 — 현실을 버텨낼 수 있는가?"**

## 3가지 원칙

| 원칙 | 물어봐야 할 질문 |
|------|----------------|
| **YAGNI** — You Aren't Gonna Need It | "현재 스펙에서 오늘 필요한가?" 아니라면 → 삭제. |
| **KISS** — Keep It Simple | "더 단순한 방법이 있는가?" 있다면 → 그걸 써라. |
| **DRY** — Don't Repeat Yourself | "이 로직이 이 레포 또는 의존성에 이미 있는가?" 있다면 → 재사용해라. |

> YAGNI > KISS > DRY 순서로 우선순위. 존재하지 않는 코드가 단순한 코드보다 낫다.  
> YAGNI는 추상화와 최적화에도 적용된다 — 명시적으로 요청받지 않은 한 도입하지 마라.

## 설치

**마켓플레이스를 통한 설치 (권장):**

```
/plugin marketplace add nuri428/skeptic-code
/plugin install skeptic-code@skepticcode
```

## 사용법

```
/skeptic-code:skeptic-code                  # 요청 형태로 모드 자동 선택
/skeptic-code:skeptic-code auto             # 구현하면서 도는 예방 가드
/skeptic-code:skeptic-code quick            # 심각도 HIGH 우선 상위 5개
/skeptic-code:skeptic-code deep             # 전체 라인 수준 감사
/skeptic-code:skeptic-code bug              # 증상 우선 결함 조사
/skeptic-code:skeptic-code architecture     # 경계·의존 방향 리뷰
/skeptic-code:skeptic-code <path>           # 특정 파일 또는 디렉토리
```

## 다섯 개의 모드

| 모드 | 우선순위 | 무엇인가 |
|---|---|---|
| `AUTO` | Scope > Correctness > Simplicity | 구현하면서 도는 경량 가드 — **새로 쓰는 코드**의 범위 초과·투기적 추상화·바퀴 재발명·조용한 실패를 입구에서 차단. diff 범위, 승인 절차 없음. |
| `QUICK` | Severity > Blast Radius > Simplicity | 집중 리뷰 — DEEP 파이프라인의 상위 5개만. |
| `DEEP` | YAGNI > KISS > DRY | 18개 항목 전체 라인 수준 감사. |
| `BUG` | Correctness > Reproduction > Root Cause > Minimal Fix > Simplicity | 증상 우선: 재현 → 깨진 invariant 명시 → 마지막 예외가 아니라 최초 위반 지점 수정 → regression test. contract mismatch·edge·state·ordering·leak·drift 등 8개 bug suspect. |
| `ARCHITECTURE` | Boundary > Dependency Direction > Coupling > Simplicity > Abstraction Cost | 경계·의존 방향·결합도·contract 검증. *구현체 하나 ≠ 불필요한 추상화* (`[BOUNDARY]` → KEEP), "레이어가 많다" 같은 취향은 증거가 아님. |

BUG·ARCHITECTURE 프로토콜은 `skills/skeptic-code/references/`에 있고 해당 모드가 돌 때만
로드됩니다 — 감사 실행이 그 컨텍스트 비용을 지불하지 않습니다.

## 8가지 용의자

| 태그 | 이름 | 죄목 | 방향 |
|------|------|------|------|
| `[GHOST]` | 죽은 코드 | 한때 필요했다. 이제는 아니다. 아직도 떠돌고 있다. | CUT |
| `[PROPHET]` | 투기적 기능 | 오지 않을 미래를 위해 작성됨. "아마 필요할 것 같아서..." | CUT |
| `[LIAR]` | 조용한 실패 | 에러를 처리한다고 주장한다. 실제로는 삼켜버린다. | FIX |
| `[TWIN]` | 중복 | 두 곳에 같은 로직. 하나는 존재해서는 안 된다. | CUT |
| `[STRANGER]` | 범위 초과 | 아무도 요청하지 않았다. 자연스럽게 추가됐다. 스펙에 없었다. | CUT |
| `[ORACLE]` | 검증 없는 가정 | 세상이 협조한다고 전제한다 — 검증도, 대비도, 테스트도 없이. | ADD |
| `[CLIFF]` | 무한 실패 경로 | 지금은 동작한다. 한계도, 재시도도, 바닥도 없어서 언젠간 무너진다. | ADD |
| `[WHEEL]` | 바퀴 재발명 | 프로젝트 패키지가 이미 제공하는 것을 직접 구현했다. | CUT |
| `[BOUNDARY]` | 의도된 경계 | 죄가 아님 — 가치가 구현체 수가 아니라 분리에 있는 경계. | KEEP |

## 작동 방식

반대 마인드셋을 가진 두 개의 탐색 패스, 그리고 적대적 검증 루프:

**Pass 1A — 존재 감사 (항목 1–12)**  
*삭제 편향.* "이 줄을 현재 스펙으로 정당화할 수 있는가?" → CUT 또는 FIX  
(항목 11–12 LIAR는 항상 FIX — 핸들러를 단순 삭제하지 말고 교체할 것)

**Pass 1B — 안전 감사 (항목 13–18)**  
*보강 편향.* "여기에 있어야 할 무언가가 빠져 있는가?" → ADD

두 패스 이후 모든 후보는 grep 증거로 검증된 뒤 판정이 내려진다. grep 결과와 줄 번호 없이는 `[CUT]`도 `[ADD]`도 없다. 그리고 **런타임** HIGH 판정에는 **재현**이 추가로 요구된다 — 가드는 읽어서가 아니라 실제로 발화시켜서 검증한다 (환경 문제로 재현 불가한 HIGH는 `[UNREPRODUCED-HIGH]`로 심각도를 유지하고 차단 여부는 사용자가 결정).

**Pass 3 — 독립 검증 (작성자 ≠ 검증자)**  
아무도 자기 시험지를 스스로 채점하지 않는다. 승인된 변경이 적용된 뒤, **그 변경을 작성하지 않은 fresh checker** — 감사자의 결론이 아니라 diff와 증거만 건네받은 별도 컨텍스트 — 가 변경을 깨뜨리려 시도한다: 증거를 재실행하고, 각 수정이 닫았다고 주장하는 결함을 재재현하고, 수정 자체를 공격한다. 판정은 정확히 셋 중 하나: `APPROVE`, `REQUEST_CHANGES`, `UNVERIFIED`. 검증자가 죽거나 타임아웃되면 아무것도 말해주지 않은 것이다 — 검증자의 죽음은 승인이 아니다. 같은 지적이 두 번 `REQUEST_CHANGES`로 돌아오면 루프를 멈추고 사람에게 에스컬레이션한다. 인증·결제·시크릿·파괴적 데이터 경로를 건드린 변경은 정확성과 보안, 두 레인의 검증을 받는다.

| 단계 | 행동 |
|------|------|
| 0 | 사전 체크 (테스트 스위트, vendored 파일, CLAUDE.md 베이스라인) |
| 1 | 전체 파일 읽기 — 스키밍 금지 |
| 2 | Pass 1A: 12개 존재 패턴 탐색 |
| 3 | Pass 1B: 6개 안전 패턴 탐색 |
| 4 | Pass 2: grep으로 모든 후보 검증 |
| 5 | 보고서 생성 |
| 6 | 발견 사항 제시 — 사용자 승인 후 수정 |
| 7 | 승인된 변경 적용, 각 변경 후 테스트 실행 |
| 8 | Pass 3: 독립 검증자 디스패치 — 수정·재디스패치 반복, `APPROVE`까지 |

**클린 결과는 유효하다.** 18개 패턴을 모두 돌렸는데 아무것도 없으면 `CLEAN` 출력 — 억지 발견 금지.

## 범위와 한계

이 스킬이 **커버하지 않는** 것:
- Race condition의 확정 판정 — BUG 모드의 `[RACE]`는 candidate-only: 패턴만으로는 SUSPECT,
  결정적 증명 또는 런타임 재현이 있어야만 BUG
- 프로파일링이나 동적 분석이 필요한 런타임 실패 시나리오

---

## 감사의 말

`skeptic-code`는 두 가지 출처에서 형태를 갖췄습니다.

**[Andrej Karpathy의 LLM 코딩 가이드라인](https://x.com/karpathy/status/2015883857489522876)**

*Simplicity First* 원칙 — 요청받지 않은 기능 추가 금지, 단일 사용 코드에 추상화 금지 — 은 `[PROPHET]`, `[STRANGER]`, helper-called-once 체크의 직접적인 출처입니다. *Think Before Coding* — 가정을 명시하고 구현 전에 확인하라 — 은 `[ORACLE]`의 출처입니다.

**프리모텀 방법론** (Gary Klein / Daniel Kahneman)

실패가 이미 일어났다고 가정하고 거슬러 올라가는 방식은 낙관적 리뷰와 전혀 다른 질의 검토를 강제합니다. 이 스킬의 "유죄 추정" 입장은 프리모텀을 라인 수준에 적용한 것입니다. `[CLIFF]`는 "실제 부하에서 이게 어떻게 실패하는가?"라는 프리모텀 질문을 grep으로 검증 가능한 패턴으로 압축한 것입니다.

**[claude-forge](https://github.com/sangrokjung/claude-forge)** (MIT)

Pass 3 검증 루프 — 작성자 ≠ 검증자, 3판정 체계, 검증자의-죽음은-승인이-아니다, 2회 반복 시 에스컬레이션 — 는 claude-forge의 `adversarial-reviewer` 에이전트, `review-loop` 스킬, `rules/adversarial-review.md`에서 가져와 적용한 것입니다.

이 접근법들 모두 이 스킬이 커버하는 것보다 훨씬 넓습니다. 위의 한계들은 실수가 아니라 의도된 설계 결정입니다.
