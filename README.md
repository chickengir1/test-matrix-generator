# test-matrix-generator

pytest 기반 테스트 매트릭스 scaffolder + 커버리지 검증 도구.

Claude Code 스킬(`/test-matrix`)과 함께 사용하면 도메인 조사 → 매트릭스 설계 → scaffold 생성 → 내용 채우기 → 실행 → 검증까지 자동화됩니다.

## 설치

```bash
# 도구 설치
mkdir -p ~/.claude/tools/test-matrix
cp generate.py ~/.claude/tools/test-matrix/

# 스킬 설치
mkdir -p ~/.claude/skills/test-matrix
cp skill.md ~/.claude/skills/test-matrix/

# (선택) Bash 권한 자동 허용 — ~/.claude/settings.json의 permissions.allow에 추가
# "Bash(*python3 ~/.claude/tools/test-matrix/generate.py*)"
```

## 사용법

### 1. Claude Code 스킬

```
/test-matrix
```

Claude가 도메인 조사 → 매트릭스 제안 → scaffold 생성 → mock/logic 채우기 → pytest 실행까지 처리합니다.

### 2. CLI 직접 사용

#### init — scaffold 생성

```bash
cat <<'EOF' | python3 generate.py init -
{
  "name": "my-feature",
  "output_dir": "tests/my_feature",
  "flows": ["create", "update", "delete"],
  "axes": [
    {
      "name": "user_role",
      "items": [
        {"id": "admin", "desc": "관리자"},
        {"id": "member", "desc": "일반회원"},
        {"id": "guest", "desc": "비회원"}
      ]
    },
    {
      "name": "data_state",
      "items": [
        {"id": "valid", "desc": "정상 데이터"},
        {"id": "empty", "desc": "빈 데이터"},
        {"id": "invalid", "desc": "잘못된 데이터"}
      ]
    }
  ],
  "api": {
    "base_url_env": "API_BASE_URL",
    "token_env": "API_TOKEN"
  }
}
EOF
```

출력:
```
init: tests/my_feature/
  flows: create, update, delete
  axes:  user_role(3) x data_state(3)
  mock tests:  27
  api tests:   27
  total slots: 54
  files: __init__.py, mocks.py, logic.py, conftest.py, test_flows.py, common.py
```

#### check — 커버리지 검증

```bash
python3 generate.py check tests/my_feature/
```

출력:
```
check: tests/my_feature/
  axes:
    user_role: 3 items
    data_state: 3 items
  combinations per flow: 9

  mock tests: 3 functions
    create: OK
    update: OK
    delete: OK

  coverage: 54/54 (100%)
```

빠진 `@pytest.mark.parametrize` 축이 있으면 경고:
```
    create: MISSING AXES: data_state
  coverage: 27/54 (50%)
```

## 생성 파일 구조

```
{output_dir}/
  __init__.py       # Python 패키지
  mocks.py          # axis별 mock 데이터 skeleton
  logic.py          # flow별 비즈니스 로직 stub
  conftest.py       # pytest fixtures
  test_flows.py     # @pytest.mark.parametrize로 매트릭스 테스트
  common.py         # HTTP helpers (api 설정 시에만)
```

## Config 스펙

```json
{
  "name": "string (필수) — 테스트 이름",
  "output_dir": "string (필수) — 출력 경로 (하이픈은 자동으로 언더스코어 변환)",
  "flows": ["string[] (필수) — 테스트할 동작 목록"],
  "axes": [
    {
      "name": "string — 축 이름",
      "items": [
        {"id": "string — 고유 식별자", "desc": "string — 설명"}
      ]
    }
  ],
  "api": {
    "base_url_env": "string — 환경변수명 (기본: API_BASE_URL)",
    "token_env": "string — 환경변수명 (기본: API_TOKEN)"
  }
}
```

- `api`는 선택. 없으면 mock 테스트만 생성.
- `axes`는 N개 가능. cartesian product로 조합.
- `output_dir`의 마지막 경로에 하이픈(`-`)이 있으면 자동으로 언더스코어(`_`)로 변환 (Python 패키지 호환).

## 설계 철학

- **generate.py는 구조만** — 일관된 파일 구조, 네이밍, parametrize 패턴 보장
- **내용은 도메인 전문가(또는 Claude)가 채움** — mock 데이터, 로직, assertion
- **init은 1회** — 생성 후 자유롭게 편집. 재실행 불필요
- **check는 수시로** — parametrize 누락, 커버리지 갭 자동 검출

## 라이선스

MIT
