---
name: dev-loop
description: Syncity 기능 하나를 설계 → implementer 위임 → 리뷰 → 검증 순서로 진행하는 표준 개발 루프. ROADMAP 항목을 구현하거나, 메트릭 규칙을 바꾸거나, 여러 파일에 걸친 수정을 할 때 사용한다.
---

# Syncity 개발 루프

메인 세션(Opus)이 판단하고, 구현은 `implementer`(Sonnet)에게, 탐색은 `explorer`(Sonnet)에게 맡긴다.

## 1. 범위 확인
- `docs/ROADMAP.md`에서 이번 작업이 현재 Phase 범위 안인지 확인한다. 범위 밖이면 사용자에게 먼저 묻는다.
- 관련 명세(`docs/METRICS.md`, `docs/OUTPUT_SCHEMA.md`, `docs/ARCHITECTURE.md`)의 해당 절을 읽는다.

## 2. 조사 (필요할 때만)
- 크레이트 API, tree-sitter 노드·필드 이름, 기존 코드 위치를 모르면 `explorer`에게 구체적인 질문으로 맡긴다.
- 독립적인 질문이 여러 개면 `explorer`를 병렬로 띄운다.

## 3. 명세 먼저
- 동작이 바뀌는 작업이면 docs/를 **메인 세션이 직접** 먼저 수정한다.
- 새로운 결정이 생겼으면 `docs/DECISIONS.md`에 항목을 추가한다.

## 4. 위임
`implementer`에게 아래 다섯 가지를 담아 지시한다.

```
목표: <한두 문장>
명세: <docs 파일과 절>
수정 범위: <파일·모듈 목록>
완료 기준: <통과해야 할 테스트·명령, 기대 결과>
하지 말 것: <범위 밖 리팩터링, 의존성 추가 여부 등>
```

- 큰 작업은 독립 단위로 쪼갠다. 서로 다른 파일을 건드리는 단위만 병렬로 위임한다.

## 5. 리뷰
- 보고를 그대로 믿지 않는다. 변경된 파일을 직접 읽고 명세와 대조한다.
- 확인할 것: 명세 일치, 결정적 출력(정렬), core에서 `unwrap`·직접 출력 없음, 바이트 기반 읽기.
- 문제가 있으면 구체적인 수정 지시로 `implementer`에게 되돌린다 (SendMessage로 같은 에이전트에 이어서).

## 6. 검증
```bash
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```
- 메트릭 로직을 건드렸으면 `/verify-metrics`도 실행한다.

## 7. 마무리
- `docs/ROADMAP.md` 체크박스를 갱신한다.
- 사용자에게 바뀐 것, 검증 결과, 남은 판단 사항을 보고한다. 커밋은 사용자가 요청할 때만 한다.
