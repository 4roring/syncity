---
name: verify-metrics
description: syncity의 함수별 순환 복잡도(CC)를 lizard 결과와 비교해 차이를 찾는다. 메트릭 로직을 수정한 뒤, 또는 회사 코드에서 결과가 이상해 보일 때 사용한다. 인자로 분석할 디렉터리를 받는다 (기본값 tests/fixtures/cpp).
---

# 메트릭 검증 (syncity vs lizard)

대상 디렉터리: `$ARGUMENTS` (비어 있으면 `tests/fixtures/cpp`)

## 준비
- lizard가 없으면 사용자에게 설치를 요청한다. 직접 설치하지 않는다.
  ```bash
  pipx install lizard
  ```
- 결과 파일은 스크래치 디렉터리에 저장한다. 저장소 안에 만들지 않는다.

## 실행
1. syncity 결과
   ```bash
   cargo run -q -p syncity-cli -- analyze <DIR> --output <SCRATCH>/syncity.json
   ```
2. lizard 결과 (CSV: NLOC, CCN, token, PARAM, length, location, file, function, long_name, start, end)
   ```bash
   lizard --csv -l cpp <DIR> > <SCRATCH>/lizard.csv
   ```
3. 두 결과를 `(파일 경로, 시작 줄)` 기준으로 매칭해 비교한다. 비교 스크립트는 스크래치 디렉터리에 작성한다.
   - lizard에만 있는 함수 → syncity가 함수를 놓침 (매크로나 ERROR 노드 의심)
   - syncity에만 있는 함수 → 오인식
   - CC가 다른 함수 → 차이 목록

## 보고
- 매칭률: 일치한 함수 수 / lizard 함수 수
- CC 일치율과, 차이가 큰 상위 10개 (파일:줄, 함수명, syncity CC, lizard CC)
- 차이의 원인 분류: 명세상 의도된 차이 / syncity 버그 / lizard 쪽 한계
- 의도된 차이는 `docs/METRICS.md`의 「알려진 차이」에 추가하자고 제안한다.
- syncity 버그는 재현용 최소 픽스처 코드와 함께 수정 작업으로 정리한다. 회사 코드를 그대로 복사하지 않는다.
