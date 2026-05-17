# skeptic-code

> "Innocent until proven guilty? Not here. **Deleted until proven necessary.**"

`skeptic-code`는 Claude Code용 냉철한 코드 감사 스킬입니다.

코드 리뷰가 "이 코드가 올바른가?"를 묻는다면,  
`skeptic-code`는 **"이 코드가 존재해야 하는가?"**를 묻습니다.

## 왜 기존 도구로는 부족한가

linter와 dead-code analyzer는 "동작하는 코드 = 필요한 코드"라고 전제합니다.  
`skeptic-code`는 그 전제를 뒤집어 기존 도구가 잡지 못하는 세 가지를 노립니다:

- **YAGNI 위반** — 실행은 되지만 현재 요구사항과 무관한 코드
- **Silent failure** — 에러를 삼키는 `except: pass` / `catch (_) {}` 패턴
- **Scope creep** — 스펙에 없는 기능이 자연스럽게 추가된 것

## 설치

```bash
/install-skill nuri428/skeptic-code
```

## 사용법

```
/skeptic-code              # 자동 범위 감지
/skeptic-code quick        # 상위 5개 발견, ~2분
/skeptic-code deep         # 전체 라인 수준 감사, ~10분
/skeptic-code <path>       # 특정 파일 또는 디렉토리
```

## 5가지 용의자

| 태그 | 이름 | 죄목 |
|------|------|------|
| `[GHOST]` | Dead code | 한때 필요했다. 이제는 아니다. 아직도 떠돌고 있다. |
| `[PROPHET]` | Speculative feature | 오지 않을 미래를 위해 작성됨. "아마 필요할 것 같아서..." |
| `[LIAR]` | Silent failure | 에러를 처리한다고 주장한다. 실제로는 삼켜버린다. |
| `[TWIN]` | Duplication | 두 곳에 같은 로직. 하나는 존재해서는 안 된다. |
| `[STRANGER]` | Scope creep | 아무도 요청하지 않았다. 자연스럽게 추가됐다. 스펙에 없었다. |

## 작동 방식

1. **읽기** — 전체 파일을 건너뛰지 않고 읽음
2. **1차 탐색** — 10가지 알려진 패턴으로 후보 식별
3. **2차 검증** — grep으로 실제 사용처 확인 후 `[CUT]` 결정
4. **보고서** — 구체적인 before→after diff 포함한 우선순위 목록
5. **적용** — 사용자 승인 후 수정, 테스트 실행 확인

## 핵심 원칙

**2-패스 검증**: grep 증거 없이는 절대 `[CUT]`을 내리지 않습니다.  
**[FALSE_ALARM]**: CLAUDE.md나 의도적 설계 결정은 건드리지 않습니다.  
**사용자 승인 우선**: 발견 결과를 먼저 제시하고, 편집은 승인 후에만 진행합니다.
