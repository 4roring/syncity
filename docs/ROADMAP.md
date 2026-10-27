# 로드맵

목표: **회사 C/C++ 프로젝트(Perforce)의 Hotspot을 한눈에 보여주는 3D 시티맵.**
다른 언어와 LLM 압축 모드는 범위 밖이다 ([DECISIONS](DECISIONS.md)).

## Phase 1 — 분석 CLI ← 현재

- [ ] Cargo workspace 골격 (`syncity-core`, `syncity-cli`), edition 2024
- [ ] `walker`: ignore + 확장자/제외/크기 필터, `.syncity.toml` 로드
- [ ] 바이트 기반 파일 읽기 (비 UTF-8 허용, BOM/CRLF 처리)
- [ ] `parser`: tree-sitter-cpp, rayon `map_init`으로 스레드별 파서
- [ ] `metrics`: lines / loc / 함수 추출 / CC / error_ratio ([METRICS](METRICS.md))
- [ ] `preprocess`: 매크로 동일 길이 공백 치환
- [ ] JSON 출력 ([OUTPUT_SCHEMA](OUTPUT_SCHEMA.md)), 결정적 정렬
- [ ] `tests/fixtures/cpp` 샘플과 기대값 테스트
- [ ] lizard와 CC 비교 검증 (`/verify-metrics`, 외부 PC에서 픽스처와 공개 C++ 프로젝트로)
- [ ] `syncity diagnose`: error_ratio 상위 파일, 매크로 후보 식별자 빈도 ([D8](DECISIONS.md))
- [ ] `.cargo/config.toml`: msvc 타깃 `+crt-static` ([D7](DECISIONS.md))
- [ ] README에 Windows 빌드 방법 (rustup, VS Build Tools, `cargo build --release`)
- [ ] (보조) GitHub Actions: `windows-latest` 빌드와 테스트, exe artifact 업로드
- [ ] Windows 경로 처리 테스트 (`\` 구분자, 드라이브 문자)
- [ ] 회사 PC에서 exe 실행 → `diagnose` 결과로 매크로 목록 보강

**완료 기준:**
- 픽스처와 공개 C++ 프로젝트에서 함수별 CC가 lizard와 대부분 일치한다.
- 회사 Windows PC에서 exe 하나로 회사 레포 전체를 분석해도 중간에 죽지 않는다.

## Phase 2 — Standalone 3D 뷰어

- Vite + TS + Three.js, `analysis.json`을 드래그 앤 드롭으로 로드
- d3 treemap 레이아웃, InstancedMesh 건물, hover 시 파일명 툴팁
- 높이 = `cc_max`(로그 스케일), 면적 = `loc`, 색상 = `cc_max`

## Phase 3 — 에디터 연동

- 사용하는 에디터(VS Code 또는 Visual Studio)를 확정한 뒤 설계
- 클릭하면 해당 파일과 함수 위치로 이동

## Phase 4 — Perforce Churn

- `syncity churn` 서브커맨드: `p4 -ztag -Mj changes` / `describe -s -ds`
- 로컬 캐시와 증분 갱신 (서버 부하 최소화)
- 색상 = churn → Hotspot(복잡도 × 변경 빈도) 시각화
