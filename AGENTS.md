# AGENTS.md

AI 코딩 에이전트(Claude Code 등)가 이 저장소에서 작업할 때 따르는 지침이다.

## 프로젝트

**Syncity**: C/C++ 코드베이스의 복잡도를 분석해 3D 시티맵으로 시각화하는 도구.
현재 단계는 **Phase 1: Rust 분석 CLI**이다. 뷰어와 에디터 연동은 아직 만들지 않는다.

| 문서 | 내용 |
|---|---|
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | 크레이트 구조, 파이프라인, CLI 인터페이스, 설정 파일 |
| [docs/METRICS.md](docs/METRICS.md) | LOC·CC 계산 규칙. **메트릭 구현의 기준** |
| [docs/OUTPUT_SCHEMA.md](docs/OUTPUT_SCHEMA.md) | 출력 JSON 계약 |
| [docs/ROADMAP.md](docs/ROADMAP.md) | 단계별 체크리스트 |
| [docs/DECISIONS.md](docs/DECISIONS.md) | 설계 결정과 그 이유 |

## 명령어

```bash
cargo build                                   # 빌드
cargo test --workspace                        # 전체 테스트
cargo clippy --workspace --all-targets -- -D warnings
cargo fmt --all
cargo run -p syncity-cli -- analyze tests/fixtures/cpp --pretty
```

## 코드 규칙

- Rust edition 2024. `cargo clippy -- -D warnings`를 통과해야 한다.
- 에러 처리:
  - `syncity-core`는 `thiserror`로 에러 타입을 정의한다.
  - `syncity-cli`는 `anyhow`를 쓴다.
  - 테스트가 아닌 core 코드에서 `unwrap()`/`expect()`를 쓰지 않는다.
- `syncity-core`는 stdout/stderr에 직접 출력하지 않는다. 출력은 cli가 담당한다.
- 출력은 결정적이어야 한다. 병렬 처리 후 반드시 경로 기준으로 정렬한다.
- 파일은 바이트로 읽는다. `read_to_string` 금지 ([D6](docs/DECISIONS.md)).
- 의존성은 꼭 필요할 때만 추가하고, 추가했다면 이유를 PR이나 커밋 메시지에 남긴다.

## 메트릭을 바꿀 때

1. [docs/METRICS.md](docs/METRICS.md)를 먼저 고친다.
2. `tests/fixtures/cpp/`에 해당 케이스를 재현하는 샘플과 기대값을 추가한다.
3. 구현을 고친다.
4. `cargo test`가 통과하는지 확인한다.

## 테스트

- 픽스처는 `tests/fixtures/cpp/`에 둔다. 파일마다 기대 함수 이름, CC, LOC를 테스트 코드나 옆의 `.expected.json`에 명시한다.
- 엣지 케이스: 매크로가 붙은 함수, 템플릿, 클래스 내부 정의, 람다, `switch`/`default`, `&&`/`||` 연쇄, CP949 주석, CRLF, BOM, 빈 파일.

## 실행 환경

- 실제 사용처는 **폐쇄망 Windows PC**다. 회사 외부망 PC에서 `cargo build --release`로 만든 `syncity.exe` 하나만 반입한다 ([D7](docs/DECISIONS.md)).
- 개발 PC는 macOS지만 Windows에서 동작해야 한다. 경로는 `std::path`로 다루고, 출력할 때만 `/`로 정규화한다.
- 런타임에 외부 파일, 네트워크, 환경 설치를 요구하는 기능을 만들지 않는다.

## 금지 사항

- **이 저장소는 public이다.** 회사 소스 코드, 회사 매크로 이름, 경로, 프로젝트명, `diagnose` 결과를 커밋하지 않는다. 픽스처는 직접 작성한 코드만 쓴다. 회사 코드에서 발견한 패턴은 일반적인 이름으로 바꾼 최소 재현 코드로 새로 작성한다.
- **분석 대상 파일을 수정하지 않는다.** 이 도구는 읽기 전용이다.
- **Perforce 쓰기 명령(`p4 edit/submit/sync/revert` 등)을 실행하지 않는다.** 조회 명령만 허용한다 (Phase 4).
- 네트워크 호출을 추가하지 않는다 (Phase 4 이전).
- ROADMAP의 현재 Phase 범위 밖 기능(뷰어, LLM 모드, 다른 언어)을 미리 구현하지 않는다.
