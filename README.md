# skeptic-code

> "Innocent until proven guilty? Not here. **Deleted until proven necessary.**"

`skeptic-code`는 Claude Code용 냉철한 코드 감사 스킬입니다.

코드 리뷰가 "이 코드가 올바른가?"를 묻는다면,  
`skeptic-code`는 **"이 코드가 존재해야 하는가?"**를 묻습니다.

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
| `[GHOST]` | 죽은 코드 | 한때 필요했다. 이제는 아니다. 아직도 떠돌고 있다. |
| `[PROPHET]` | 투기적 기능 | 오지 않을 미래를 위해 작성됨. "아마 필요할 것 같아서..." |
| `[LIAR]` | 조용한 실패 | 에러를 처리한다고 주장한다. 실제로는 삼켜버린다. |
| `[TWIN]` | 중복 | 두 곳에 같은 로직. 하나는 존재해서는 안 된다. |
| `[STRANGER]` | 범위 초과 | 아무도 요청하지 않았다. 자연스럽게 추가됐다. 스펙에 없었다. |

## 작동 방식

1. **읽기** — 전체 파일을 건너뛰지 않고 읽음
2. **1차 탐색** — 10가지 알려진 패턴으로 후보 식별
3. **2차 검증** — grep으로 실제 사용처 확인 후 `[CUT]` 결정
4. **보고서** — 구체적인 before→after diff 포함한 우선순위 목록
5. **적용** — 사용자 승인 후 수정, 테스트 실행 확인

## 특징

- **2-패스 검증**: 패턴만으로 절대 삭제하지 않음 — 반드시 grep 증거 필요
- **[QUESTION] 안전망**: 불확실한 경우 삭제 대신 질문으로 처리
- **[FALSE_ALARM]**: CLAUDE.md나 의도적 설계 결정은 건드리지 않음
