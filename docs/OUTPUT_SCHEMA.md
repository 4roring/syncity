# 출력 JSON 스키마 (v1)

CLI와 뷰어가 공유하는 **계약(contract)**이다. 필드를 깨는 변경(삭제, 이름 변경, 의미 변경)을 하면 `schema_version`을 올린다. 필드 추가는 버전을 올리지 않는다.

## 예시

```json
{
  "schema_version": 1,
  "tool": { "name": "syncity", "version": "0.1.0" },
  "root": "/Users/me/p4/project",
  "summary": {
    "files": 2,
    "functions": 3,
    "loc": 210,
    "files_with_errors": 0,
    "elapsed_ms": 12
  },
  "nodes": [
    { "id": 0, "parent": null, "kind": "dir",  "name": "",     "path": "" },
    { "id": 1, "parent": 0,    "kind": "dir",  "name": "src",  "path": "src" },
    {
      "id": 2, "parent": 1, "kind": "file", "name": "foo.cpp", "path": "src/foo.cpp",
      "metrics": {
        "lines": 150, "loc": 120, "functions": 3,
        "cc_sum": 18, "cc_max": 12, "error_ratio": 0.0
      },
      "functions": [
        { "name": "Foo::bar", "line": 10, "end_line": 40, "loc": 28, "cc": 12 }
      ]
    }
  ]
}
```

## 규칙

- **`nodes`는 평탄한 배열**이며 `parent`로 트리를 구성한다. 뷰어는 `d3.stratify()`로 바로 변환할 수 있다.
- `id`는 0부터 시작하는 배열 인덱스와 같다. 루트는 항상 `id: 0`이고 `parent: null`이다.
- 부모 노드가 항상 자식보다 앞에 온다.
- 정렬: 경로 기준 사전순(디렉터리 우선 아님, 단순 경로 문자열 비교). 같은 입력이면 항상 같은 출력이 나온다.
- `path`는 `root` 기준 상대 경로이며 구분자는 항상 `/`이다 (Windows 포함).
- 분석 대상 파일이 하나도 없는 디렉터리는 출력하지 않는다.
- `kind: "file"` 노드만 `metrics`, `functions`를 가진다.
- 파일 처리에 실패하면 `metrics`를 생략하고 `"error": "<메시지>"`를 넣는다.
- `elapsed_ms`는 실행마다 달라지므로, 스냅샷 테스트에서는 0으로 고정하거나 제외한다.

## 필드 정의

메트릭 필드의 정확한 의미는 [METRICS.md](METRICS.md)를 따른다.
