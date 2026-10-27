# Syncity

C/C++ 코드베이스를 분석해 3D 시티맵으로 시각화하는 도구.

- 바닥 면적 = 코드 줄 수(LOC), 높이 = 순환 복잡도, 색상 = 복잡도 (향후 Perforce churn)
- 분석은 로컬에서만 이루어지며, 소스 파일을 수정하지 않는다.

```bash
cargo run -p syncity-cli -- analyze <DIR> --pretty > analysis.json
```

문서: [아키텍처](docs/ARCHITECTURE.md) · [메트릭 명세](docs/METRICS.md) · [출력 스키마](docs/OUTPUT_SCHEMA.md) · [로드맵](docs/ROADMAP.md) · [설계 결정](docs/DECISIONS.md)
