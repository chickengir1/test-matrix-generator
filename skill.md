# Test Matrix Generator

pytest 기반 테스트 매트릭스를 생성한다.

## Trigger

`/test-matrix`

## Instructions

### Step 1: 도메인 파악

유저에게 질문한다:
- "어떤 기능을 테스트하려고? 관련 소스 파일은?"

답변 기반으로 코드베이스를 조사한다:
- **범위가 좁을 때**: Explore agent로 빠르게 조사
- **범위가 넓을 때**: Agent Team을 구성하여 병렬 조사 (예: UI 컴포넌트 팀 + API 서비스 팀)
- 도메인 모델 파악 (타입, 인터페이스, enum, 분기 조건)
- 엣지 케이스 식별

### Step 2: 매트릭스 설계

조사 결과를 바탕으로 매트릭스를 테이블로 제안한다:
- **Flows**: 테스트할 동작 목록
- **Axes**: 데이터 변형 축 (이름 + 항목 목록)
- cartesian product로 전체 조합 수 계산

유저 확인을 받는다. 축 추가/제거/수정 반영.

### Step 3: scaffold 생성

config JSON을 구성하고 scaffolder를 호출한다:

```bash
cat <<'EOF' | python3 ~/.claude/tools/test-matrix/generate.py init -
{
  "name": "프로젝트명",
  "output_dir": "{레포지토리 루트}/.claude/tests/{주제명}",
  "flows": ["flow1", "flow2"],
  "axes": [
    {"name": "axis_name", "items": [{"id": "...", "desc": "..."}]}
  ],
  "api": {"base_url_env": "...", "token_env": "..."}
}
EOF
```

- `output_dir`은 현재 레포지토리의 `.claude/tests/e2e/` (없으면 자동 생성)
- `api`는 E2E 테스트가 필요할 때만 포함

### Step 4: 내용 채우기

생성된 skeleton 파일에 Step 1에서 파악한 도메인 지식으로 실제 내용을 작성한다:

1. **mocks.py** — 각 item에 도메인 기반 mock 데이터 필드 추가
2. **logic.py** — 프론트엔드/백엔드 소스 코드 로직을 Python으로 재현
3. **test_flows.py** — flow별 assertion 로직 작성
4. **conftest.py** — API fixtures 구현 (E2E 테스트 시)
5. **common.py** — base URL, 인증 등 환경 맞춤 수정

### Step 5: 실행 + 검증

```bash
cd {레포지토리 루트}/.claude && python3 -m pytest tests/e2e/test_flows.py -v
```

실행 후 커버리지 검증:

```bash
python3 ~/.claude/tools/test-matrix/generate.py check {레포지토리 루트}/.claude/tests/{주제명}/
```

결과를 유저에게 보고한다:
- passed / failed / skipped 수
- 매트릭스 커버리지 (빠진 조합 경고)
- 실패 원인과 수정 제안

## Rules

- 도메인 로직을 충분히 파악하기 전에 코드를 생성하지 않는다
- 매트릭스는 반드시 유저 확인 후 생성한다
- mock 데이터는 실제 프로덕션 데이터 구조를 반영한다
- scaffold 후 반드시 내용을 채운다 (TODO 상태로 두지 않는다)
