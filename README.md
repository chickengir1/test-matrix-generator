# test-matrix-generator

시나리오 기반 pytest 테스트 매트릭스 scaffolder + 커버리지 검증 도구.

Given/When/Then 구조의 테스트를 생성하고, variation axes로 조합을 만듭니다. API 호출/페이로드 검증은 Postman MCP에 위임하며, 이 도구는 **로직 검증**에 집중합니다.

## 설치

```bash
# 도구
mkdir -p ~/.claude/tools/test-matrix
cp generate.py ~/.claude/tools/test-matrix/

# 스킬
mkdir -p ~/.claude/skills/test-matrix
cp skill.md ~/.claude/skills/test-matrix/

# (선택) ~/.claude/settings.json의 permissions.allow에 추가:
# "Bash(*python3 ~/.claude/tools/test-matrix/generate.py*)"
```

## 사용법

### Claude Code 스킬

```
/test-matrix
```

Claude가 UI 흐름(component → service → API)을 추적하고, 시나리오 정의 → 로직 포팅 → 테스트 실행까지 처리합니다.

### CLI 직접 사용

#### init — scaffold 생성

```bash
cat <<'EOF' | python3 generate.py init -
{
  "name": "bulk-update",
  "output_dir": "tests/bulk_update",
  "scenarios": [
    {
      "id": "bulk_update",
      "desc": "user bulk-updates a list",
      "given": "editor role, 3 items exist",
      "when": "update name field",
      "then": "3 changed, 0 failed",
      "axes": [
        {
          "name": "role",
          "items": [
            {"id": "editor", "desc": "editor role"},
            {"id": "viewer", "desc": "viewer role", "expect_fail": true},
            {"id": "admin", "desc": "admin role"}
          ]
        },
        {
          "name": "item_count",
          "items": [
            {"id": "empty", "desc": "0 items"},
            {"id": "single", "desc": "1 item"},
            {"id": "bulk", "desc": "100 items"}
          ]
        }
      ]
    }
  ]
}
EOF
```

출력:
```
init: tests/bulk_update/
  bulk_update: role(3) x item_count(3) = 9 cases
  total: 9 cases
  files: __init__.py, mocks.py, logic.py, test_scenarios.py
```

#### check — 커버리지 검증

```bash
python3 generate.py check tests/bulk_update/
```

출력:
```
check: tests/bulk_update/
  unfilled:
    ! mocks.py: 6 TODO(s)
    ! logic.py: has NotImplementedError

  axes:
    role: 3 items
    item_count: 3 items

  scenarios:
    standalone: 1 tests, 9 cases (axes: item_count, role)

  summary:
    total tests: 1
    total cases: 9
    parametrized: 1/1
```

## 생성 파일 구조

```
{output_dir}/
  __init__.py          # Python 패키지
  mocks.py             # 시나리오별 Given 상태 mock 데이터
  logic.py             # When 단계 로직 (소스 코드를 여기에 포팅)
  test_scenarios.py    # @pytest.mark.parametrize 시나리오 테스트
```

## Config 스펙

```json
{
  "name": "string — 테스트 이름",
  "output_dir": "string — 출력 경로 (하이픈은 자동으로 언더스코어 변환)",
  "scenarios": [
    {
      "id": "string — 시나리오 식별자",
      "desc": "string — 설명",
      "given": "string — 전제 조건",
      "when": "string — 실행할 동작",
      "then": "string — 기대 결과",
      "axes": [
        {
          "name": "string — 변형 축 이름",
          "items": [
            {
              "id": "string — 고유 식별자",
              "desc": "string — 설명",
              "expect_fail": "boolean (선택) — true이면 pytest.raises로 검증"
            }
          ]
        }
      ]
    }
  ]
}
```

- `scenarios` 필수. 각 시나리오에 최소 하나의 axis 필요.
- `expect_fail` 항목은 `pytest.raises`로 에러 경로를 검증.
- axes는 cartesian product로 조합.

## 워크플로우

1. **시나리오 정의** — UI 흐름(component → service → API) 추적, Given/When/Then 정의
2. **로직 포팅** — When 단계의 소스 코드 로직을 Python으로 재현
3. **API 응답 확인** — Postman MCP로 응답/페이로드 구조 확인
4. **mock 데이터** — 실제 API 응답 기반으로 Given 상태 데이터 작성
5. **scaffold** — `generate.py init`으로 테스트 구조 생성
6. **실행** — `pytest test_scenarios.py -v`
7. **검증** — `generate.py check`로 커버리지 및 미완성 TODO 확인

## 라이선스

MIT
