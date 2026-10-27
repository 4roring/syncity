# Syncity 아키텍처

C/C++ 코드베이스를 분석해 **3D 시티맵**으로 시각화하는 도구.
현재 범위는 **분석 CLI**이며, 뷰어와 VS Code 익스텐션은 이후 단계에서 붙인다 ([ROADMAP](ROADMAP.md)).

## 전체 흐름

```
[ 로컬 C/C++ 소스 (Perforce workspace) ]
        │  (읽기 전용)
        ▼
[ syncity-core ]  walker → preprocess → parser(tree-sitter-cpp) → metrics
        │
        ▼
[ syncity-cli ]   clap 인자 처리, JSON 직렬화 → stdout / --output 파일
        │
        ▼  analysis.json  (스키마: OUTPUT_SCHEMA.md)
[ (Phase 2) Three.js 뷰어 ]  →  [ (Phase 3) VS Code 익스텐션 ]
```

## 크레이트 구성

```text
syncity/
├── Cargo.toml                 # workspace
├── crates/
│   ├── syncity-core/          # 라이브러리: 모든 분석 로직 (I/O 출력 없음)
│   │   └── src/
│   │       ├── lib.rs         # pub fn analyze(root, &Config) -> Analysis
│   │       ├── config.rs      # .syncity.toml 로드
│   │       ├── walker.rs      # ignore 기반 파일 수집 + 확장자/크기/제외 필터
│   │       ├── preprocess.rs  # 매크로 토큰을 동일 길이 공백으로 치환
│   │       ├── parser.rs      # tree-sitter-cpp 파서 (스레드별 인스턴스)
│   │       ├── metrics.rs     # LOC, 함수별 CC, ERROR 비율
│   │       └── model.rs       # 출력 데이터 구조 (serde)
│   └── syncity-cli/           # 바이너리 `syncity`
│       └── src/main.rs
├── tests/fixtures/cpp/        # 직접 작성한 C/C++ 샘플 + 기대값
└── docs/
```

**core / cli 분리 이유:** 테스트는 core만 대상으로 작성하고, 나중에 다른 진입점(익스텐션용 데몬, LLM exporter 등)이 생겨도 core를 그대로 재사용한다.

## 처리 파이프라인

1. **수집 (`walker`)**
   - `ignore::WalkBuilder`로 `.gitignore`·`.ignore`·숨김 파일 규칙을 따른다.
   - `.syncity.toml`의 `exclude` glob, 확장자 화이트리스트, 최대 파일 크기로 한 번 더 거른다.
   - `ignore`는 바이너리 파일을 걸러주지 않는다. 확장자 화이트리스트가 1차 방어선이다.
2. **읽기**
   - 파일은 **바이트(`Vec<u8>`)로 읽는다. UTF-8을 요구하지 않는다.**
   - 회사 코드에는 CP949/EUC-KR 주석이 흔하기 때문이다. tree-sitter는 바이트 단위로 동작하므로 문제없다.
   - 식별자 이름만 lossy UTF-8로 변환한다. BOM은 건너뛰고, CRLF는 줄 수 계산에서 처리한다.
3. **전처리 (`preprocess`)**
   - 설정된 매크로 토큰(예: `DLLEXPORT`)을 **같은 바이트 길이의 공백**으로 치환한다.
   - 오프셋과 줄 번호가 바뀌지 않는다. 상세 규칙은 [METRICS.md](METRICS.md#매크로-전처리).
4. **파싱 (`parser`)**
   - `rayon`으로 파일 단위 병렬 처리. `tree_sitter::Parser`는 `Sync`가 아니므로 `par_iter().map_init(|| new_parser(), ...)`로 스레드마다 하나씩 쓴다.
   - v1은 `.c`를 포함한 모든 파일을 **tree-sitter-cpp**로 파싱한다 ([DECISIONS](DECISIONS.md#d3)).
5. **메트릭 (`metrics`)**: [METRICS.md](METRICS.md) 명세를 따른다.
6. **출력**
   - 결과를 경로 기준으로 정렬해 **결정적(deterministic)** 출력을 보장한다. 병렬 순서와 무관하게 항상 같은 JSON이 나온다.
   - stdout에는 JSON만, 진행 상황과 경고는 stderr로 보낸다.

## CLI 인터페이스 (v1)

```
syncity analyze <DIR> [--output <FILE>] [--config <FILE>] [--threads <N>] [--pretty]
```

| 옵션 | 기본값 | 설명 |
|---|---|---|
| `<DIR>` | (필수) | 분석 루트 |
| `--output` | stdout | JSON 저장 경로 |
| `--config` | `<DIR>/.syncity.toml` (있으면) | 설정 파일 |
| `--threads` | CPU 코어 수 | rayon 스레드 수 |
| `--pretty` | off | 들여쓴 JSON |

- **종료 코드:** `0` 성공 / `1` 치명적 오류 / `2` 인자 오류
- 파일 하나가 실패해도 전체를 중단하지 않는다. 실패 내용은 해당 파일의 `error` 필드에 기록한다.

## 설정 파일 `.syncity.toml`

```toml
[scan]
extensions = ["c", "h", "cc", "cpp", "cxx", "hpp", "hh", "hxx", "inl", "ipp"]
exclude = ["third_party/**", "**/*.pb.cc", "**/*.pb.h", "**/moc_*.cpp"]
max_file_size_kb = 2048

[preprocess]
# 함수 인식을 방해하는 회사 매크로. 동일 길이 공백으로 치환된다.
strip_macros = ["DLLEXPORT", "API_EXPORT", "FORCEINLINE"]
```

## 배포 (폐쇄망 Windows)

```
[개인 PC] 개발 → git push (GitHub public)
                    │
                    ▼
[회사 외부망 PC (Windows)] git clone → cargo build --release → syncity.exe
                    │ 반입                    (보조: GitHub Actions artifact)
                    ▼
[회사 개발망 PC (Windows)] syncity.exe analyze D:\p4\project
```

- 타깃은 `x86_64-pc-windows-msvc`이다. `.cargo/config.toml`에서 CRT를 정적 링크하므로, exe 하나로 실행되고 설치가 필요 없다.
- 빌드 PC에는 rustup(MSVC)과 Visual Studio Build Tools(C++)가 필요하다.
- 회사에서 문제가 생기면 `syncity diagnose`로 원인을 확인한다. 코드를 밖으로 가져오지 않고 매크로 이름 등 요약 정보만 가져온다 ([D8](DECISIONS.md)).

## 하지 않는 것 (v1)

- 네트워크 호출 (Perforce 연동은 Phase 4에서 별도 서브커맨드로)
- 소스 파일 수정 (항상 읽기 전용)
- 진짜 전처리 (`#include` 확장, `#ifdef` 평가)
