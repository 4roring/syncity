# 메트릭 명세 (C/C++)

구현과 테스트의 **기준 문서**다. 동작을 바꾸려면 이 문서를 먼저 고친다.

## 1. 라인 수

| 이름 | 정의 |
|---|---|
| `lines` | 파일의 물리적 줄 수. 마지막 줄이 개행으로 끝나지 않아도 1줄로 센다. 빈 파일은 0 |
| `loc` | **주석도 공백도 아닌 문자가 하나 이상 있는 줄**의 수 |

- 주석 판정은 tree-sitter `comment` 노드 범위를 기준으로 한다. 정규식을 쓰지 않는다.
- 전처리 지시문(`#include`, `#define` 등)이 있는 줄도 `loc`에 포함한다.
- 함수의 `loc`도 같은 규칙을 따르며, 함수 정의 노드 범위 안에서만 센다.

## 2. 순환 복잡도 (CC)

함수 하나의 CC = **1 + 아래 결정 지점 수**

| 노드 (tree-sitter-cpp) | 가산 | 비고 |
|---|---|---|
| `if_statement` | +1 | `else if`는 중첩된 `if_statement`이므로 자동으로 +1 |
| `for_statement` | +1 | |
| `for_range_loop` | +1 | |
| `while_statement` | +1 | |
| `do_statement` | +1 | |
| `case_statement` | +1 | **`default:`는 제외** (노드에 `value` 필드가 없는 경우) |
| `catch_clause` | +1 | |
| `conditional_expression` | +1 | 삼항 연산자 `?:` |
| `binary_expression` (연산자 `&&`, `\|\|`, `and`, `or`) | +1 | 연산자 하나당 +1 |

다음은 세지 않는다.

- `switch_statement` 자체 (`case`에서 센다)
- `else` 단독
- `goto`, `break`, `continue`, `return`
- `#if` / `#ifdef` 분기. 다만 두 분기가 모두 파싱되므로, 분기 안에 있는 `if` 등은 **양쪽 다 센다** (알려진 과대 계상)

**검증 기준:** [lizard](https://github.com/terryyin/lizard)의 CCN과 일치하는 것을 목표로 한다. 차이가 나는 경우는 이 문서의 「알려진 차이」 절에 기록한다.

## 3. 함수 단위

- 함수 = `function_definition` 노드. 클래스 본문 안의 인라인 정의와 템플릿 함수를 포함한다.
- 선언만 있는 것(`declaration`, 프로토타입)은 함수가 아니다.
- **람다(`lambda_expression`)**: 별도 함수로 만들지 않고, 결정 지점을 **감싸는 함수에 합산**한다 (v1).
- **중첩된 `function_definition`** (로컬 클래스의 메서드 등): 별도 함수로 만든다. 바깥 함수에는 합산하지 않는다.
- **이름 (`name`)**
  - 선언자에 적힌 한정 이름을 그대로 쓴다 (`Foo::bar`, `ns::Foo::bar`, `operator==`, `~Foo`).
  - 클래스·구조체 본문 안의 정의는 바깥 클래스 이름을 붙인다 (`Foo::bar`). namespace는 붙이지 않는다 (v1).
  - 이름을 찾지 못하면 `"<anonymous>"`.
- **위치:** `line`, `end_line`은 1부터 시작하며 함수 정의 노드의 시작·끝 줄이다.

## 4. 파일 집계

| 이름 | 정의 |
|---|---|
| `functions` | 파일 안의 함수 개수 |
| `cc_sum` | 모든 함수 CC의 합 (함수가 없으면 0) |
| `cc_max` | 가장 큰 함수 CC (함수가 없으면 0) |
| `error_ratio` | `ERROR` 또는 `MISSING` 노드가 덮는 바이트 수 ÷ 파일 바이트 수 (0.0–1.0, 소수 4자리) |

시각화 기본 매핑(Phase 2 예정): 바닥 면적 = `loc`, 높이 = `cc_max`(로그 스케일), 색상 = `cc_max` → 이후 churn으로 교체.

## 매크로 전처리

회사 코드의 `DLLEXPORT void Foo()` 같은 패턴 때문에 tree-sitter가 함수를 인식하지 못하는 문제를 막기 위한 단계다.

- `strip_macros`에 등록된 식별자를 찾아 **같은 바이트 수의 공백(0x20)**으로 치환한다.
- 매칭은 **식별자 경계 기준**이다. `DLLEXPORT_X`, `MY_DLLEXPORT`는 치환하지 않는다.
- 주석과 문자열 리터럴 안의 토큰도 치환된다. 길이가 같으므로 메트릭에는 영향이 없다.
- 줄바꿈 문자는 절대 바꾸지 않는다. 줄 번호가 원본과 정확히 일치해야 한다.
- v1에서는 함수형 매크로 호출(`DECLARE_FOO(x)`)의 인자까지 지우지 않는다. 필요해지면 `strip_macro_calls`를 추가한다.

## 알려진 차이 / 한계

- `#ifdef` 양쪽 분기를 모두 세서 CC가 실제보다 높게 나올 수 있다.
- 매크로로 생성되는 함수(`DEFINE_HANDLER(Foo) { ... }`)는 이름이 부정확할 수 있다. `error_ratio`로 신뢰도를 판단한다.
- (lizard와 비교하며 발견되는 차이를 여기에 추가)
